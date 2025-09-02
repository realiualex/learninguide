
## Cipher Suite
一套 Cipher Suite，是 TLS 某一个版本的一种实现。比如  `TLS_AES_128_GCM_SHA256` 这个 cipher suite，就是 TLS 1.3 的，TLS 1.2 则使用完全不同的 Cipher Suite（如  ECDHE-ECDSA-AES128-GCM-SHA256 就是 TLS 1.2）

## TLS 版本差异
### TLS 1.3 vs 1.2
#### TLS 1.3
以`TLS_AES_128_GCM_SHA256` 为例，在 TLS 1.3 的 extension 扩展字段里，提供了 key_share 的扩展，在 TLS handshake的时候，client hello 会在 key_share 里提供自己的 key_share pub key，并表明用的是哪个 group (这里指的是密钥交换组)，如可以使用 X25519 或者 X25519MLKEM768 (抗量子) 的密钥交换方法，然后 Server Hello 的时候，也会返回server 的 pub key，以及支持 的 TLS version 信息（如果客户端提供的 key_share 服务器不支持，会发送一个 HelloRetryRequest 要求客户端重新发送合适的 key_share）。之后双方通过在 X25519曲线上的 ECDHE 算法，双方在本地算出一个共享 secret（在本地算出来，不需要网络传输这个秘密，有点类似 TLS 1.2之前的 pre_master_key，但命名和流程不同），再通过 HKDF 派生出 AES128 对称式加密密钥（也是在本地完成，不需要网络传输这个 AES key）。

#### TLS 1.2
TLS 1.2 及其之前的版本，有一个 pre_master_key 的概念，如果支持 Forward Secrecy，就是 ECDHE交换出来的 pre_master_key，如果不支持 ECDHE，就是 client 生成一个 pre_master_key（通常是一个随机序列），然后用 server 的公钥，加密然后用这个key 发给server。之后双方各自用在本地这个 pre_master_key，加上双方各自的随机数，通过一个伪随机函数 PRF 生成对称式加密的 master key 和会话密钥(只要 client_random, server_random 和 pre_master_key 一样，执行多次，生成的对称式加密密钥也是一样的)


TLS 1.2 的 Forward Secrecy ( 前向加密)：
1. 如cipher  `ECDHE-ECDSA-AES128-GCM-SHA256`  ，这个属于 Forward Secrecy ，即生成 pre_master_key 的时候不依赖一个长期的 key，靠 ECDHE 来交换出 pre_master_key。ECDSA 是用于身份验证的算法，浏览器打开 https 网站是否告警，就是靠这个 ECDSA 签名验证来确认的，所以这种情况就要求服务器上配置的 https 证书颁发的时候，必须使用 ECDSA 签发的。同样的如果是 `ECDHE-RSA-AES128-GCM-SHA256` 就要求服务器的 ssl 证书是 RSA 签发的，同样的 pre_master_key 也是 ECDHE 交换出来的，RSA只用于自己身份的验证，不会用来加密这个 pre_master_key。 
2. 如 cipher `AES128-SHA256`，则不支持 Forward Secrecy。此时是用 服务器端的 RSA 公钥，本地生成一个 pre_master_key ，然后 RSA 公钥加密后，发给服务器端。


## Post Quantum 
在 TLS 1.3 的扩展中，如果支持 X25519MLKEM768，那么就能使用一种抗量子攻击的混合密钥交换机制。该机制结合了经典的 ECDHE（基于 X25519）和后量子密钥封装机制（ML-KEM，早期称为 Kyber），以提升密钥协商的安全性，尤其针对未来量子计算机威胁。ML-KEM 是一种基于格的非对称加密算法，是替代传统 RSA 密钥交换的有力后量子方案之一。



## curl 测试
```
# 测试 ML-KEM 抗量子攻击（要求 openssl 3.5+）
curl https://pq.cloudflareresearch.com/cdn-cgi/trace --curves X25519MLKEM768

curl -v --tlsv1.2 --tls-max 1.2 https://kms.ap-northeast-1.amazonaws.com

curl -v --tlsv1.3 https://kms.ap-northeast-1.amazonaws.com

```