# 1 kafka搭建常用命令

## 1.1 kafka单机搭建和配置

    kafka搭建需要依赖jdk和zookeeper，其中jdk必装zookeeper可以使用kafka集成的，也可以单独安装。这里跳过jdk安装和zookeeper安装。

    安装kafka非常简单，将tar包复制到指定的目录，进行解压即可。

    kafka配置，kafka配置目录为server.properties

```properties
# kafka服务唯一id,集群中每个装有kafka的机器都是一个broker，每个broker的唯一id
broker.id=0
# 访问许可，如果kafka服务需要被外部主机访问到需要配置
listeners=PLAINTEXT://192.168.56.11:9092

num.network.threads=3

num.io.threads=8

socket.send.buffer.bytes=102400

socket.receive.buffer.bytes=102400

socket.request.max.bytes=104857600
# kafka数据存储目录
log.dirs=/tmp/kafka-logs
# topic默认分区数
num.partitions=1

num.recovery.threads.per.data.dir=1
# 分区默认副本因子（分区的存储份数）
offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1
####### record保留策略 ###########
# kafka record默认存储时间，过了时间会被删除
log.retention.hours=168

log.retention.check.interval.ms=300000
# zookeeper配置，集群配置localhost:2181;localhost:2182;localhost:2183
zookeeper.connect=localhost:2181

zookeeper.connection.timeout.ms=18000

group.initial.rebalance.delay.ms=0
```

    启动kafka服务，在启动kafka服务之前需要先启动zookeeper。

```shell
./kafka­-server­start.sh [-daemon] ../config/server.properties
```

    停止kafka服务

```shell
./kafka-server-stop.sh
```

## 1.2 kafka客户端API常用命令

1. 创建一个名为test-topic主题，且主题只有一个分区，一个备份

```shell
./kafka-topics.sh --create bootstrap-server 192.168.56.11:9092\
--replication-factor 1 --partitions 1 --topic test-topic
```

2. 查看创建的topic信息

```shell
./kafka-topics.sh --describe --topic test-topic \
--bootstrap-server 192.168.56.11:9092
```

3. 发送消息到test-topic主题（kafka提供了命令行工具，可以输入文件或命令行中读取消息发送到kafka集群，每一行是一条消息）

```shell
./kafka-console-producer.sh --broker-list 192.168.56.11:9092 --topic test-topic
```

4. 消费消息（kafka提供了消费消息的命令行工具，将存储的消息输出出来）

```shell
# 消费消息
./kafka-console-consumer.sh --bootstrap-server 192.168.56.11:9092 --topic test-topic \
--from-beginning

# 消费多个主题
kafka‐console‐consumer.sh ‐‐bootstrap‐server 192.168.56.11:9092 ‐‐whitelist "test|test‐2"

# 给消费者绑定消费组
./kafka‐console‐consumer.sh ‐‐bootstrap‐server 192.168.56.11:9092 ‐‐consumer‐property group.id=testGroup ‐‐topic t
et
```

    --from-beginning：参数希望消费者从头开始消费，如果不指定则消费者从下一条消息开始消费。

5. 查看当前Kafka存在的topic

```shell
./kafka-topics.sh --list --bootstrap-server 192.168.56.11:9092
```

6. 删除topic

```shell
./kafka-topics.sh --delete --topic test-topic --bootstrap-server 192.168.56.11:9092
```

7. 消费组命令

```shell
# 查看消费组列表
./bin/kafka‐consumer‐groups.sh ‐‐bootstrap‐server 192.168.56.11:9092 ‐‐list

# 查看消费组的消费偏移量
bin/kafka‐consumer‐groups.sh ‐‐bootstrap‐server 192.168.65.60:9092 ‐‐describe ‐‐group testGroup
```

# 2 kafka核心架构

![kafka-framework](../picture/mq/kafka-framework.png)

## 2.1 kafka中术语

| 术语            | 注释                                                                                            |
| ------------- | --------------------------------------------------------------------------------------------- |
| Broker        | 集群中每一台装有kafka的机器称为broker                                                                      |
| Topic         | 每次发送消息都要指定一个Topic，用于对消息进行分类，是一个逻辑概念                                                           |
| Partition     | 真正存储消息的容器，一个topic可以分为多个partition，会保证每个partition中的消息有序，是一个物理概念                                 |
| Producer      | 生产消息的生产者                                                                                      |
| Consumer      | 用于消费消息的消费者                                                                                    |
| ConsumerGroup | 消费组，每一个consumer都要指定一个consumer group，一个消息可以被多个不同的consumer组消费，但是一个consumer group指定有一个consumer消费 |
| Record        | kafka中也将消息称为Record 记录                                                                         |

## 2.2 Topic和Partition

kafka中topic是对消费的分类，可以将同一类型的消息发布到统一的Topic中。每个Topic中都有一个或多个Partition。

partition是真正存储数据的容器，partition是一个有序的message列表，partiton中的每个消息都有唯一的编号，称为offset。每个partition保证有序。每个partition对应一个commit log文件，用于存partition中的数据。

集群模式下，同一个topic中的partition会存储在不同的broker中，并且会有备份，提高容灾能力。如下：

![kafka-partition](../picture/mq/kafka-partition.png)

主题my-replicated-topic 有两个partition，partition-0 partition-1；并且每个partition有3个副本。Leader表示partition-0 存储在0节点，partition-1存储在2节点。replicas表示partition的副本存储几点，Isr是replicas的子集，表示当前还存活的，并且已经同步备份的节点。

> 只有leader节点才提供读写操作

## 2.3 Consumer

kafka中每个consumer都是基于自己在commit log中的消费进度工作的，消费进度offset由consumer自己来维护，一般情况下，我们是逐条消费commit log中的消息，也客户通过指定offset来重复消费某些，或者跳过某些消息。

这种模式意味着kafka中consumer对集群的影响非常小，添加多个或减少多个consumer对集群来说没有影响，应为consumer的消费进度由自己保存。

## 2.4 持久化

kafka一般不会删除消息，无论消息是否被消费。只会根据配置的日志保留时间（log.retention.hours）确认多久后删除日志，默认保留一周。kafka的性能与保留的消息数据量没有关系。

## 2.5 消费模式

kafka中消息消费模式有两种：一种是单播消费，一种是多播消费。

单播消费：一条消息只能被 一个消费者消费一次的模式，kafka只允许同一条消息被同一个消费组中的一个消费者消费。所以只需要保证消费者在一个消费组中就为单播消费。

多播消费：一条消息可以被多个消费者消费的模式，实现多播，只需要保证消费者属于不同的消费者即可。

# 3 Java API使用


