title: rsyslog存储json日志到kafka
date: 2016-04-27 10:56:01
tags: [rsyslog, kafka]
categories: 大数据分析
---

最近的数据分析平台需要添加syslog的分析功能，目标是通过分析syslog和网络的原始数据报文来做一个智能化的服务或设备的故障定位。

首先是收集日志，日志收集器有第三方探针Flume, Scribe, Logstash, Heka等等和采用系统自带的rsyslog探针。由于是对外的服务，所以采用跟日志易类似的方案，不在用户系统上按照任何第三方软件，只要配置一下rsyslog的转发规则即可。

日志收集上来需要转发到目前的数据平台，这里也有很多第三方的软件可以用，比如logstash，heka等等也就是shipper的作用，目前选用的还是rsyslog来接收，暂时没有考虑到数据量特别大的情况下分布式和集群的方案。目前来说rsyslog占用系统资源非常少，单机大概能处理到2w eps，一年内估计够用了。

rsyslog也有发送到kafka的模块，不过ubuntu 14.04默认没有安装。

### 添加输出kafka支持

1.参考 http://www.rsyslog.com/ubuntu-repository/, 升级到最新的稳定版本v8，默认14.04是v7的不支持kafka输出模块。

```
sudo add-apt-repository ppa:adiscon/v8-stable
sudo apt-get update
sudo apt-get install rsyslog
```
安装完成后运行rsyslogd -v看看版本是否已经是最新的稳定版本

2.安装kafka输出模块，imptcp模块，目前rsyslog的插件都是以deb或者rpm的包单独发布的，不需要从源码重新编译。
```
sudo apt-get install rsyslog-kafka rsyslog-imptcp
```
安装成功后 /usr/lib/rsyslog 目录下会出现相关的so库，rsyslog会从这个目录下去加载相关的动态库模块。

3.安装librdkafka依赖，从github下载最新代码编译安装，librdkafka最新代码支持最新的kafka 0.9和一些额外的配置。
```
git clone https://github.com/edenhill/librdkafka.git
cd librdkafka
./configure --prefix=/usr
make
sudo make install
```

### 修改rsyslog的配置文件
默认rsyslog的配置文件是/etc/rsyslog.conf 和 /etc/rsyslog.d 目录下的私人定制配置。
不要轻易修改全局的rsyslog.conf, 在 /etc/rsyslog.d 目录下新建一个 foo.conf 文件，主要功能有四个：
1.配置udp和tcp输入支持，端口号都是514。
2.增加一个自定义的json模版来输出自定义的格式，msg:2:$表示从第二个字符开始，这样会将msg开头的空格去掉。
3.配置输出到kafka集群。
4.过滤，只有通过udp和tcp模块进来的syslog才输出到kafka中。

具体配置内容如下：

```
module(load="imudp") # needs to be done just once
input(type="imudp" port="514")

module(load="imptcp") # needs to be done just once
input(type="imptcp" port="514")

$RepeatedMsgReduction off

$WorkDirectory /var/spool/rsyslog

template(name="jtpl"
         type="string"
         string="{%msg:2:$:jsonf%,%app-name:::jsonf:app_name%,%procid:::jsonf:pid%,%hostname:::jsonf%,%programname:::jsonf:pname%,%syslogfacility-text:::jsonf:facility%,%syslogseverity-text:::jsonf:severity%,%timereported:::date-rfc3339,jsonf%,%timegenerated:::date-rfc3339,jsonf%}\n"
        )

module(load="omkafka")
if $inputname == "imudp" or $inputname == "imtcp" then {
    action (type="omkafka" topic="rsyslog" broker="192.168.10.79" partitions.auto="on" template="jtpl" confParam=["compression.codec=snappy", "socket.keepalive.enable=true"])
}
```
具体配置相关的指令和语法直接官方文档就可以查看到。
配置修改完成后需要运行service rsyslog restart重新启动rsyslog使得配置生效。

### 验证配置生效

远端终端输入 logger "x" 命令发送一条log。查看本地的/var/log/syslog文件可以发现最终生成的log如下：
```
Apr 27 13:00:58 node-79 root: x
```

通过kafka的consumer来查看kafka中的数据直接就是json格式了。
目前使用的是kafkacat的命令行来查看：

```
kafkacat -C -b 192.168.10.79:9092 -t rsyslog
```
对应的log的kafka中的存储如下：
```
{"msg":"x","app_name":"root","pid":"-","hostname":"node-79","pname":"root","facility":"user","severity":"notice","timereported":"2016-04-27T13:00:58+08:00","timegenerated":"2016-04-27T13:00:55.872019+08:00"}
```

后面就可以接各种大数据处理平台对日志进行分析了。

### 后记
模版配置中的时间目前用的是rfc3339，也可以通过指定date-unixtimestamp来生成1970年到现在的秒数，只能精确到秒。

后面测试会基于当前配置用nginx的错误日志和access日志来作为输入源。

测试过程中还尝试了直接输出到elasticsearch，不过不管是通过源代码编译还是通过apt-get安装，es的rsyslog插件加载的时候会出现coredump，所以直接放弃了，通过gdb发现是传递给了里面的strlen函数一个NULL的参数，具体原因没有详查。
