title: Spark Yarn 整合
date: 2015-07-22 14:36:03
tags: spark
categories: 编程
---
本文档接上一篇 《[yarn hdfs 环境搭建](http://youfu.xyz/2015/07/22/yarn-hdfs/)》
### **下载安装spark**
``` bash
wget http://apache.fayea.com/spark/spark-1.4.0/spark-1.4.0-bin-hadoop2.6.tgz
tar -xzvf spark-1.4.0-bin-hadoop2.6.tgz
```

### **修改spark环境变量**
``` bash
export SCALA_HOME=/root/scala-2.10.4
export JAVA_HOME=/root/jdk1.8.0_45
export HADOOP_HOME=/root/hadoop-2.6.0
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export SPARK_MASTER_IP=master
export SPARK_LOCAL_DIRS=/root/spark-1.4.0-bin-hadoop2.6
```
<!-- more -->

### **分发到所有的slave机器上**
<!-- more -->
``` bash
[root@master ~]# pscp -h hosts.txt -r /root/spark-1.4.0-bin-hadoop2.6 /root/

[1] 14:14:06 [SUCCESS] slave2
[2] 14:14:07 [SUCCESS] slave1
```
### **启动Spark**
```
[root@master spark-1.4.0-bin-hadoop2.6]# sbin/start-all.sh
```
### **查看启动情况**
```
[root@master spark-1.4.0-bin-hadoop2.6]# jps
17664 Worker
1826 NameNode
2435 NodeManager
2341 ResourceManager
17814 Jps
2152 SecondaryNameNode
17501 Master
1951 DataNode
```
### ** Slave上查看**
```
[root@slave1 ~]# jps
15696 Worker
1905 NodeManager
1751 DataNode
15815 Jps
```
### **web界面**
http://192.168.10.37:8080/

### **spark在yarn集群上运行测试**
```
./bin/spark-submit --class org.apache.spark.examples.SparkPi --master yarn-cluster  lib/spark-examples*.jar 10
```

###**看运行情况**
http://192.168.10.37:8088/cluster/apps/FINISHED

![图片名称](http://7teb9r.com1.z0.glb.clouddn.com/spark-yarn.png)

可以看到spark的任务已经在Yarn上运行成功了。
