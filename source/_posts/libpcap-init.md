title: libpcap 初始化
date: 2014-12-19 22:16:52
tags: libpcap
categories: libpcap
---
libpcap 普通抓包主要调用两个函数:
``` c
pcap_t *pcap_open_live(const char *device, int snaplen, int promisc, int to_ms, char *errbuf);
int pcap_loop(pcap_t *p, int cnt, pcap_handler callback, u_char *user);
```
本文主要分析pcap_open_live函数的内部流程，这个函数完成libpcap抓包所需要的数据结构和方法初始化工作。

具体的函数调用流程以及函数功能分析如下：

    pcap.c : 759行        pcap_open_live
    pcap.c : 399行              pcap_create
    pcap.c: 576行               pcap_set_snaplen / promisc / timeout
    pcap.c: 716行               pcap_activate
    
首先分析pcap_create()函数的实现，这个函数主要功能是分配了pcap_t，pcap_linux等数据结构内存，初始化了相关的一些回调函数。
<!-- more -->
根据不同的#define, 有很多不同的pcap_create函数原型，这里仅仅关注linux下普通网络接口设备。

如果是普通的网卡设备，pcap_create中会调用pcap_create_interface() 这个函数，这个函数在不同的平台下会有不同的实现，这里也是仅仅关注linux平台下，这个函数的实现位于pcap-linux.c文件中。

pcap_create_interface主要完成了三个事情：

1. 调用pcap_create_common函数，完成了pcap_t handle的内存分配和一些初始化工作。
简单分析一下pcap_create_common函数，这个函数就是内存分配和初始化结构体的域，值得注意     的是，这里面多分配了sizeof(struct pcap_linux)内存，将 handle->priv 指向了这块内存。这种私        有数据的编程思想值得学习一番。这样的用法在linux驱动或者内核源码中到处都是。

2. 初始化下面这两个方法

    handle->activate_op = pcap_activate_linux;
    handle->can_set_rfmon_op = pcap_can_set_rfmon_linux;

3. 设置时间戳的来源以及精度
这里初始化了三种时间戳类型，从主机获取，网卡提供但没有同步到主机时间，网卡提供并且已经同步到主机时间。
主机获取主要通过一些time或者gettimeofday等系统调用，会降低pcap性能，尽量能不用就不用。pcap支持两种时间精度，macro second和nano second，默认是macro second。
后面会有一篇文章专门来讲述linux下网络程序时间戳的获取以及精度和计算方式。

pcap_create函数到这里就结束了，返回了一个pcap_t *p的指针。
如果没有任何问题的话，就根据用户传递进来的参数设置snaplen(捉包的最大包长)，promisic(混杂模式)，timeout(超时)。
 
基本设置完成后调用pcap_activate函数，这个函数主要是初始化各种方法。

    status = p->activate_op(p);
这个回调函数在前面pcap_create_interface()中设置：

    handle->activate_op = pcap_activate_linux;
这个函数将完成最后的壮举，稍有不慎，前功尽弃，设置了捉包的具体函数，捉包的具体方法等等一系列将产生重大影响的东西。

这个函数中值得重点关注的主要是下面几个方面：

1.设置了读取数据包的真正函数，后面会重点分析这个函数的具体实现。

``` c
handle->read_op = pcap_read_linux;
```
2.从linux kernel 2.2以后内核提供的直接从二层捉包的socket API就已经变化了，新的内核提供的是一个新的协议族PF_PACKET, 老的内核是使用socket type SOCK_PACKET，libpcap为了兼容，首先会尝试支不支持PF_PACKET, 如果不支持，那么再尝试SOCK_PACKET。
这两个函数里面会创建好socket 套接字，并且绑定好对应的网口。
```	c
status = activate_new(handle);
status = activate_old(handle));
```
3.如果支持PF_PACKET, 那么再尝试内核是否支持memory mmap access, 这个可以大幅度提升数据包的处理性能，如果发现支持memory mmap，那么创建好packet buffer后整个pcap_activate_linux函数直接就返回了。
``` c
activate_mmap(handle, &status)
```
4.如果不支持mmap，那么在套接字建立好之后会申请数据包的存储空间。成功后整个函数返回。
``` c
handle->buffer = malloc(handle->bufsize + handle->offset);
```
	
pcap_t *pcap_open_live()函数的整个流程就是这样, 最终返回一个pcap_t类型的指针，这个里面包涵了libpcap抓包需要的所有信息。
下一步就可以调用pcap_loop进行数据包的抓取了。