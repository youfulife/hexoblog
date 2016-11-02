title: elasticsearch ubuntu下最大文件描述符解决
date: 2016-09-29 16:12:39
tags: ELK
categories: 编程
---

ElasticSearch 文件描述符问题是一个比较容易出现的问题，尤其是data节点上，默认应该是1024，根本不够用。
由于 ElasticSearch 的运行用户只能是非root用户，需要创建一个非root权限的elasticsearch用户来运行。
具体的集群环境搭建参考之前的《Elastic Search 2.0 集群搭建》博文。

安装环境是ubuntu 14.04 x86_64。

使用supervisor管理es的进程启动，配置如下：

```
[program:elasticsearch]
command=sudo -E -u elasticsearch /opt/elasticsearch/bin/elasticsearch
environment=ES_HEAP_SIZE="8g"
startsecs=3
autostart=true
autorestart=true
startretries=3

```
sudo 一定要加上-E 选项，表示继承当前启动的环境变量，这样supervisor中设置的environment才能起到作用。
-u elasticsearch 选项表示以elasticsearch用户来运行。

启动之后通过es的进程id来查看es最大的描述符数量
```
cat /proc/9144/limits |grep files
```
一般设置为65535就可以了，4096是肯定不够的。

ubuntu下如何设置一个进程的文件描述符呢？尤其是通过supervisor托管，而且还是sudo启动的进程，而且运行这个程序的用户还没有登陆权限，这个还是比较复杂的，很多博文里提到的linux下设置文件描述符的方式一般是指redhat系列的操作系统，在这里都不管用。

首先修改 /etc/security/limits.conf 文件，在文件最后面添加下面4行，这个是大部分文章中都提到的。
```
root soft nofile 65535
root hard nofile 65535
* soft nofile 65535
* hard nofile 65535
```
表示root用户和其余非root用户打开的文件描述符的限制都是65535。

在redhat系列下可能设置了这个然后重启机器或者重新登陆就有用了，但是ubuntu下不行。还要设置另外两个配置文件。
修改/etc/pam.d/su 和 /etc/pam.d/sudo，两个文件都添加下面一行

```
session    required   pam_limits.so
```

其中su文件中已经有这一行了，只要把前面的注释去掉就可以了，sudo文件需要自己添加。
注意，如果只修改su文件，那么针对ES的这种情况还是没用，必须设置sudo文件才行。
其余的比如/etc/sysctl.conf等文件压根没必要设置，还有系统默认可以最大打开的文件描述符也没必要设置，这个值一般都非常大。

设置完成后就重新启动一下机器吧。

再次通过查看proc下面的进程信息，发现设置生效了。
