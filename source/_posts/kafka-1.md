title: Kafka 环境搭建 (一)
date: 2015-07-22 14:12:58
tags: 大数据
categories: 大数据分析
---
下列搭建流程目前只在单台服务器上测试通过。
操作系统：CentOS 6.4

## **直接二进制环境**

### 下载kafka官网编译好的二进制文件

```
wget http://apache.fayea.com/kafka/0.8.2.1/kafka_2.10-0.8.2.1.tgz
tar -xzvf kafka_2.10-0.8.2.1.tgz
cd kafka_2.10-0.8.2.1
```
### 启动zookeeper
``` bash
bin/zookeeper-server-start.sh config/zookeeper.properties
```
### 启动kafka server
首先修改server.properties配置文件，将host.name 绑定到localhost，否则本地测试会报错。
``` 
host.name=localhost
```
启动server：
``` bash
bin/kafka-server-start.sh config/server.properties
```
<!-- more -->
### 创建一个topic
名字叫”HelloWorld”, 只有一个分区和一个备份。
``` bash
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic HelloWorld
```
输出:
```
Created topic "HelloWorld".
```

可以使用—-list选项来查看当前的所有的topic。
``` bash
bin/kafka-topics.sh --list --zookeeper localhost:2181
```
输出:
```
HelloWorld
```

### 生产者Producer
Kafka提供了一个命令行的工具，可以从输入文件或者命令行中读取消息并发送给Kafka集群。每一行是一条消息。
``` bash
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic HelloWorld
```
输入:
```
HelloWorld!       
HelloKafka!

```
### 消费者consumer

Kafka也提供了一个消费消息的命令行工具。
``` bash
bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic HelloWorld --from-beginning
```
输出:
```
HelloWorld!
HelloKafka!
```
可以看到消费者成功的接收到了topic HelloWorld的消息。

## 源代码编译Kafka
### 安装openjdk
``` bash
yum install   java-1.7.0-openjdk
yum install   java-1.7.0-openjdk-devel
```
### 安装gradle
``` bash
wget https://services.gradle.org/distributions/gradle-2.4-all.zip
unzip gradle-2.4-all.zip
cd gradle-2.4
export PATH=/root/gradle-2.4/bin:$PATH
```
运行gradle －v命令，查看gradle是否正确安装
```
[root@81 kafka-0.8.2.1-src]# gradle -v
```
输出:
```
------------------------------------------------------------
Gradle 2.4
------------------------------------------------------------

Build time:   2015-05-05 08:09:24 UTC
Build number: none
Revision:     5c9c3bc20ca1c281ac7972643f1e2d190f2c943c

Groovy:       2.3.10
Ant:          Apache Ant(TM) version 1.9.4 compiled on April 29 2014
JVM:          1.7.0_79 (Oracle Corporation 24.79-b02)
OS:           Linux 2.6.32-358.el6.x86_64 amd64
```
### 下载Kafka源代码
``` bash
wget http://apache.fayea.com/kafka/0.8.2.1/kafka-0.8.2.1-src.tgz
tar -xzvf kafka-0.8.2.1-src.tgz 
cd kafka-0.8.2.1-src
```
### 下载gradle wrapper
```
[root@81 kafka-0.8.2.1-src]# gradle
```
### 下载依赖的jar包
```
[root@81 kafka-0.8.2.1-src]# ./gradlew jar
```
### 生成release包
```
[root@81 kafka_2.10-0.8.2.1]# ./gradlew releaseTarGz
```
生成的包位于 core/build/distributions/kafka_2.10-0.8.2.1.tgz

之后的kafka环境搭建和二进制方式搭建一样。

参考: http://kafka.apache.org/documentation.html#quickstart 


