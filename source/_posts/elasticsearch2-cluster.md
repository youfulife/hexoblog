title: Elastic Search 2.0 集群搭建
date: 2015-11-03 11:47:54
tags: elk
---

Elastic Search 2.0 和 1.x 是不兼容的，其中很多的默认配置都修改了，直接下载下来运行只能
玩单机版了。

本文介绍Elastic Search 2.0下的集群搭建配置，目前使用了3台服务器。
各个组件的版本如下：
```
Kibana 4.2.0
Logstash 2.0.0
Elastic Search 2.0.0
```
<!-- more -->
## Elastic Search 配置
Elastic Search 2.0 为了安全性考虑，默认只监听本地127.0.0.1网口，而且1.X版本的多播方式
自动发现es节点也关闭了。

1.修改config/elasticsearch.yml文件，配置如下：
```
# network.host: 192.168.0.1
network.host: _non_loopback_
# The default list of hosts is ["127.0.0.1", "[::1]"]
discovery.zen.ping.multicast.enabled: false
discovery.zen.ping.unicast.hosts: ["es0", "es1", "es2"]
```
主要作用就是设置es监听非loopback网口，关闭multicast功能，配置unicast hosts, 需要将hosts修改成自己的es node的hostname。

2.Elastic Search 2.0 默认不支持root直接启动了，测试了一下1.7.2还是支持root直接启动的。
所以还需要添加一个非root的用户名。

```
export ES_HOME=/opt/elasticsearch
useradd -d $ES_HOME -s /bin/sh elasticsearch
chown -R elasticsearch $ES_HOME
```

3.修改 bin/elasticsearch 文件，文件开头添加下面三行：
```
ES_USER=elasticsearch
ES_GROUP=elasticsearch
ES_HEAP_SIZE=4g
```
默认的heap size太小，这里改成4g。

现在可以通过 elasticsearch 用户名来启动elastic search了。
```
sudo -u elasticsearch ./bin/elasticsearch
```
由于我使用root用户登录的，所以要sudo切换到elasticsearch用户启动程序。


## Kibana 配置

由于现在es并不会监听lo端口，就算kibana和es运行在一个机器上也不行，因为kibana默认会连接
http://localhost:9200，所以需要修改kibana的默认配置文件。

```
# The Elasticsearch instance to use for all your queries.
elasticsearch.url: "http://192.168.10.85:9200"
```
把其中的ip地址修改成集群中任何一个节点的ip或者host都可以。

然后就可以启动 kibana 了，ES 2.0 下 marvel 2.0监控单个集群状态免费了，并且集成到Kibana 4.2版本中了。

安装marvel过程如下：
1.在ES 2.0中安装
```
bin/plugin install license
bin/plugin install marvel-agent
```
2.在Kibana 4.2.0中安装
```
bin/kibana plugin --install elasticsearch/marvel/latest
```
启动Kibana后访问 http://192.168.10.85:5601/app/marvel 就可以看到集群的监控信息了。

## Logstash 配置

logstash配置也变有一定的变化，比如原来output中的host字符串变成hosts数组了。
不过基本配置和1.x版本差不多。
可以修改logstash.lib.sh 提高一下logstash的默认heap内存：

```
# Defaults you can override with environment variables
LS_HEAP_SIZE="${LS_HEAP_SIZE:=2000m}"
```

贴一个自己用的 从 kafka 读数据 输出到ealstic search的配置：
```
input {
    kafka {
        zk_connect => "zk-node1:2181,zk-node2:2181,zk-node3:2181"
        group_id => "logstash-test"
        type => "test"
        topic_id => "test"
        consumer_threads => 3
        #decorate_events => true
    }
}

output {
    #stdout {
    #    codec => rubydebug
    #    codec => dots
    #}
    elasticsearch {
        hosts => ["es0:9200", "es1:9200", "es2:9200"]
        index => "logstash-%{+YYYY.MM.dd}"
        codec => "json"
    }
}
```
