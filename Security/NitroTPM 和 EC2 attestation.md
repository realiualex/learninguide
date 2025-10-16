 
## NitroTPM
PCR 是 TPM 一个很重要的参数，用 tpm2-tools 工具包，可以对tpm 的多数api做测试
EC2 instance attestation 是基于 nitroTPM 实现的机密计算。nitroTPM 是符合 TPM 2.0 标准的软件实现。一般 TPM 有几个作用：
1. 保证操作系统启动的时候的完整性: 保证OS内核没有被篡改
2. 本机密钥解封 (local key unsealing): 将一个密钥(或加密blob)，通过一个policy跟PCR值绑定，当PCR值没有被篡改，TPM才允许 unseal 这个密钥。这样能防止数据被篡改，或者迁移后被人拿到 (实际上 tpm 的 policy 很灵活，不仅仅跟 pcr绑定 `policy-pcr`，还可以跟 password`policy-password`，nv索引值`policy-nv`，外部签名授权 `policy-authorize`，甚至是多个条件` policy-or` 进行绑定)
3. 本机软件自检：能确保软件运行在符合预期的AWS实例上

### nitro tpm 做 seal / unseal
#### seal
seal/unseal 的本质，就是将一个 blob (private key之类的)，将其创建成一个 tpm 对象，并将它的 authPolicy绑定到某一个pcr状态，然后tpm内部感知到 pcr 与authPolicy要求一致时，tpm才会做 unseal还原出原来的 private key

示例
1. 创建一个 primary key (作为 parent)，生成的文件名 primary.crx
```
> tpm2_createprimary -C o -g sha256 -G rsa -c primary.ctx
> 
name-alg:
  value: sha256
  raw: 0xb
attributes:
  value: fixedtpm|fixedparent|sensitivedataorigin|userwithauth|restricted|decrypt
  raw: 0x30072
type:
  value: rsa
  raw: 0x1
exponent: 65537
bits: 2048
scheme:
  value: null
  raw: 0x10
scheme-halg:
  value: (null)
  raw: 0x0
sym-alg:
  value: aes
  raw: 0x6
sym-mode:
  value: cfb
  raw: 0x43
sym-keybits: 128
rsa: c6d189aaa88b6b8d4dd9829ac7a0588e5e6f54c16f53128eca6424e06f90a06d15e5700c874d332de97a0c0601cd60b08b9dc2c02177e40422daf4121d452632fb2c535aa392eb57b04deed39c1f4757df9abbb553a96fd6e7b00e379255ffa767a638e5c144c756864aa29aee1d898ee3256b1e88ff32602a6dc4ec5f2331153eef2ad00ee599b70f685491b5d49fe2a8dea893e8d44696b6
```

2. 将要绑定的 pcr 值读出来，比如我们这次绑定 pcr 0,1,2,7 这四个 (文件名 pcr.bin)
```
> tpm2_pcrread -o pcr.bin sha256:0,1,2,7

  sha256:
    0 : 0x737F767A12F54E70EECBC8684011323AE2FE2DD9F90785577969D7A2013E8C12
    1 : 0xA2C8E3CAB49EEEF62654E0C4113702C1B549904EB4FD553FE74D4C9414CF4207
    2 : 0x3D458CFE55CC03EA1F443F1562BEEC8DF51C75E14A9FCF9A7234A13F198E7969
    7 : 0x65CAF8DD1E0EA7A6347B635D2B379C93B9A1351EDC2AFC3ECDA700E534EB3068
```

3. 基于上一步的pcr值，生成一个 policy (文件名 pcr.policy)
```
> tpm2_createpolicy --policy-pcr -l sha256:0,1,2,7 -f pcr.bin -L pcr.policy
8c10480c94aefab990aaa81100947138f351d2c87c4bb4529a985a0000d22e8b
```

4. 用上一部创建的 policy, 创建一个 seal 对象，并把 secret 写进去 (注意这个命令的输出里的 authorization policy就是上一部的 policy)。此时会生成3个文件 (seal.pub, seal.priv, sealed.ctx)
```
> echo -n 'mysecret' | tpm2_create -C primary.ctx -L pcr.policy -i- -u seal.pub -r seal.priv -c sealed.ctx

name-alg:
  value: sha256
  raw: 0xb
attributes:
  value: fixedtpm|fixedparent
  raw: 0x12
type:
  value: keyedhash
  raw: 0x8
algorithm:
  value: null
  raw: 0x10
keyedhash: 05d3b34446ae69ae148a2f3bba0344cbb3e0c669f840ff59656ae01229830bfe
authorization policy: 8c10480c94aefab990aaa81100947138f351d2c87c4bb4529a985a0000d22e8b
```

#### unseal
* 自动做
最简单的 unseal 就是直接用 pcr 参数做，比如下面的命令，就能直接返回之前的密文即 mysecret (tpm2-tools在内部自动完成)
```
> tpm2_unseal -c sealed.ctx -p pcr:sha256:0,1,2,7

```

* 手动做
1. 我们需要先创建一个 policy session
```
tpm2_startauthsession --policy-session -S session.ctx
```

