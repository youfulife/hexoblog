title: Kafka 环境搭建(二)
date: 2015-07-22 14:20:55
tags: kafka
categories: 大数据分析
---
在一台物理机上搭建一个3个节点的小Kafka 集群。

首先将上次配置好的server.properties 复制两份，作为每个node的配置文件。
``` bash
> cp config/server.properties config/server-1.properties
> cp config/server.properties config/server-2.properties
```

修改相关的配置文件，因为在同一台机器上，所以端口号和日志目录需要修改。broker id是kafka集群中每一个node的标示，不管在不在同一个机器上都必须是唯一的。

config/server-1.properties:
    broker.id=1
    port=9093
    log.dir=/tmp/kafka-logs-1

config/server-2.properties:
    broker.id=2
    port=9094
    log.dir=/tmp/kafka-logs-2

<!-- more -->
之前启动的zookeeper和server.properties node依然存在，启动新增的两个node:

``` bash
> bin/kafka-server-start.sh config/server-1.properties
> bin/kafka-server-start.sh config/server-2.properties
```
如果是从头开始，那么只要参考前面的单一节点操作，依次启动zookeeper和3个新的node即可。

新建一个topic, 设置3个备份。
``` bash
> bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 3 --partitions 1 --topic my-replicated-topic
```
可以通过--describe来查看当前集群中topic的配置状态:
```bash
bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic my-replicated-topic
```
输出：

Topic:my-replicated-topic     PartitionCount:1     ReplicationFactor:3     Configs:
     Topic: my-replicated-topic     **Partition: 0**     **Leader: 1**     Replicas: 1,2,0     Isr: 1,2,0

可以看到node1被选举为Partition:0的leader。

现在可以使用生产者脚本和消费者脚本来测试一下：
``` bash
> bin/kafka-console-producer.sh --broker-list localhost:9092 --topic my-replicated-topic
```
输入：

my test message 1
my test message 2
^C

``` bash
> bin/kafka-console-consumer.sh --zookeeper localhost:2181 --from-beginning --topic my-replicated-topic
```
输出：

my test message 1
my test message 2
^c

### **容错性测试**

通过kill掉某一个node来测试Kafka的容错性，关闭掉当前的leader节点。

找到当前leader节点：
```
ps -elf |grep server-1.properties
```
输出：

0 S root     **16284** 15317  2  80   0 - 1351442 futex_ 09:29 pts/2  00:00:20 java -Xmx1G -Xms1G -server...

关掉node1
``` bash
> kill -9  16284
```
查看当前的配置变化
``` bash
bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic my-replicated-topic
```
输出：

Topic:my-replicated-topic     PartitionCount:1     ReplicationFactor:3     Configs:
     Topic: my-replicated-topic     Partition: 0     Leader: 2     Replicas: 1,2,0     Isr: 2,0


运行消费者命令，发现没有任何影响。
``` bash
bin/kafka-console-consumer.sh --zookeeper localhost:2181 --from-beginning --topic my-replicated-topic
```
输出：

my test message 1
my test message 2

原先的生产者脚本可以继续输入：
```
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic my-replicated-topic
```

my test message 1
my test message 2
**hi gogogo
lll**

消费者脚本也可以继续接受新的数据:
```
bin/kafka-console-consumer.sh --zookeeper localhost:2181 --from-beginning --topic my-replicated-topic
```
`
my test message 1
my test message 2
**hi gogogo
lll**


同一台机器上的多个node集群搭建完毕。
