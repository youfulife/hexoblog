title: libpcap mmap收包详解
date: 2014-12-14 10:10:58
tags: libpcap
categories: libpcap
---
libpcap用户层收包的过程主要是调用pcap_loop函数，pcap_loop通过调用read_op回调函数来完成收包。初始化的过程中根据内核提供的支持，read_op的值也有所不同。
这里主要关注最快的PACKET V3下的mmap收包方式。在active_mmap中，read_op会被初始化为下面的函数：
``` c
#ifdef HAVE_TPACKET3
    case TPACKET_V3:
    handle->read_op = pcap_read_linux_mmap_v3;
    break;
#endif
```
<!-- more -->
这些宏定义如果内核支持会在内核头文件中给出，不需要用户自己指定，内核不支持的话自己指定了也没用。
下面主要关注pcap_read_linux_mmap_v3的流程：
首先要知道V3的数据包在用户层和内核之间的管理是安照block来进行的，一个block中可以有很多的frame，当处理完这个block中的所有frame后才把这个block返回给kernel。
首先判断ring buffer的状态是不是TP_STATUS_USER，如果是的话就说明有可用的packet了。
``` c
/* wait for frames availability.*/
ret = pcap_wait_for_frames_mmap(handle);
```
在while循环中获得ring buffer中当前block的第一个packet frame,设置handlep中相关的field，这里主要有下面几个域：
``` c
struct pcap_linux{
    …
#ifdef HAVE_TPACKET3
    unsigned char *current_packet; /* 当前block中的V3 frame指针，如果NULL，说明当前block中没有数据包了，要去下一个block中取了*/
    int packets_left; /* 当前block中还没有处理的数据包*/
#endif
…
}
```
相关代码：
``` c
if (handlep->current_packet == NULL) {
    //获得下一个block的地址
    h.raw = pcap_get_ring_frame(handle, TP_STATUS_USER);
    if (!h.raw)
        break;
    //设置current_packet指向当前block的第一个frame
    handlep->current_packet = h.raw + h.h3->hdr.bh1.offset_to_first_pkt;
    //设置packets_left为当前block中的全部frame的个数
    handlep->packets_left = h.h3->hdr.bh1.num_pkts;
}
```
下面就是在一个while循环中依次处理当前block中的所有数据包，数据包的主要处理是在pcap_handle_packet_mmap()函数中，这个函数中主要有packet的bpf_filter功能，Vlan处理功能，针对SOCK_DGRAM 模式下sll 头部的设置功能，以及最终将2层数据包传递给用户callback的功能，这个callback就是在pcap_look中传递进来的回调函数。callback()返回后，
pcap_handle_packet_mmap紧接着也返回了，标志着一个数据包的彻底处理完成。
``` c
int packets_to_read = handlep->packets_left;
while(packets_to_read–) {
    struct tpacket3_hdr* tp3_hdr = (struct tpacket3_hdr*) handlep->current_packet;
    pcap_handle_packet_mmap(……);
    ……
    handlep->current_packet += tp3_hdr->tp_next_offset;
    handlep->packets_left–;
}
```
一个block全部处理完成后，首先将这个block设置标志位交还给内核继续使用，然后继续下一个block
``` c
if (handlep->packets_left <= 0) {
    //设置好标志位后内核就可以继续使用了
    h.h3->hdr.bh1.block_status = TP_STATUS_KERNEL;
    ……..
}
/* next block */
if (++handle->offset >= handle->cc)
handle->offset = 0;
```
具体的fame的结构如下：
``` c
/*
   Frame structure:

   - Start. Frame must be aligned to TPACKET_ALIGNMENT=16
   - struct tpacket_hdr
   - pad to TPACKET_ALIGNMENT=16
   - struct sockaddr_ll
   - Gap, chosen so that packet data (Start+tp_net) aligns to TPACKET_ALIGNMENT=16
   - Start+tp_mac: [ Optional MAC header ]
   - Start+tp_net: Packet data, aligned to TPACKET_ALIGNMENT=16.
   - Pad to align to TPACKET_ALIGNMENT=16
 */
```
只要把相关的V3 packet的数据结构和mmap的流程搞清楚，收包的流程还是挺简单的。