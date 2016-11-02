title: 出墙术
date: 2015-03-30 10:10:01
tags: vpn
categories: 编程
---

![](http://7sbqk1.com1.z0.glb.clouddn.com/google.jpg)

作为一个程序员，翻墙术是一个无法避免的技能，就跟会写代码一样重要。
过去的几年尝试了一些翻墙术，主要有linode自己搭建vps，goagent，GreenVpn等。
vps主要是国外的，当时是日本的机房，延迟大概100多ms，挺慢的，而且不便宜。
goagent不稳定，需要定时更新，上传证书，没事更新一下ip地址，毕竟是免费的，做到这样已经相当不错了。
GreenVpn各种各样的山寨版，搞不清楚哪个是正版，试用了一下IOS的客户端，连接速度还行，但是关闭了之后应该有后台进程运行，导致上不了网，手机重新启动后才行。

本来想用曲径的vpn服务，无奈现在曲径的vpn只有企业版的了，找了一圈也没找到个人的注册地址，估计不提供新的个人用户服务了。

搜索一把，发现有个叫红杏的chrome翻墙插件，试用了一段时间感觉不错，缺点是只能在chrome下用，也没有对应的手机客户端，不过作为一个偶尔登录一下1024的程序员来讲已经够用了。
10块钱一个月，方便快捷，没有goagent那种繁琐的操作，省钱省力，不过安全性不知道如何，上google查资料是够了，尽量不要用来支付。

下面是官方版的链接：http://honx.in/i/VRipUYkWGhWQjlk6

百度上直接搜的很多都是山寨的。
</br>
---
<!-- more -->
下面是2014年早期的一篇vps使用经验的文章，直接贴在下面：

最近关于google的一切都不管用了， 最喜欢用的google浏览器不得不在地址栏先输入baidu，然后再用百度搜索技术性的文章， 真是令人恼火啊。
听说升级最新的goagent就好用了， 关键是google把gmail的服务也屏蔽了，上传不上去啊。后来， 某一个瞬间，可能管理员松懈了几分钟，终于最新的goagent上传上去了，赶紧兴奋的打开浏览器试一试，发现还是不行，看来这次google真的悲剧了。
后来在freebuf上看到修改goagent中的iplist可以，后来试了一下，确实可以，但是过了大概半个小时，发现又不行了。
后来想起mactalk推荐过有个叫曲径的比较靠谱的收费vpn，发现竟然不支持linux，只支持mac，windows，手机。59块钱一个月，好悲惨，花钱都没地方花啊。
无聊翻看知道创宇研发技能表，发现有个翻墙的技能， 里面有ssh隧道技术。瞬间泪流满面啊。
我的VPS每月3000G的流量啊，竟然之前仅仅用来写一些没几个人访问的博客，真是天大的浪费啊，太可耻了。

果断打开ssh动态转发功能：
```
sudo ssh -N -f -D 8888 user@romote-ip
```
浏览器中用switchsharp配置好socks5代理，点击百度，google，facebook，哇，速度飞快啊。

这里注意的是只能填写Socks Host那一栏， 前面的HTTP proxy等千万不要填写。一开始不知道，全部填写了127.0.0.1 和 port 8888，或者选择了use the same proxy server for all protocols， 发现根本连不上，后来通过tcpdump和wireshark抓包分析原因，发现根本就没有通过ssh的22端口出去，但是已经转发到了ssh监听的8888端口上， 问题肯定出在本机的ssh 客户端。
仅仅配置socks 那一栏， 选择socks v5发现可以了。socks v5是socks v4的升级版， 都是4层的代理， 支持UDP协议, ipv6, 更加安全可靠。

如果通过vps上google被识别为机器人，那么不要慌张，不要骂娘，是ipv6导致的，只要将vps的ipv6禁止了就可以了。
ubuntu 14.04禁止ipv6的方法如下：
打开/etc/sysctl.conf文件，在文件最后添加如下一行配置：
```
net.ipv6.conf.all.disable_ipv6=1
```
运行下面命令使之生效：
```
sysctl -p
```
通过ifconfig 发现没有了inet6的相关信息。
```
cat  /proc/sys/net/ipv6/conf/all/disable_ipv6
```
发现 内核中的值也是1了。

windows系统下可以下一个putty 工具包，里面有一个plink.exe的程序，在cmd中进入到putty工具包目录下运行下面命令：
```
plink -N -D 8888 user@romote-ip
```
从此之后告别goagent了。
