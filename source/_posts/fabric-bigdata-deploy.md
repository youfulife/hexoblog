title: 基于fabric的集群自动部署
date: 2016-03-30 14:21:32
tags: python, elk
categories: 编程
---

最近基于fabric实现了一个大数据分析组件的多节点自动部署项目hunter。

目前主要处理组件：
openresty：主要用作代理，数据预处理，权限验证等。
redis：配合nginx使用，主要作为部分数据的缓存。
kafka：数据传输队列，所有数据都会先存到kafka中。zookeeper：跟kakfa绑定在一起的，并没有做其他用途。
elasticsearch：最终数据存储，索引，目前将master，client，data节点全部分在不同的node上。
mongodb：数据存储。
kibana：数据展示。
kepi: 自己实现的kafka->es/mongodb的数据shipper，类似于logstash，heka的角色。
shadow: 自己实现的es和mongodb离线处理脚本，可以自动生成es和mongodb的查询语句。
crobjob: 一些后台的crontab定时执行脚本。

目前的功能主要包括各个模块的代码自动分发，jinja2模版配置文件，基于节点环境自动修改，基于supervisor
来管理各个模块的进程启动和监控，部署成功后通过脚本测试整个集群的数据连通性。

示例配置文件如下：

```
[DEFAULT]
user = root
password = 123456
deploy_dir = /opt
url = git@192.168.10.43:BigData/%(repo)s.git
template = prey/templates/%(conf)s
deploy = on

[dawn]
repo = dawn
hosts = ["192.168.10.79"]
conf = nginx.conf
;deploy = off

[elasticsearch]
repo = elasticsearch
hosts = ["192.168.10.79"]
conf = elasticsearch.conf
deploy = off
es_client = ["192.168.10.79"]
es_data = ["192.168.10.79"]
es_master = ["192.168.10.79"]

```

如果某个模块部署到多个节点，那么只要在相应的模块下面的hosts列表中添加ip地址就可以了。
目前暂时只支持从git下载源代码。

其中elastic search的配置文件模版如下：
```
cluster.name: elasticsearch
node.name: {{ node_name }}
node.master: {{ is_master_node }}
node.data: {{ is_data_node }}
http.enabled: {{ http_enabled}}
network.host: _non_loopback_
discovery.zen.ping.multicast.enabled: false
discovery.zen.ping.unicast.hosts: {{ cluster_hosts }}
discovery.zen.minimum_master_nodes: {{ min_master_nodes }}
script.inline: on
script.indexed: on
script.file: on
```
部署过程会根据配置文件和节点的环境自动填充相应的变量。
后续会整理一个比较好的支持开源组件的版本发布到github上。

最后的效果就是配置好ip地址和用户名密码之后运行下面两个命令，不需要手动修改任何组件的任何配置文件，集群就轰隆隆的跑起来了。

```
fab deploy
fab start
```

目前用来实时处理大量网络数据，整个集群（目前是17台阿里云主机）的每个组件都可以横向扩展，而且线上已经稳定运行3个月了。
