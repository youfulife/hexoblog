title: linux Packet socket (1)简介
date: 2014-12-20 14:15:20
tags: libpcap
categories: libpcap
---
本文主要来自于linux自带的man packet手册：
http://man7.org/linux/man-pages/man7/packet.7.html
平时经常使用的INET套接字提供的是7层的抓包能力，抓上来的data直接就是tcp或者udp的payload，无需关心L3和L4的头部信息。Packet套接字提供的是L2的抓包能力，也叫raw socket，意思就是不经过操作系统tcp/ip协议栈处理的packet，抓上来的包需要自己处理tcp/ip的头部信息。

目前使用packet套接字的主要有libpcap，netsniﬀ-ng，hostapd（hostapd是一个用户层的无线AP管理程序）。

linux提供了的packet 套接字函数API如下：
```
       #include <sys/socket.h>
       #include <netpacket/packet.h>
       #include <net/ethernet.h> /* the L2 protocols */

       packet_socket = socket(AF_PACKET, int socket_type, int protocol);
```
socket_type有SOCK_RAW 和 SOCK_DGRAM，这两个的主要区别是2层的头部处理。
<!-- more -->
如果指定SOCK_RAW那么我们得到的数据包含所有的L2 header和payload，
如果指定SOCK_DGRAM, 那么我们收到的数据会去掉L2的header，是IP header和payload。
二层的头部信息会放到一个通用的struct sockaddr_ll结构体中。

protocol主要是**linux/if_ether.h**中定义的协议类型，我们可以指定ETH_P_IP来抓取IP packet，ETH_P_ARP 来抓取ARP的packet，一般情况下我们可以指定ETH_P_ALL来抓取所有类  型的packet。
注意：传入参数的时候应该转化成网络字节序htons(ETH_P_ALL)。

sockaddr_ll结构体用来表似乎一个设备独立的物理层地址信息，定义如下：
```
struct sockaddr_ll {
               unsigned short sll_family;   /* Always AF_PACKET */
               unsigned short sll_protocol; /* Physical layer protocol */
               int            sll_ifindex;  /* Interface number */
               unsigned short sll_hatype;   /* ARP hardware type */
               unsigned char  sll_pkttype;  /* Packet type */
               unsigned char  sll_halen;    /* Length of address */
               unsigned char  sll_addr[8];  /* Physical layer address */
           };
```
每个域的定义如下：

sll_family:  总是AF_PACKET。

ssll_protocol: <linux/if_ether.h>中定义的那些协议类型，也就是我们传给socket的第二个参数，注意是网络序。

sll_ifindex: 内核中网卡的index，定义在ifreq结构体中，可以参考下面的链接：
http://man7.org/linux/man-pages/man7/netdevice.7.html
if_nametoindex()函数提供了从网卡名到index的转换，后面的示例代码中会用到这个函数。如果man找不到这个函数用法，那么需要安装 manpages-posix-dev 。

sll_hatype: ARP硬件类型，在头文件<linux/if_arp.h>中定义，比如ARPHRD_ETHER表示10Mbps 的Ethernet网卡类型。内核使用ARPHDR_XXX来表示网卡类型。

sll_pkttype: 表示当前接收的数据包的类型，主要有下面几种合法的值：

       PACKET_HOST 发送给当前主机的包,
       PACKET_BROADCAST 广播数据包,
       PACKET_MULTICAST 多播数据包
       PACKET_OTHERHOST 由于网卡设置了混杂模式收到的发送给别的主机的包
       PACKET_OUTGOING 从本机发出的，不小心loopback到当前socket了
       这些类型只有接收的时候才有意义。

sll_halen: 表示当前mac地址的长度。

sll_addr: 存储当前的mac地址。

发送数据包的时候只要设置下面几个域就足够了：
sll_family, sll_addr, sll_halen, sll_ifindex. 其余的都应该设置为0

sll_hatype 和 sll_pkttype在接收数据包的时候会被设置为当前数据包的信息。
对于bind()函数来说，只有sll_protocol 和 sll_ifindex会被用到。

