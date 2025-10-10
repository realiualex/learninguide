
## 在 attestable EC2 环境里用 NitroTPM
### 环境准备
1. 用 kiwi-ng 做一个 attestable disk，将 disk 传到AWS Snapshot，通过Snapshot 做成AMI，用AMI启动 EC2
2. 创建一个对称式加密的 KMS，然后resource policy 的condition 里只允许 EC2的 PCR4 和 PCR7

### 加解密过程

通过KMS的AES对称式加密，将一个 base64 string 加密，拿到密文

```shell
b64_content=$(echo "this is a test content" | base64)
aws kms encrypt --key-id $kms_arn --plaintext $b64_content

```

### 在 nitro TPM EC2上

实际环境可以细拆一下，但一般往往是同一个环境：

1. 在要解密的环境：创建一对 rsa key pair

```shell
private_key="$(openssl genrsa | base64 --wrap 0)"
public_key="$(openssl rsa \
    -pubout \
    -in <(base64 --decode <<< "$private_key") \
    -outform DER \
    2> /dev/null \
    | base64 --wrap 0)"
```

2. 在nitroTPM环境： 访问 nitroTPM 的 attestation 接口，将刚刚创建 的 rsa 的pub key发给 nitroTPM，申请一个 attestation document

```shell
attestation_doc="$(nitro-tpm-attest \
    --public-key <(base64 --decode <<< "$public_key") \
    | base64 --wrap 0)"
```

3. 在能访问KMS接口环境(应该和nitroTPM环境一样才有意义)：将密文以及 nitroTPM attestation document，一起发给 KMS做解密。KMS解密后，会用 rsa pub key 再做一层加密，加密成 CMS 格式返回
```shell
plaintext_cms=$(aws kms decrypt \
    --key-id "<KMS-key-ID>" \
    --recipient "KeyEncryptionAlgorithm=RSAES_OAEP_SHA_256,AttestationDocument=$attestation_doc" \
    --ciphertext-blob fileb://<(base64 --decode <<< "<Base64-encoded-ciphertext>") \
    --output text \
    --query CiphertextForRecipient)
```

4. 在要解密的环境：将拿到的cms 消息，用第一步产生的 rsa private key做解密。这样保证在互联网传输的时候，消息始终被加密。
```shell
openssl cms \
    -decrypt \
    -inkey <(base64 --decode <<< "$private_key") \
    -inform DER \
    -in <(base64 --decode <<< "$plaintext_cms")
```