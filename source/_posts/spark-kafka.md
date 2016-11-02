title: spark streaming kafka 整合测试
date: 2015-07-22 14:43:08
tags: spark
categories: 编程
---
运行环境：
kafka 和zookeeper 运行在192.168.10.81

spark集群是192.168.10.37/38/39

修改kafka的server配置文件，将localhost修改为实际的ip地址或者host，否则spark连接会出问题：
``` html
host.name=192.168.10.81
zookeeper.connect=192.168.10.81:2181
```
<!-- more -->

启动zookeeper：
```
[root@81 kafka_2.10-0.8.2.1]# bin/zookeeper-server-start.sh config/zookeeper.prorties
```
启动kafka：
```
[root@81 kafka_2.10-0.8.2.1]# bin/kafka-server-start.sh config/server.properties
```
<!-- more -->

spark 日志配置文件conf/log4j.properties 选项修改，这样终端上就不会出现大量的INFO级别的信息。
```
log4j.rootCategory=WARN, console
```

下载spark-kafka的依赖包spark-streaming-kafka-assembly
```
wget http://search.maven.org/remotecontent?filepath=org/apache/spark/spark-streaming-kafka-assembly_2.10/1.4.0/spark-streaming-kafka-assembly_2.10-1.4.0.jar
```
将此文件放到下面目录下external/kafka-assembly

运行spark下面自带的python示例程序。
**Direct Approach (No Receivers)**
```
[root@master spark-1.4.0-bin-hadoop2.6]# bin/spark-submit --jars external/kafka-assembly/spark-streaming-kafka-assembly_2.10-1.4.0.jar examples/src/main/python/streaming/direct_kafka_wordcount.py 192.168.10.81:9092 test
```
或者

**Receiver-based Approach**
```
[root@master spark-1.4.0-bin-hadoop2.6]# bin/spark-submit --jars external/kafka-assembly/spark-streaming-kafka-assembly_2.10-1.4.0.jar examples/src/main/python/streaming/direct_kafka_wordcount.py 192.168.10.81:9092 test
```
启动kafka自带的producer终端
```
[root@81 kafka_2.10-0.8.2.1]# bin/kafka-console-producer.sh --broker-list 192.1610.81:9092 --topic test
```
输入单词：
i love you

spark的输出如下：
``` html
-------------------------------------------
Time: 2015-07-15 15:28:40
-------------------------------------------
(u'i', 1)
(u'you', 1)
(u'love', 1)
()
-------------------------------------------
Time: 2015-07-15 15:28:42
-------------------------------------------
```

如此，后续程序开发可以参考spark给出的示例，只要完成自己的业务逻辑就可以了。
