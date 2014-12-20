title: libpcap mmap 详解2
date: 2014-12-20 13:49:39
tags: libpcap
categories: libpcap
---

本文主要内容来自：
https://www.kernel.org/doc/Documentation/networking/packet_mmap.txt

###1. 为什么要使用mmap？

PACKET_MMAP提供了一个映射到用户空间的大小可以配置的ring buffer，这样就可以减少一次内核和用户层之间的memcpy copy，这里说zero copy是指内核到用户层的zero copy，而数据包要到达packet_rx_ring是需要从驱动copy到内核一次的。而且还可以进行批处理，减少系统调用。
使用mmap方式的同时最好再使用NAPI的方式收取数据包，可以进一步提高效率。
如果不采用mmap方式，抓包效率非常低下，每抓一个包都需要一次系统调用，如果还要获得包的timestamp，那么就需要两次系统调用。

###2. 如何使用Packet mmap？

从系统调用来看，收包主要有下面几步：
```
[setup]     socket() -------> 创建套接字
            setsockopt() ---> 通过设置PACKET_RX_RING选项分配rx ring buffer
            mmap() ---------> mmap分配好的buffer到用户空间

[capture]   poll() ---------> 轮询模式等待数据包的到来

[shutdown]  close() --------> 关闭套接字释放资源

```
<!-- more -->
首先 通过socket系统调用创建一个Packet socket:
```
 int fd = socket(PF_PACKET, mode, htons(ETH_P_ALL));
```
其中mode可以选择SOCK_RAW或者SOCK_DGRAM, 关于详细的Packet socket的介绍请看之前的文章《libpcap socket 初始化详解》 [ http://youfu.xyz/2014/12/19/libpcap-socket/ ] 。

发包跟收包流程差不多：
```
[setup]          socket() -------> 创建发包套接字
                 setsockopt() ---> 通过设置PACKET_TX_RING选项分配tx ring buffer
                 bind() ------> 绑定套接字到具体的网口
                 mmap() ---------> mmap分配好的buffer到用户空间

[transmission]   poll() ---------> 等待数据包可发送 (可选)
                 send() ---------> 发送tx ring中可以发送的数据包
                                   
[shutdown]       close() --------> 关闭套接字释放资源
```
也是通过socket先创建一个socket套接字：
```
int fd = socket(PF_PACKET, mode, 0);
```
如果我们仅仅想通过这个socket发包的话，参数协议类型可以设置为0，这样就不会去调用packet_rcv()了，当然，这里设置成0，在bind的时候，struct sockaddr_ll中的sll_protocol也要是0。否则，可以设置为htons(ETH_P_ALL)或者其他别的协议。
发包socket如果要使用mmap必须强制绑定到某个网卡，这样就可以获取到tx ring每个frame的头部大小。
和收包一样，每个等待发送的frame有两个部分组成：
```
 --------------------
| struct tpacket_hdr | 头部，包含当前frame各种状态
|--------------------|
| data buffer        |
.                    . 即将通过网卡发送出去的真正数据
 --------------------
```
bind系统调用通过struct sockaddr_ll中的sll_ifindex 将socket和具体的网卡联系起来。
初始化示例代码：
```
 struct sockaddr_ll my_addr;
 struct ifreq s_ifr;
 ...
 strncpy (s_ifr.ifr_name, "eth0", sizeof(s_ifr.ifr_name));
 /* get interface index of eth0 */
 ioctl(this->socket, SIOCGIFINDEX, &s_ifr);

 /* fill sockaddr_ll struct to prepare binding */
 my_addr.sll_family = AF_PACKET;
 my_addr.sll_protocol = htons(ETH_P_ALL);
 my_addr.sll_ifindex =  s_ifr.ifr_ifindex;

 /* bind socket to eth0 */
 bind(this->socket, (struct sockaddr *)&my_addr, sizeof(struct sockaddr_ll));
```
用户的数据应该放在**frame base + TPACKET_HDRLEN – sizeof(struct sockaddr_ll)**处，这样不管你选择什么样的**socket mode(SOCK_DGRAM 或 SOCK_RAW)**, 用户要发送的数据的偏移量如下：
```
frame base + TPACKET_ALIGN(sizeof(struct tpacket_hdr))
```

###3. Packet mmap选项设置
用户层设置PACKET_MMAP通过setsockopt系统调用：
```
 - Capture process
     setsockopt(fd, SOL_PACKET, PACKET_RX_RING, (void *) &req, sizeof(req))
 - Transmission process
     setsockopt(fd, SOL_PACKET, PACKET_TX_RING, (void *) &req, sizeof(req))
```
其中最重要的参数req结构体定义如下：
```
    struct tpacket_req
    {
        unsigned int    tp_block_size;  /* Minimal size of contiguous block */
        unsigned int    tp_block_nr;    /* Number of blocks */
        unsigned int    tp_frame_size;  /* Size of frame */
        unsigned int    tp_frame_nr;    /* Total number of frames */
    };
```
这个结构体定义在**/usr/include/linux/if_packet.h**头文件下。
创建好的mmap的ring buffer是按照block来组织的，每一个block是物理地址连续的内存区域。每一个block可以存放有一个或者多个frame。
其中结构体中的tp_frame_nr是一个冗余信息，可以由其他三个域推算出来。
```
    frames_per_block = tp_block_size/tp_frame_size
    frames_per_block * tp_block_nr == tp_frame_nr
```
示例如下：
```
     tp_block_size= 4096
     tp_frame_size= 2048
     tp_block_nr  = 4
     tp_frame_nr  = 8
```
得到ring buffer的结构:
```

        block #1                 block #2         
+---------+---------+    +---------+---------+    
| frame 1 | frame 2 |    | frame 3 | frame 4 |    
+---------+---------+    +---------+---------+    

        block #3                 block #4
+---------+---------+    +---------+---------+
| frame 5 | frame 6 |    | frame 7 | frame 8 |
+---------+---------+    +---------+---------+
```
一个frame不能跨越两个block。

###4. Packet mmap选项设置的约束条件
在 2.4.26 (for the 2.4 branch) and 2.6.5 (2.6 branch)之前的内核版本中PACKET_MMAP buffer最大只能放下32768个frame(32 位)，在64bit 架构中最多放16384个frame。
首先是block size的限制。
每一个block是一个物理地址连续的区域，这些内存区域是调用通过__get_free_pages()函数分配的，这个函数式按照page来分配的，能分配的最大的page size是通过宏MAX_ORDER来限制的。精确的计算方式如下：
```
   PAGE_SIZE << MAX_ORDER

   In a i386 architecture PAGE_SIZE is 4096 bytes 
   In a 2.4/i386 kernel MAX_ORDER is 10
   In a 2.6/i386 kernel MAX_ORDER is 11
```
因此2.4/2.6内核下，32bit的系统通过__get_free_pages()最多可以分配4MB/8MB。
用户层的程序可以包含下面两个头文件来使用PAGE_SIZE MAX_ORDER
```
/usr/include/sys/user.h
/usr/include/linux/mmzone.h
```
###5. Packet mmap frame结构
这个结构来源于include/linux/if_packet.h
```
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
在packet_set_ring中会检查下面一些条件：

       tp_block_size 必须是PAGE_SIZE 的倍数
       tp_frame_size 必须比 TPACKET_HDRLEN 大
       tp_frame_size 必须是 TPACKET_ALIGNMENT 倍数
       tp_frame_nr   必须等于 frames_per_block * tp_block_nr

###6. 通过mmap系统调用来映射ring buffer到用户层
尽管整个ring buffer是由物理内存不连续的一些blocks组成的，但是映射完成后，在用户层程序看来虚拟地址是连续的。
mmap(0, size, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);

因为一个frame无法跨越两个block，所以如果 tp_block_size 不能被tp_frame_size 整除，那么每一个tp_block_size/tp_frame_size frames和下一个frame之间就会有一个gap。
如果想用一个socket来收包和发包，必须使用一次mmap系统调用来映射Rx和Tx ring
```
    ...
    setsockopt(fd, SOL_PACKET, PACKET_RX_RING, &foo, sizeof(foo));
    setsockopt(fd, SOL_PACKET, PACKET_TX_RING, &bar, sizeof(bar));
    ...
    rx_ring = mmap(0, size * 2, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
    tx_ring = rx_ring + size;
```
其中RX必须在第一个，TX紧跟其后。
###7. polling模式读取数据包
每一个frame的开始都有一个struct tpacket_hdr的结构体，里面有一个field是tp_status。
如果status == 0， 那么这个frame就可以为内核所用，否则就用户程序就可以读取这个frame了。
```
     #define TP_STATUS_KERNEL        0
     #define TP_STATUS_USER          1
```
开始时内核初始化所有的frame状态都是TP_STATUS_KERNEL，如果内核收到了一个包，并且将包copy到ring buffer后，就修改状态为TP_STATUS_USER，这样用户程序就可以读取并处理了，处理完了用户程序再设置frame的状态为TP_STATUS_KERNEL，供内核下次使用。
用户程序可以通过polling mode来查看ring中是否有新的数据包可用：
```
    struct pollfd pfd;

    pfd.fd = fd;
    pfd.revents = 0;
    pfd.events = POLLIN|POLLRDNORM|POLLERR;

    if (status == TP_STATUS_KERNEL)
        retval = poll(&pfd, 1, timeout);
```
### 8. polling模式发送数据包
```
#define TP_STATUS_AVAILABLE 0                      // Frame 可用
#define TP_STATUS_SEND_REQUEST 1             // Frame 即将发送
#define TP_STATUS_SENDING 2                        // Frame 正在发送
#define TP_STATUS_WRONG_FORMAT 4         // Frame 格式出错
```
内核初始化所有的frames为TP_STATUS_AVAILABLE ，用户层将要发送的数据填充完成后，设置tp_len为当前data buffer size，然后将status设置为TP_STATUS_SEND_REQUEST。这个可以一次设置好多包，一旦用户层程序准备好想发送了，调用send函数。
所有TP_STATUS_SEND_REQUEST 状态的数据包就会传给driver。传输过程中，内核将每一个发送的frame的状态设置为TP_STATUS_SENDING。每一个frame传输完成后，内核将frame状态改为TP_STATUS_AVAILABLE ，这样用户层程序又可以用了。
```
    header->tp_len = in_i_size;
    header->tp_status = TP_STATUS_SEND_REQUEST;
    retval = send(this->socket, NULL, 0, 0);
```
用户层可以使用 poll() 来检查frame是否可用：
```
(status == TP_STATUS_SENDING)

    struct pollfd pfd;
    pfd.fd = fd;
    pfd.revents = 0;
    pfd.events = POLLOUT;
    retval = poll(&pfd, 1, timeout);
```
OK，整个mmap的rx和tx流程基本完成。

