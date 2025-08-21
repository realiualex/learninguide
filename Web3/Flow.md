## 改进
Flow 跟 ETH相比，很大的改进是：
1. 速度更快 
2. 账户地址以及存储 和 认证的 key pair分离
	1. 支持 ECDSA_secp256k1 和 ECDSA_P256 两种曲线
	2. 原生支持 multi-sig，不依赖合约实现。multi-sig 可以走不同的曲线

## 存储
Flow 和 ETH的很大不同点，在于Flow账户需要有一个存储，账户余额越多，存储就越多。当我们想创建一个 flow 账户的时候，可以先用 flow cli 创建一对 key pair，然后再用一个已经有 flow 余额的账户，给这个pub key激活，就能生成 flow 地址（注意地址是链上生成，并不是公钥推导，也就意味着，必须有一个上链激活的过程）

* 激活地址的时候，目标地址里，要有storage才行，也就意味着，目标地址要有 flow 余额。所以原始地址，要有flow，在激活的时候，会自动转给要激活的地址一些，这样要激活的地址就有余额，也就有storage了，就能激活成功
* 在 flowscan.io 上，我们看一个 flow 地址的余额，在 Tokens 里能看到 Storage Path: /storage/flowTokenVaule 里有余额，在 Flow Balance 里也有。实际上真正的余额是在  /storage/flowTokenVaule ，但需要 Cadence 脚本才能读到。Flow 为了简化，所以链上提供了一个api，能通过试图来粗略读出来 flow 的余额 （不是100%精确，也是链上的视图），这样不用 Cadence 脚本，会简单一些
