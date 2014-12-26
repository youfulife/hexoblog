title: libpcap mmap 详解0 
date: 2014-12-26 15:39:13
tags: libpcap
categories: libpcap
---
libpcap从1.1之后支持PACKET socket接口提供的mmap()方式从内核读取数据包，大大提高了数据包的采集速度。
PACKET_MMAP提供了一个映射到用户空间的大小可以配置的ring buffer，这样就可以减少一次内核和用户层之间的memcpy copy，这里说zero copy是指内核到用户层的zero copy，而数据包要到达packet_rx_ring是需要从驱动copy到内核一次的。而且还可以进行批处理，减少系统调用。使用mmap方式的同时最好再使用NAPI的方式收取数据包，可以进一步提高效率。

如果不采用mmap方式，抓包效率非常低下，每抓一个包都需要一次系统调用，如果还要获得包的timestamp，那么就需要两次系统调用。
<!-- more -->
使用PACKET_MMAP的收包和发包的主要流程如下：
``` c
[setup]     socket() -------> creation of the capture socket
            setsockopt() ---> allocation of the circular buffer (ring)
                              option: PACKET_RX_RING
            mmap() ---------> mapping of the allocated buffer to the
                              user process

[capture]   poll() ---------> to wait for incoming packets

[shutdown]  close() --------> destruction of the capture socket and
                              deallocation of all associated 
                              resources.
```
每一个抓的frame都有下面两部分组成：
``` c
 --------------------
| struct tpacket_hdr | Header. It contains the status of
|                    | of this frame
|--------------------|
| data buffer        |
.                    .  Data that will be sent over the network                             interface.
.                    .
 --------------------
```
用户层通过setsockopt来设置启用PACKET_MMAP方式：
``` c
setsockopt(fd, SOL_PACKET, PACKET_RX_RING, (void *) &req, sizeof(req))
```
这个里面最主要的参数就是req参数，这个数据结构的定义如下，位于头文件**/usr/include/linux/if_packet.h**中：
``` c
struct tpacket_req
    {
        unsigned int    tp_block_size;  /* Minimal size of contiguous block */
        unsigned int    tp_block_nr;    /* Number of blocks */
        unsigned int    tp_frame_size;  /* Size of frame */
        unsigned int    tp_frame_nr;    /* Total number of frames */
    };

struct tpacket_req3 {
        unsigned int tp_block_size; /* Minimal size of contiguous block */
        unsigned int tp_block_nr; /* Number of blocks */
        unsigned int tp_frame_size; /* Size of frame */
        unsigned int tp_frame_nr; /* Total number of frames */
        unsigned int tp_retire_blk_tov; /* timeout in msecs */
        unsigned int tp_sizeof_priv; /* offset to private data area */
        unsigned int tp_feature_req_word;
};
```
这样就已经准备好创建一个mmap的环形缓冲区了，之后用户层读取数据包和时间戳就不需要系统调用了。从头文件的定义中可以看到frame的数据结构，这里的frame不是单纯的L2的frame，这个frame包含了PACKET_MMAP自己的一些东西。
``` c
/*
   Frame structure:

   - Start. Frame must be aligned to TPACKET_ALIGNMENT=16
   - struct tpacket_hdr
   - pad to TPACKET_ALIGNMENT=16
   - struct sockaddr_ll
   - Gap, chosen so that packet data (Start+tp_net) aligns to 
     TPACKET_ALIGNMENT=16
   - Start+tp_mac: [ Optional MAC header ]
   - Start+tp_net: Packet data, aligned to TPACKET_ALIGNMENT=16.
   - Pad to align to TPACKET_ALIGNMENT=16
 */
```
重点关注**struct tpacket_hdr**这个数据结构，这个数据结构主要就是L2 frame的一些metadata，比如timestamp，snaplen，status等。
之后就是调用平时经常见到的系统调用mmap()来完成内存映射了：
``` c
 mmap(0, size, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
```
ps: 以上内容主要来自https://www.kernel.org/doc/Documentation/networking/packet_mmap.txt

mmap ring的初始化主要是在activate_mmap函数中，下面详细分析这个函数的流程，以及mmap的一些思想。
activate_mmap()首先分配一个2M的ring buffer
``` c
if (handle->opt.buffer_size == 0) {
/* by default request 2M for the ring buffer */
handle->opt.buffer_size = 2*1024*1024;
}
```
然后设置mmap buffer中 tpacket header的类型和长度。
``` c
ret = prepare_tpacket_socket(handle);
```
这个函数中会根据内核头文件中的宏定义查看支持的tpacket的类型，优先采用最高级的类型也就是Tpacket V3, 具体的类型获取和设置是在init_tpacket()函数中通过调用getsockopt和setsockopt来完成的。

头部设置完成后开始建立 mmap ring 了，这个过程是在函数 create_ring() 中完成的，这个函数是 pcap mmap 建立过程中最重要的函数。
首先根据链路层的类型，TPACKET类型，初始化 struct tpacket_req 数据结构，初始化完成后就调用setsockopt函数进行mmap设置了。
设置成功后，调用mmap()系统调用，完成内存映射：
``` c
/* memory map the rx ring */
handlep->mmapbuflen = req.tp_block_nr * req.tp_block_size;
handlep->mmapbuf = mmap(0, handlep->mmapbuflen, PROT_READ|PROT_WRITE, MAP_SHARED, handle->fd, 0);
```
映射成功后创建一个缓冲区，用来存储frame header的地址并且进行初始化。
``` c
/* allocate a ring for each frame header pointer*/
handle->cc = req.tp_frame_nr;
handle->buffer = malloc(handle->cc * sizeof(union thdr *));
/* fill the header ring with proper frame ptr*/
handle->offset = 0;
for (i=0; i<req.tp_block_nr; ++i) {
    void *base = &handlep->mmapbuf[i*req.tp_block_size];
    for (j=0; j<frames_per_block; ++j, ++handle->offset) {
        RING_GET_FRAME(handle) = base;
        base += req.tp_frame_size;
    }
}
handle->bufsize = req.tp_frame_size;
handle->offset = 0;
```
create_ring()成功返回后，表示以后数据包的读取都是mmap的方式了，终于高大上了，所以相应的各种操作数据包的方法也得跟着变了，都使用mmap特有的方法了，这里主要是read和cleanup。
OK，到现在为止pcap_open_live()函数就算结束了，pcap的初始化工作也就完成了。
下面要开始数据包的抓取了。