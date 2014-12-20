title: linux Packet socket (2) 套接字选项
date: 2014-12-20 14:27:39
tags: libpcap
categories: libpcap
---
linux 提供了两个函数来获取和设置套接字的一些选项：
```
       #include <sys/types.h>          
       #include <sys/socket.h>

       int getsockopt(int sockfd, int level, int optname, void *optval, socklen_t *optlen);
       int setsockopt(int sockfd, int level, int optname, const void *optval, socklen_t optlen);
```
通过指定不同的level和optname来设置不同层次的具体选项的值。比如要设置socket API这个层次的一些选项，level就指定为SOL_SOCKET，如果要设置Packet socket这个层次的一些选项，level就指定为SOL_PACKET。
<!-- more -->
从这里也可以看出，linux网络相关的一切都是分层次的，等级森严啊。
下面重点关注Packet socket这个level的一些重要选项，有些选项跟收包性能有很大的关系，尤其是最新的3.x内核，提供了很多多核方面的支持，但是libpcap好像还没有跟上多核的脚步，要想在最新的内核下使用socket来获得较高的IO性能，原始的libpcap就显得有点臃肿和力不从心了。
多核下可以尝试使用Go语言通过raw socket编程来写一个高效的多核并行数据包处理程序。
扯远了，再远就收不回来了。
Level SOL_PACKET下有下面一些主要的选项：
```
       PACKET_ADD_MEMBERSHIP
       PACKET_DROP_MEMBERSHIP
``` 
设置网卡多播和混杂模式，optval是下面struct packet_mreq类型:
```            
                  struct packet_mreq {
                      int            mr_ifindex;    /* interface index */
                      unsigned short mr_type;       /* action */
                      unsigned short mr_alen;       /* address length */
                      unsigned char  mr_address[8]; /* physical layer address */
                  };
```
mr_ifindex 要设置的网卡索引
mr_type 指定设置具体类型

PACKET_MR_PROMISC 混杂模式
PACKET_MR_MULTICAST 接收mr_address组的数据包
PACKET_MR_ALLMULTI 接收所有的组的包
PACKET_AUXDATA (since Linux 2.6.21)

设置recvmsg()接收的每个包都有一个metadata的结构体，这个结构体存储了当前包的一些信息，可以通过 cmsg(3)来读取，定义如下：
```
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
PACKET_FANOUT (since Linux 3.1)
这个选项是linux 3.1新加的，这个时候多核设备已经铺天盖地了。linux内核也得跟上时代的脚步吧，于是增加了多线程处理功能的选项。再配合上多核的cpu affinity， 就可以做load balancing了。跟网卡提供的RSS机制有点类似。在这种模式下，可以同时开启多个Packet socket，组成一个fanout group，一个数据包只会根据规则匹配送给组内的其中一个socket。

PACKET_FANOUT 选项就可以将一个socket添加到特定的fanout group中。
每一个网卡最多可以有65536个独立的fanout group。整数类型的选项值的低十六位表示group id。一般情况下都使用进程特有的id，比如当前进程的id来作为group_id。第一个加入到某个group的socket会默认创建一个组。也就是小组长。这个socket的设置决定了后来的socket能否成功的加入这个group。为了成功的加入到一个已经存在group中，后续的想加入的sockets必须和小组长使用相同的协议，相同的硬件设置，相同的fanout模式和标志位。
这个里面就有点江湖气息了。进来容易出去难，如果想退出，只能关闭socket。如果这个组里所有的socket都关闭了，这个组也就不存在了。

Fanout支持好多的模式。
**PACKET_FANOUT_HASH** hash模式，也是默认模式。对每一个数据包，根据ip地址和tcp端口号(可选) 算出一个hash，保证同一个IP flow或者tcp flow的包hash到相同的socket。
**PACKET_FANOUT_LB** load-balance 模式，从0开始循环选择socket。
**PACKET_FANOUT_CPU**根据cpu来选择，数据包从哪个cpu id上来就选择哪个。
**PACKET_FANOUT_ROLLOVER**积压模式，先把所有的数据包给一个socket，等这个socket处理不过来的时候就换下一个socket，这个比较粗暴啊。
**PACKET_FANOUT_RND** 随机模式，通过一个伪随机数生成器随机找一个socket
**PACKET_FANOUT_QM**(Linux 3.14以后可用，这个没弄明白) selects the socket using the recorded queue_mapping of the received  skb.

Fanout 模式可以增加额外的选项。比如ip分片，分片的包里是没有tcp port信息的，如果指定按照四元组来hash，那么可能同一条流会hash到不同的socket。
**PACKET_FANOUT_FLAG_DEFRAG** 在进行fanout之前会进行分片重组。
**PACKET_FANOUT_FLAG_ROLLOVER** 设置ROLLOVER模式为备份策略。如果原来的模式，某一个socket已经处理不过来了，那么设置了这个后，后面的数据包会自动选择下一个空闲的socket。Fanout的模式和额外的选项是在选项值的高16位设置的。
```
              fanout_arg = (fanout_id | (fanout_type << 16));
