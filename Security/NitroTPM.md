
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

## 解析 attestation document
### 生成 attestation document
```shell
attestation_doc="$(nitro-tpm-attest --public-key <(base64 --decode <<< "$public_key") --user-data <(echo "mydata") --nonce <(echo "mynonce") | base64 --wrap 0)"

```

### 解码 cbor 格式数据
AWS NitroTPM 产生的 attestation document 用的是 cbor 格式，这是一个二进制格式，效率比 json 要高，我们可以 将其还原回json看下字段


```python
# pip install cbor2
import cbor2
from cbor2 import CBORTag
import json
import base64

att_doc_raw = ATTESTATION_DOCUMENT_BY_CLI

att_doc_b64 = att_doc_b64.replace("\n", "").replace(" ", "")
cbor_data = base64.b64decode(att_doc_b64)

def decode_recursive(obj):
    if isinstance(obj, CBORTag):
        return decode_recursive(obj.value)
    elif obj is cbor2.break_marker:   # 判断 break_marker
        return "break_marker"          # 或 None
    elif isinstance(obj, bytes):
        try:
            return decode_recursive(cbor2.loads(obj))
        except Exception:
            return obj.hex()
    elif isinstance(obj, list):
        return [decode_recursive(i) for i in obj]
    elif isinstance(obj, dict):
        return {decode_recursive(k): decode_recursive(v) for k, v in obj.items()}
    else:
        return obj

decoded = cbor2.loads(cbor_data)
decoded_clean = decode_recursive(decoded)

print(json.dumps(decoded_clean, indent=2))

```

生成的数据格式是这样的，其中能看到 nitrotpm 的pcr 值，如果我们带了 user data 和 nonce，那么attestation document里，以 hex 十六进制的方式显示。最后一段 `c173c001c93...a335e7f` 就是 TPM的签名值，

```json
[
  {
    "1": -35
  },
  {},
  {
    "module_id": "i-INSTNCE_ID-tpm0000000000000000",
    "digest": "SHA384",
    "timestamp": 1760093815777,
    "nitrotpm_pcrs": {
      "0": "6e901b16932f6e036747d7a57696e4a2cc864008ebc016826ca1d7bd42ab5ac8286ccf49cde6c0284cbc4b63d978a2ec",
      "1": "95aa7d2b587d7514fbd2ad1d743cd239ebb47192d996d77bd97239a970d47deda5c2cad05d2b9906931d7794f6331dcd",
      "2": "8923b0f955d08da077c96aaba522b9dece",
      "3": "8923b0f955d08da077c96aaba522b9dece",
      "4": "6f83b230e6ff6a284d37beb55e790c1223b8f2181a40c0d90348dc9b8160ae3e8cd73ad1a0a7bc1295406ca74d7a29dc",
      "5": -7,
      "6": "8923b0f955d08da077c96aaba522b9dece",
      "7": "98441c7f7625d10058c47683aec486ce311c633235eb555593a7ee791121e3578ae72d04ecef661f272d59058b77af35",
      "8": 0,
      "9": [
        14
      ],
      "10": 20,
      "11": "74a25ddbbf637c0578d9d73bdad40b6a6029c996605729b6cbb8566dc0ad86ae621aae783be2b2f1027e86e6906dce78",
      "12": 0,
      "13": 0,
      "14": 0,
      "15": 0,
      "16": 0,
      "17": "break_marker",
      "18": "break_marker",
      "19": "break_marker",
      "20": "break_marker",
      "21": "break_marker",
      "22": "break_marker",
      "23": 0
    },
    "certificate": -17,
    "cabundle": [
      -17,
      -17,
      -17,
      -17
    ],
    "public_key": -17,
    "user_data": "6d79646174610a",
    "nonce": "6d796e6f6e63650a"
  },
  "c173c001c939e4f3858b81827970b7cdaca0cef1f3482696ba35645968065c150ec127152b174495642eebd14eea3a1abe8f314cf211bb109cb2d4d38120ed5a567872eb39a2d28e8b9166d8d65d55dd49d3af274b6d36ead73f8b7caa335e7f"
]
```
