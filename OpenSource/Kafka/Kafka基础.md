
在早期版本中，对kafka操作，是通过 zookeeper实现的。后来是通过 bootstrap server实现的。

## 列出topic
如果是AWS EC2，要指定IAM认证，需要创建[client.properties](https://docs.aws.amazon.com/msk/latest/developerguide/create-topic.html)
```
bin/kafka-topics.sh --list --bootstrap-server host1_domain:9098,host2_domain:9098,host3_domain:9098 --command-config bin/client.properties
```

## 读取某一个topic的消息

```
bin/kafka-console-consumer.sh \
--bootstrap-server BOOTSTRAP_SERVER_STRING \
--consumer.config bin/client.properties \
--topic TOPIC_NAME \
--from-beginning
```

## 列出partition
```
 bin/kafka-topics.sh --bootstrap-server BOOTSTRAP_SERVER_STRING --command-config bin/client.properties --describe
```

## 删除topic

```
 bin/kafka-topics.sh --bootstrap-server BOOTSTRAP_SERVER_STRING --command-config bin/client.properties --delete --topic TOPIC_NAME
```

## compact topic 和 non-compact topic
 compact topic 和 非 compact topic，主要影响数据保留的清理。如果是 compact topic，则会根据 kafka topic里的key进行清理，旧的key会被压缩删除。如果是 非 compact topic，是根据retention.ms 或 retention.bytes进行清理。默认是non-compact topic.

## 其他
对于加载配置文件，不同的 Kafka 命令行工具使用不同的参数名：

- kafka-topics.sh 使用 `--command-config`
- kafka-console-consumer.sh 使用 `--consumer.config`
- kafka-console-producer.sh 使用 `--producer.config`

## 运维配置
在 kafka 的配置里，对于持久化和可靠性，比较关键的指标是 RF 和 mini ISR，RF 指的是一共多少个副本，min ISR 指的是最少有多少个副本是可用的(一般 miniISR 为 RF-1，即允许集群里有1个kafka节点故障)。
所以对于一个3节点的 kafka来说：
RF 如果为 2，min ISR为1，则表示每条消息副本为2，只要 leader写入成功就认为成功。这种场景对性能比较好，只允许坏1个节点（因为每个partition 的leader 和 follower 都是分散在多个节点上，RF是2，如果坏2个节点，有可能某一个partition数据全丢了）。
如果 RF 为3， min ISR 为2，也允许坏一个节点（性能一般，但数据持久化有保障）。
`RF =3 ， minISR =2` 是最常见的配置

```
auto.create.topics.enable=false
default.replication.factor=2
min.insync.replicas=1
num.io.threads=8
num.network.threads=8
num.partitions=1
num.replica.fetchers=2
log.flush.interval.ms=5000
replica.lag.time.max.ms=30000
socket.receive.buffer.bytes=1024000
socket.request.max.bytes=104857600
socket.send.buffer.bytes=1024000
unclean.leader.election.enable=false
zookeeper.session.timeout.ms=18000
log.retention.hours=24
```