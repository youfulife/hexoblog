title: spark streaming sql kafka mongodb 整合(二)
date: 2015-10-14 14:05:41
tags: spark
categories: 大数据分析
---


接上一篇，这次加入了对中间数据的处理，主要处理流程如下：
1. spark streaming从kafka中取出json格式的数据，注意一个kafka的json message需要在一行，否则spark解析会出错。

2. 将spark streaming的DStream转换成spark sql的Data Frame，用spark sql的sql语句进行数据的CRUD，聚合等操作。

2. spark sql 操作结果 存储到 mongodb 集群中。

<!-- more -->

使用的是基于python的API，示例代码如下：

```python
import sys
import json
import pymongo
import time
from datetime import *
from pyspark import SparkContext
from pyspark.sql import SQLContext
from pyspark.streaming import StreamingContext
from pyspark.streaming.kafka import KafkaUtils


# Lazily instantiated global instance of SQLContext
def getSqlContextInstance(sparkContext):
    if 'sqlContextSingletonInstance' not in globals():
        globals()['sqlContextSingletonInstance'] = SQLContext(sparkContext)
    return globals()['sqlContextSingletonInstance']


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


def process_server(ts, sqlContext):
    sql_query = "select * from test"
    # 结果转换成 json格式 字符串 的数组，每一个json记录是以字符串格式存在的。
    collects = sqlContext.sql(sql_query).toJSON().collect()
    # 要插入到mongodb中还要用json.loads将字符串转换成json格式。      
    insert_mongodb('test_db', 'test_collection', list(json.loads(x) for x in collects))


def process_rdd(ts, rdd):
    #print rdd.count()
    if rdd.count():
        try:
            lines = rdd.map(lambda x: x[1])
            sqlContext = getSqlContextInstance(rdd.context)
            df = sqlContext.jsonRDD(lines)
            #df.printSchema()
            df.registerAsTable("test")
            process_server(ts, sqlContext)
        except Exception as e:
            print e        


if __name__ == "__main__":
    batch_duration = 10
    zkQuorum, topic = sys.argv[1:]
    sc = SparkContext(appName="Test")
    ssc = StreamingContext(sc, batch_duration)
    try:
        kafka_streams =KafkaUtils.createStream(ssc, zkQuorum, "spark-streaming-sql-mongodb-test-consumer", {topic: 1})
        kafka_streams.foreachRDD(process_rdd)
    except Exception as e:
        print e
    ssc.start()
    ssc.awaitTermination()
    ssc.stop()
```

这样一套流程下来，从kafka->spark streaming->spark sql->mongodb的流程就打通了。
