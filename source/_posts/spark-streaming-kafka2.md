title: spark streaming sql kafka mongodb 整合(一)
date: 2015-10-14 14:41:00
tags: spark
categories: 大数据分析
---
前段时间看了一下spark入门，因为项目需要从kafka中实时取数据，然后经过spark的处理，把处理后的数据存储到mongodb中。

根据官网的介绍：
http://spark.apache.org/docs/latest/streaming-kafka-integration.html

总结一下主要的数据处理流程:

1. spark streaming从kafka中取出json格式的数据，注意一个kafka的json message需要在一行，否则spark解析会出错。

2. spark streaming取出后的数据存储到 mongodb 集群中。

代码使用的是基于python的API：
<!-- more -->

```python

import sys
import json
import pymongo
from pyspark import SparkContext
from pyspark.streaming import StreamingContext
from pyspark.streaming.kafka import KafkaUtils

def getMongoClientInstance():
    if 'mongoClientSingletonInstance' not in globals():
        globals()['mongoClientSingletonInstance'] = pymongo.MongoClient('mongo-rs0, mongo-rs1, mongo-rs2', 27017)
    return globals()['mongoClientSingletonInstance']


def insert_mongodb(db_name, collection_name, documents):
    mongo_client = getMongoClientInstance()
    try:
        mongo_client[db_name][collection_name].insert_many(documents)
    except pymongo.errors.AutoReconnect as e:
        pass


def process_rdd(ts, rdd):
    #print rdd.count()
    if rdd.count():
        try:
            lines = rdd.map(lambda x: x[1]).collect()
            insert_mongodb("test_db", "test_collection", list(json.loads(x) for x in lines))
        except Exception as e:
            print e        


if __name__ == "__main__":
    batch_duration = 10
    zkQuorum, topic = sys.argv[1:]
    sc = SparkContext(appName="Test")
    ssc = StreamingContext(sc, batch_duration)
    try:
        kafka_streams =KafkaUtils.createStream(ssc, zkQuorum, "kafka-spark-streaming-mongodb-test", {topic: 1})
        kafka_streams.foreachRDD(process_rdd)
    except Exception as e:
        print e
    ssc.start()
    ssc.awaitTermination()
    ssc.stop()

```

运行命令如下：

```
/opt/spark/bin/spark-submit --jars /opt/spark/lib/spark-streaming-kafka-assembly_2.10-1.4.1.jar kafka-spark-streaming-mongodb-test.py zk-node1:2181,zk-node2:2181,zk-node3:2181 test

参数说明：
--jars 添加额外的jar包
zk-node1:2181,zk-node2:2181,zk-node3:2181 指定zookeeper的集群，kafka使用zookeeper管理
test 指定要接收信息的kafka topic
```

spark集群的环境变量 /opt/spark/conf/spark-env.sh 配置如下：
``` bash
export JAVA_HOME=/opt/jdk1.8.0_45
export SPARK_HOME=/opt/spark
export PATH=$SPARK_HOME/sbin:$SPARK_HOME/bin:$PATH

export SPARK_MASTER_IP=$(cat /etc/hostname)
export SPARK_MASTER_PORT=7077
export SPARK_MASTER_WEBUI_PORT=8080
```

spark 默认配置文件/opt/spark/conf/spark-defaults.conf如下：

```
# Default system properties included when running spark-submit.
# This is useful for setting default environmental settings.
spark.master    spark://spark-master0:7077,spark-master1:7077
spark.eventLog.enabled           true
spark.eventLog.dir               /opt/spark/logs
spark.history.fs.logDirectory    /opt/spark/logs
# spark.serializer                 org.apache.spark.serializer.KryoSerializer
spark.driver.memory              5g
spark.executor.memory            8g
spark.python.worker.memory       12g
spark.task.cpus                  1
spark.cores.max                  4
spark.local.dir                  /tmp
spark.executor.extraJavaOptions  -Duser.timezone=Asia/Shanghai
```
具体的集群环境搭建就不细说了，各种搭建教程数不胜数。

可以把spark streaming当做一个纯粹的kafka的高可用consumer来使用，把数据存储到mongodb中或者和别的系统(比如redis，elk等)来结合。

这样就省去了自己写kafka consumer。在生产环境中，一个consumer需要考虑分布式，负载，高可用，集群管理等各种各样的因素。