```
       PACKET_LOSS (with PACKET_TX_RING)
              通过mmap发送数据包时，如果遇到格式错误的数据包, 自动跳过，
              自动设置tp_status为TP_STATUS_AVAILABLE，
              保证后续ring中的数据包能够及时自动的发送。
              如果不设置这个选项，那么一旦出现格式错误的数据包，
              默认 tp_status 设置为 TP_STATUS_WRONG_FORMAT
              中断发送，并且阻塞住当前的发送线程。 要想恢复后续数据的发送，
              必须将tp_status 重新设置为 TP_STATUS_SEND_REQUEST, 然后重新
              调用send()函数发送。
              

       PACKET_RESERVE (with PACKET_RX_RING)
              在数据包的头部预留出一定长度的空间，
              一般在接收vlan数据包的时候，在2层的头部预留出vlan tag空间大小
              

       PACKET_RX_RING
              创建一个mmap的ring buffer来提供数据包异步接收功能。
              这个ring buffer是一个用户空间地址连续的数组，
              每个数组元素长度为tp_snaplen，类似于char ring[slot_num][tp_snaplen]              
              每个数据包的前面都有一个描述当前数据包的metadata，
              定义和 tpacket_auxdata 相同，但是有些域的意义变了。 
              原来标识号协议的那几个域现在存储着数据到metadata头部的偏移量。
              tp_net 存储网络层的偏移量，
              如果socket 类型是 SOCK_DGRAM, 那么 tp_mac 也是到网络层的偏移，
              因为SOCK_DGRAM 收上来的包没有链路层的头部。如果是 SOCK_RAW,
              tp_mac存的是链路层的偏移。
              tp_status 用来表明ring slot的状态，也就是当前的slot是否可用。
              初始时，所有slot的 tp_status 等于 TP_STATUS_KERNEL. 
              socket收到一个数据包并且copy到ring slot后，就会把当前这个slot
              的所有权交给用户程序，并且把tp_status设置为 TP_STATUS_USER ，
              如果应用程序处理完了当前的数据包，那么就把 tp_status 设置为
              TP_STATUS_KERNEL. 这样下一次socket又可以继续使用这个slot了。
              说白了就是一个标志位flag。Packet
              具体的的实现可以参考内核源代码中提供的文档
              Documentation/networking/packet_mmap.txt
              

       PACKET_STATISTICS
              获得Packet socket内部的一些统计数字，比如总共包数和丢包数

                  struct tpacket_stats {
                      unsigned int tp_packets;  /* Total packet count */
                      unsigned int tp_drops;    /* Dropped packet count */
                  };

              调用一次接收后会重置计数器。
              TPACKET_V3 版本的统计数字的结构体和上面的V1, V2版本的不一样。

       PACKET_TIMESTAMP (with PACKET_RX_RING; since Linux 2.6.36)
              设置rx ring中metadata 头部中的时间戳。
              默认是在数据包copy的时候，获取一个软件时间戳。
              除了默认之外，还支持下面内核文档中描述的两种硬件时间戳
              Documentation/networking/timestamping.txt 
              

       PACKET_TX_RING (since Linux 2.6.31)
              创建tx mmap ring。
              参数选项和 PACKET_RX_RING 相同，
              初始时，tp_status 等于 TP_STATUS_AVAILABLE 
              用户程序将要发送的数据包写到ring slot中后，
              修改 tp_status 为 TP_STATUS_SEND_REQUEST.
              然后调用send或者类似的发送函数进行数据包的发送，
              成功发送完成后tp_status 重新设置为 TP_STATUS_AVAILABLE。
              除非设置了 PACKET_LOSS 否则一旦出现错误就会直接中断当前和后续发送。

       PACKET_VERSION (with PACKET_RX_RING; since Linux 2.6.27)
              默认情况下，PACKET_RX_RING 创建一个 TPACKET_V1 类型ring buffer。
              通过设置这个选项，可以创建一个其他类型的ring buffer。
              比如TPACKET_V2 和 TPACKET_V3。

       PACKET_QDISC_BYPASS (since Linux 3.14)
              默认情况下，Packet socket的数据包会通过内核的qdisc（流量控制）层。

              但是像发包器这样的程序有时候希望测试设备的最大性能。比如pktgen。
              通过设置这个选项为1，就可以略过qdisc层。
              这个设置还有一个副作用，就是原本qdisc 层有一个packet的buffer，
              跳过qdisc，这个缓冲区就无法使用，如果传输队列比较忙，可能会丢包。
              是否设置跳过，自己承担相应的风险吧。