2. 在 session 里“声称”PCR符合
```
tpm2_policypcr -S session.ctx -l sha256:0,1,2,7   
```

3. 执行 unseal 动作
```
tpm2_unseal -c sealed.ctx -p session:session.ctx
```

4. 关闭session
```
tpm2_flushcontext session.ctx
```

### tpm 创建 rsa key

1. 先创建一个主密钥 primary key （`-C o` 指的是使用 owner hierachy， `primary.ctx` 是主密钥的上下文句柄）。此时相当于在 TPM 内部生成一个非导出的 根密钥 (在 endorsement key EK 或 SRK之下)
```
tpm2_createprimary -C o -c primary.ctx
```

2. 在这个主密钥下，创建长度为2048的 rsa 密钥 (rsa.priv 是 TPM加密保护后的私钥blob，并不是明文的)。这一步，我们也可以加上 `-L pcr.policy` ，在创建密钥的时候绑定某一个pcr，这样必须在pcr匹配的时候，rsa才能被 tpm 使用
```
tpm2_create -G rsa2048 -u rsa.pub -r rsa.priv -C primary.ctx
```

3. 把这个密钥加载回TPM，使其变成一个活跃的句柄 ( 生成TPM内部的 rsa.ctx 文件，用于后续的签名或解密操作)
```
tpm2_load -C primary.ctx -u rsa.pub -r rsa.priv -c rsa.ctx
```

#### TPM rsa 签名

1. 假设我们要签名的内容是
```
echo "hello TPM" > message.txt
```

2. TPM 内部会话，会使用刚刚生成的 RSA 私钥完成签名，将签名结果写入到 sig.bin 文件中，私钥从没有在外部出现过，签名动作在 TPM 内部完成 （签名是 RSASSA-PSS 格式，并不是 PKCS#1 v1.5填充格式）
```
tpm2_sign -c rsa.ctx -g sha256 -o sig.bin message.txt
```

验证签名 ( 输出为空)
```
tpm2_verifysignature -c rsa.ctx -m message.txt -s sig.bin -g sha256
echo $?
```

### tpm 持久化
默认tpm createprimary 创建的key是在内存里，并没有持久化，一旦掉电，这个key就消失了。（但如果有 .ctx文件，也是可以的）
如果想要持久化，需要用 `evictcontrol` 命令，后面要跟上 handler的地址，一般是从 `0x81000000` 开头的范围。
查看现在用了哪些handler
```
tpm2_getcap handles-persistent
```

然后可以写一个没有用的，如果 `0x81000001` 用了，我们就 +1
```
tpm2_createprimary -C o -g sha256 -G rsa -c primary.ctx
tpm2_evictcontrol -c primary.ctx 0x81000002

```

假设掉电后，我们开机，可以通过句柄，找回原来的pub key
```
tpm2_readpublic -c 0x81000002 -o pub.pem
```

也可以通过句柄直接操作 sign 等动作
```
tpm2_sign -c 0x81000001 -g sha256 -o sig.bin message.txt
```

## 在 attestable EC2 环境里用 NitroTPM
### 解决的问题

如果我们有一个 KMS，放在远程AWS账号，该KMS Policy只允许业务AWS账号的一个特定的 IAM Role 过来 decrypt。在没有 KMS attestation 的情况下，如果业务AWS账号被人黑了，或者运维存心做破坏，只要业务账号的人员，assume 到这个 role上，就可以 decrypt 了。但如果使用 KMS attestation，可以限制特定的PCR值，而该PCR与 EC2 instance系统又绑定到一起，且该EC2 instance无法登录，也没人能篡改里面的镜像。从而保证只有特定的EC2 ami才能访问到这个kms。可以将开发的程序打包到这个EC2 ami里面，从而避免敏感信息泄露。

### 环境准备
1. 用 kiwi-ng 做一个 attestable disk，将 disk 传到AWS Snapshot，通过Snapshot 做成AMI，用AMI启动 EC2
2. 创建一个对称式加密的 KMS，然后resource policy 的condition 里只允许 EC2的 PCR4 和 PCR7
### 镜像制作
EC2 attestation 需要自己做一个 image，可以用 kiwi-ng 来做，kiwi-ng的本质是，从软件仓库里，下载需要的软件，然后打包做成镜像，镜像可以是 dmg格式，也可以是iso 等格式。在 kiwi-ng 做镜像的时候，可以使用 overlayfs 将 文件系统 设置为只读，也可以用 verity_blocks = "all" 对整个系统的文件完整性做校验。这样即使镜像启动的操作系统里，被安装了某一个软件，一旦重启，就会被重置。当然我们也可以不用 overlayfs，也不做完整性校验，做普通的 linux发行版

kiwi-ng做好镜像后，可以用 `coldsnap` 这个软件，将 .raw 格式的磁盘文件，上传到 AWS snapshot里，再通过 `aws ec2 register-image ` 的命令将 snapshot 注册为 ami，在注册的时候，要启用 tpm。之后使用这个镜像创建的 EC2，就有 TPM 功能了


### 加密

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
