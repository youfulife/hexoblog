title: libpcap socket 初始化详解
date: 2014-12-19 22:38:36
tags: libpcap
categories: libpcap
---
libpcap使用linux kernel提供的socket套接字来进行数据包的抓取。
``` c
int socket(int domain, int type, int protocol);
```
参数说明：

    domain是协议族
    type是通信双方使用的socket类型
    protocol是指定socket使用的具体协议

搞过web服务器或者客户端编程的人对这个应该很熟悉：
``` c
socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
socket(AF_INET, SOCK_STREAM, IPPROTO_UDP);
```
通过这两个socket可以读取linux 内核tcp/ip协议栈处理过的L4层的payload。
而要从网卡驱动直接抓取L2层的包，那么需要使用AF_PACKET协议族。
man packet 命令可以看到相关的使用方法。
 http://man7.org/linux/man-pages/man7/packet.7.html
 <!-- more -->
简要解释：
``` c
packet_socket = socket(AF_PACKET, int socket_type, int protocol);
```
AF_PACKET是linux定义的协议族的一种，表示从L2来抓取或者发送数据包，这样用户就可以通过自己的协议栈来处理数据包了。
最主要的是第二个参数**socket_type**，有两种type供选择， **SOCK_RAW**和**SOCK_DGRAM**。主要区别如下：

SOCK_RAW: 包括了L2的头部。
SOCK_DGRAM: 去掉了L2的头部，将L2的信息放到了一个数据结构sockaddr_ll中。

第三个参数protocol是指以太网的协议类型，这个数值是网络序的。我们可以设置成htons(ETH_P_ALL)，表示所有类型的包都接收。具体的L2类型编号在<linux/if_ether.h>头文件中。
下面来看一下下面这个数据结构：
``` c
The sockaddr_ll is a device independent physical layer address.
struct sockaddr_ll {
    unsigned short sll_family; /* Always AF_PACKET */
    unsigned short sll_protocol; /* Physical layer protocol */
    int sll_ifindex; /* Interface number */
    unsigned short sll_hatype; /* ARP hardware type */
    unsigned char sll_pkttype; /* Packet type */
    unsigned char sll_halen; /* Length of address */
    unsigned char sll_addr[8]; /* Physical layer address */
};
```
注意：AP_PACKET是linux 2.2之后的内核中出现的新的特性，早期的版本仅仅支持SOCK_PACKET, libpcap为了兼容，两种方式都支持，不过我觉得目前通用的抓包平台已经大部分都是2.6之后的版本了。

libpcap中socket相关设置在函数active_new()中，主要的流程如下：
``` c
/*
* Open a socket with protocol family packet. If the
* “any” device was specified, we open a SOCK_DGRAM
* socket for the cooked interface, otherwise we first
* try a SOCK_RAW socket for the raw interface.
*/
sock_fd = is_any_device ?
              socket(PF_PACKET, SOCK_DGRAM, htons(ETH_P_ALL)) :
              socket(PF_PACKET, SOCK_RAW, htons(ETH_P_ALL));
```
首先判断我们是不是从所有网卡上抓包，如果是，那么用SOCK_DGRAM，如果指定的是特定的网口，那么使用SOCK_RAW。
如果成功，那么说明内核支持PF_PACKET的方式，设置不使用sock_packet的标志位：
``` c
handlep->sock_packet = 0;
```
获取并设置 lo虚拟网卡的内核ifindex，目前libpcap不支持loopback device是别的名字。为什么要单独获取这个呢，因为如果指定读取所有网卡上的数据，那么libpcap不会抓取用于本地通信的lo网口上的数据包。
如果指定了特定的网口，再返回来设置要不要使用SOCK_RAW type。如果是libpcap hold不住的网卡类型，那么libpcap会保守的采用cooked模式。
首先根据ioctl得到硬件类型, 写过驱动的对ioctl应该很熟悉：
``` c
arptype = iface_get_arptype(sock_fd, device, handle->errbuf);
```
将内核返回的arptype映射为libpcap自己的硬件类型表示法dlt_type。
``` c
map_arphrd_to_dlt(handle, arptype, 1);
```
判断硬件类型是能不能搞定，如果不够搞定或者返回的硬件类型libpcap不认识，再次调用socket, 并且把cooked设置为1：
``` c
sock_fd = socket(PF_PACKET, SOCK_DGRAM, htons(ETH_P_ALL));
handlep->cooked = 1;
```
之后就调用bind函数将得到的sock_fd和ifindex进行绑定。如果没有指定特定的网口，那么做如下设置，sock_fd不绑定任何device：
``` c
handlep->cooked = 1;
handle->linktype = DLT_LINUX_SLL;
/*
* We’re not bound to a device.
* For now, we’re using this as an indication
* that we can’t transmit; stop doing that only
* if we figure out how to transmit in cooked
* mode.
*/
handlep->ifindex = -1;
```

下面就是通过setsockopt来设置一些socket的特性了。首先来看一下setsockopt和getsockopt的声明：
``` c
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>
int getsockopt(int sockfd, int level, int optname, void *optval, socklen_t *optlen);
int setsockopt(int sockfd, int level, int optname, const void *optval, socklen_t optlen);
```
关于具体的这两个函数的参数用法以及各种选项的作用，请自行阅读《unix网络编程 套接字API》第七章。
Packet socket选项的设置是通过level SOL_PACKET来控制的。

###1.设置混杂模式
指定optname为PACKET_ADD_MEMBERSHIP，第四个参数optval指向一个packet_mreq类型的结构体，这个结构体中需要指定特定网络接口index以及PACKET_MR_PROMISC网卡行为。
``` c
struct packet_mreq {
     int            mr_ifindex;    /* interface index */
     unsigned short mr_type;       /* action */
     unsigned short mr_alen;       /* address length */
     unsigned char  mr_address[8]; /* physical layer address */
};
```
如果是传统的SOCK_PACKET套接口，首先使用SIOCGIFFLAGS iotcl获取已有的网口标志位，然后或(||)上SIOCADDMULTI，再使用SIOCSIFFLAGS ioctl设置标志完成。
###2.看内核是否支持辅助数据
socket 选项PACKET_AUXDATA 是linux 2.6.21之后引入的，如果这个打开了这个选项，那么每一个数据包都会添加一个当前包的一些metadata，这些metadata可以通过cmsg来读取。metadata数据结构的定义如下：
``` c
struct tpacket_auxdata {
                      __u32 tp_status;
                      __u32 tp_len;      /* packet length */
                      __u32 tp_snaplen;  /* captured length */
                      __u16 tp_mac;
                      __u16 tp_net;
                      __u16 tp_vlan_tci;
                      __u16 tp_padding;
                  };
```
###3.设置rx buffer size
``` c
if (handlep->cooked) {
    if (handle->snapshot < SLL_HDR_LEN + 1)
        handle->snapshot = SLL_HDR_LEN + 1;
}
handle->bufsize = handle->snapshot;
```
###4.根据硬件网口的类型，设置vlan的offset，为vlan tag预留出位置来。
###5.设置时间精度
通过socket timestamp选项SO_TIMESTAMPNS，设置时间精度为nanosecond。这个时间戳有可能是软件生产的也有可能是硬件生成的，具体看网卡是否支持硬件时间戳。

linux内核关于数据包时间戳的详细描述：
https://www.kernel.org/doc/Documentation/networking/timestamping.txt
 
OK, active_new()函数到此就结束了，libpcap读取数据包的socket也都搞定了，下一步就是分配数据包的存储缓冲区了。