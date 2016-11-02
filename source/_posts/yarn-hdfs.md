title: yarn hdfs 环境搭建
date: 2015-07-22 14:28:58
tags: spark
categories: 编程
---

## **环境准备**
操作系统：centOS 7.0
Hadoop版本：hadoop-2.6.0
spark版本：spark-1.4.0-bin-hadoop2.6

###**修改主机名**
搭建1个master，2个slave的集群方案。
首先修改主机名vim /etc/hostname，在master上修改为master，其中一个slave上修改为slave1，另一个slave2。
<!-- more -->
###**配置hosts**

在每台主机上修改host文件，添加下面三行
vim /etc/hosts

192.168.10.37      master
192.168.10.38      slave1
192.168.10.39      slave2

配置之后ping一下用户名看是否生效
ping slave1
ping slave2


## **SSH 免密码登录**


在所有机器上都生成私钥和公钥

``` bash
ssh-keygen -t rsa   #一路回车
```
需要让机器间都能相互访问，就把每个机子上的id_rsa.pub发给master节点，传输公钥可以用scp来传输。

slave1上：
``` bash
scp ~/.ssh/id_rsa.pub root@master:~/.ssh/id_rsa.pub.slave1
```
slave2上：
``` bash
scp ~/.ssh/id_rsa.pub root@master:~/.ssh/id_rsa.pub.slave2
```
在master上，将所有公钥加到用于认证的公钥文件authorized_keys中

将公钥文件authorized_keys分发给每台slave
``` bash
cat ~/.ssh/id_rsa.pub* >> ~/.ssh/authorized_keys
scp ~/.ssh/authorized_keys root@slave1:~/.ssh/
scp ~/.ssh/authorized_keys root@slave2:~/.ssh/
```

在每台机子上验证SSH无密码通信
```
ssh master
ssh slave1
ssh slave2
```
如果登陆测试不成功，则可能需要修改文件authorized_keys的权限（权限的设置非常重要，因为不安全的设置安全设置,会让你不能使用RSA功能 ）
```
chmod 600 ~/.ssh/authorized_keys
```

## **安装 Java**
从官网下载最新版Java
```
wget http://download.oracle.com/otn-pub/java/jdk/8u45-b14/jdk-8u45-linux-x64.tar.gz?AuthParam=1436251225_faa358076a5128993bddaea12fcde6bc
tar -xzvf jdk-8u45-linux-x64.gz
```
修改环境变量vim /etc/profile，添加下列内容，注意将路径替换成自己的：
```
export WORK_SPACE=/root
export JAVA_HOME=$WORK_SPACE/jdk1.8.0_45
export JRE_HOME=/root/jdk1.8.0_45/jre
export PATH=$JAVA_HOME/bin:$JAVA_HOME/jre/bin:$PATH
export CLASSPATH=$CLASSPATH:.:$JAVA_HOME/lib:$JAVA_HOME/jre/lib
```
然后使环境变量生效，并验证 Java 是否安装成功。
```

[root@master hadoop-2.6.0]# java -version
java version "1.8.0_45"
Java(TM) SE Runtime Environment (build 1.8.0_45-b14)
Java HotSpot(TM) 64-Bit Server VM (build 25.45-b02, mixed mode)
```
## **安装scala**
Spark官方要求 Scala 版本为 2.10.x，注意不要下错版本。
```
wget http://www.scala-lang.org/files/archive/scala-2.10.4.tgz
tar -xzvf scala-2.10.4.tgz
```
再次修改环境变量 vim /etc/profile，添加以下内容：
```
export SCALA_HOME=$WORK_SPACE/scala-2.10.4
export PATH=$PATH:$SCALA_HOME/bin
```
同样的方法使环境变量生效，并验证 scala 是否安装成功
``` bash
source /etc/profile   #生效环境变量
[root@master hadoop-2.6.0]# scala -version
Scala code runner version 2.10.4 -- Copyright 2002-2013, LAMP/EPFL
```
## **安装配置 Hadoop YARN**
从官网下载 hadoop2.6.0 版本，目前hadoop最新稳定版本为2.6.0
``` bash
wget http://apache.fayea.com/hadoop/common/hadoop-2.6.0/hadoop-2.6.0.tar.gz
tar -zxvf hadoop-2.6.0.tar.gz
```

### **配置 Hadoop**
```
cd /root/hadoop-2.6.0/etc/hadoop
```
进入hadoop配置目录，需要配置有以下6个文件：hadoop-env.sh，slaves，core-site.xml，hdfs-site.xml，maprd-site.xml，yarn-site.xml


