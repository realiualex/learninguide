Cross Domain Solution 指的是：设置不同的安全域，控制不同的域之间的数据传输。 cross domain里的domain，指的是不同安全级别的网络，例如 机密网络到非机密网络。一般有两种场景：

## OWT: One Way Transfer
Data Diode， 一根光纤，发送端的光纤，只能发射光子，接收端的只能接收光子。从物理层保证数据只能单向传输。像tcp握手，是无法在两个 security domain之间实现的。所以要么用 udp 或其他单向传输协议(syslog, MQTT one way)，要么 CDS 产品支持在设备两侧跟客户端做TCP握手，然后客户端处协议在握手之后，开始发数据的时候，通过 OWT 的光纤，发给对端。

## MDG: multi-domain Data Guard
如果双方都要通信，那么一般是由两个网卡，两根 OWT 光纤实现的，并且 MDG 会对内容做过滤，协议识别，还有病毒查杀等功能集成在一起。 MDG 是非标准术语，一般也叫做 bi-dirctional 或 multi level CDS.


## 厂商
如 dataflowx 公司提供的 datadiodex，主要是 OWT 单向传输的产品
https://www.dataflowx.com/datadiodex