#### **在hadoop-env.sh中配置JAVA_HOME **
```
export WORK_SPACE=/root
export JAVA_HOME=$WORK_SPACE/jdk1.8.0_45
```
#### **在slaves中配置slave节点的ip或者host**
```
[root@master hadoop]# cat slaves
localhost
slave1
slave2
```
#### **修改core-site.xml**
``` html
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://master:9000/</value>
    </property>
    <property>
         <name>hadoop.tmp.dir</name>
         <value>file:/root/hadoop-2.6.0/tmp</value>
    </property>
</configuration>
```
#### **修改hdfs-site.xml**
``` html
<configuration>
    <property>
        <name>dfs.namenode.secondary.http-address</name>
        <value>master:9001</value>
    </property>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:/root/hadoop-2.6.0/dfs/name</value>
    </property>
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:/root/hadoop-2.6.0/dfs/data</value>
    </property>
    <property>
        <name>dfs.replication</name>
        <value>3</value>
    </property>
</configuration>
```
#### **修改mapred-site.xml**
``` html
<configuration>
    <property>
      <name>mapreduce.framework.name</name>
      <value>yarn</value>
    </property>
</configuration>
```
#### **修改yarn-site.xml**
``` html
<configuration>
<!-- Site specific YARN configuration properties -->
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>    <value>org.apache.hadoop.mapred.ShuffleHandler</value>
    </property>
    <property>
        <name>yarn.resourcemanager.address</name>
        <value>master:8032</value>
    </property>
    <property>
        <name>yarn.resourcemanager.scheduler.address</name>
        <value>master:8030</value>
    </property>
    <property>
        <name>yarn.resourcemanager.resource-tracker.address</name>
        <value>master:8035</value>
    </property>
    <property>
        <name>yarn.resourcemanager.admin.address</name>
        <value>master:8033</value>
    </property>
    <property>
        <name>yarn.resourcemanager.webapp.address</name>
        <value>master:8088</value>
    </property>
</configuration>
```
将配置好的hadoop-2.6.0文件夹分发给所有slaves吧!
```
scp -r ~/hadoop-2.6.0 root@slave1:/root/
scp -r ~/hadoop-2.6.0 root@slave2:/root/
```
### **启动 Hadoop**
在 master 上执行以下操作，就可以启动 hadoop 了。
```
cd /root/hadoop-2.6.0     #进入hadoop目录
bin/hadoop namenode -format     #第一次运行之前，首先格式化namenode，后续再启动就不要格式化namenode了
sbin/start-dfs.sh               #启动dfs
sbin/start-yarn.sh              #启动yarn
```

### **验证 Hadoop 是否安装成功**
可以通过jps命令查看各个节点启动的进程是否正常。在 master 上应该有以下几个进程：
```
[root@master hadoop]# jps
1826 NameNode
2435 NodeManager
2341 ResourceManager
6423 Jps
2152 SecondaryNameNode
1951 DataNode
```
在slave上运行jps：
```
[root@slave1 ~]# jps
1905 NodeManager
1751 DataNode
5371 Jps
```
也可以在浏览器中输入
http://192.168.10.37:50070/
http://192.168.10.37:8088/

来查看hadoop相关的web界面。

如果web无法访问，试试centOS 7.0上关闭防火墙
```
systemctl stop firewalld.service
systemctl disable firewalld.service
```

### **测试HDFS**

hdfs创建文件：
```
bin/hdfs dfs -mkdir /test-hdfs
```
hdfs从本地上传文件：
```
bin/hadoop dfs -copyFromLocal /root/4300.txt /test-hdfs
```
查看上传的文件：
```
[root@master hadoop-2.6.0]#  bin/hdfs dfs -ls /test-hdfs
Found 1 items
-rw-r--r--   3 root supergroup    1573079 2015-07-22 16:50 /test-hdfs/4300.txt
```
也可以在web界面中查看。
![](http://7teb9r.com1.z0.glb.clouddn.com/hadoop-fs.png)

运行wordcounter 测试程序
```
[root@master hadoop-2.6.0]# bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.6.0.jar wordcount /test-hdfs output_wc3
```
测试一下mapreduce功能
```
root@master hadoop-2.6.0]# bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.6.0.jar pi 2 2
```
输出：
```
Number of Maps  = 2
Samples per Map = 2
Wrote input for Map #0
Wrote input for Map #1
Starting Job
15/07/22 11:04:00 INFO client.RMProxy: Connecting to ResourceManager at master/192.168.10.37:8032
15/07/22 11:04:00 INFO input.FileInputFormat: Total input paths to process : 2
15/07/22 11:04:00 INFO mapreduce.JobSubmitter: number of splits:2
15/07/22 11:04:01 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1436319535233_0004
15/07/22 11:04:01 INFO impl.YarnClientImpl: Submitted application application_1436319535233_0004
15/07/22 11:04:01 INFO mapreduce.Job: The url to track the job: http://master:8088/proxy/application_1436319535233_0004/
15/07/22 11:04:01 INFO mapreduce.Job: Running job: job_1436319535233_0004
15/07/22 11:04:09 INFO mapreduce.Job: Job job_1436319535233_0004 running in uber mode : false
15/07/22 11:04:09 INFO mapreduce.Job:  map 0% reduce 0%
15/07/22 11:04:17 INFO mapreduce.Job:  map 100% reduce 0%
15/07/22 11:04:25 INFO mapreduce.Job:  map 100% reduce 100%
15/07/22 11:04:26 INFO mapreduce.Job: Job job_1436319535233_0004 completed successfully
```
