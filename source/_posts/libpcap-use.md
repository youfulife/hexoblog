title: libpcap使用
date: 2014-12-09 09:23:54
tags: libpcap
categories: libpcap
---

查看pcap.h文件， 可以看到libpcap给出的对外接口。
平时使用libpcap捉包常用的数据结构和API如下：
``` c
char    *pcap_lookupdev(char *);
```
这个函数得到当前可以使用的第一个网卡接口，返回的网卡接口可以作为pcap_open_live的参数。
<!-- more -->
``` c
char errbuf[PCAP_ERRBUF_SIZE];
pcap_t *pcap_open_live(const char *device, int snaplen, int promisc, int to_ms, char *errbuf);
```
这个函数打开一个网络设备，返回一个pcap_t类型的描述符，如果返回值不是NULL，那么就可以在这个设备上开始抓包了。相关的数据结构的初始化已经在这个函数里搞好了。
``` c
pcap_t *pcap_open_offline(const char *, char *);
```
这个函数主要是用来读取pcap文件的，如果你的数据包是从文件中读取，那么用这个函数。
``` c
struct pcap_pkthdr {
    struct timeval ts; /* time stamp */
    bpf_u_int32 caplen; /* length of portion present */
    bpf_u_int32 len; /* length this packet (off wire) */
};
int pcap_loop(pcap_t *p, int cnt, pcap_handler callback, u_char *user);
typedef void (*pcap_handler)(u_char *user, const struct pcap_pkthdr *h, const u_char *data);
```
通过上面这三个自古以来就是不可分割的整体。
pcap_pkthdr表示的是当前包的信息，其中主要是caplen和len可能会让人迷糊，caplen表示的是当前包的data的长度，len表示包的原来长度，大多数情况下这两个长度的值是一样的，但是，当来了一个超长包，比如10000bytes，而我们在pcap_open_live中设定的snaplen只有1000, 那么caplen就是1000, len就是10000, 剩下的9000就被libpcap给截断了。
这种情况下需要注意的是后续包的处理中如果使用ip头部的iplen做为偏移量来移动指针，那么可能会越界，需要做严格的长度检查。


pcap_loop作为永恒的主角。
参数cnt表示捕捉cnt个包后退出，大部分情况都是传入一个-1,表示永远不会退出，根本停不下来。callback表示用户层的处理函数，数据包捕捉住后libpcap就将数据包交给pcap_handler了。
user传入用户自定义或者一些私有数据的指针，这个指针也会原封不动的传递给pcap_handler的第一个参数。
pcap_handler中完全就是用户说了算了，数据包的生命都交给你了，这个函数返回的时候也就是当前数据包终结的时候。
第三个参数data指向真正数据包的内容，从二层的MAC地址开始。
``` c
struct pcap_stat {
    u_int ps_recv; /* number of packets received */
    u_int ps_drop; /* number of packets dropped */
    u_int ps_ifdrop; /* drops by interface — only supported on some platforms */
#ifdef WIN32
    u_int bs_capt; /* number of packets that reach the application */
#endif /* WIN32 */
};
int pcap_stats(pcap_t *, struct pcap_stat *);
```
struct pcap_stat和pcap_stats()之前没有用过，再查看pcap.h的时候发现的，这个数据结构可以用来统计数据包的个数，尤其是网卡丢包的个数。
pcap_stats函数需要传入一个pcap_t类型的指针，可以通过pcap_loop的user参数传递给pcap_handler函数，然后在pcap_handler中调用pcap_stats函数，得到收到的包个数和丢包个数。
``` c
void pcap_close(pcap_t *);
```
关闭打开的pcap_t描述符，释放各种资源。
下面给出简单的demo源代码：
``` c
#include <pcap.h>
struct pcap_stat pkt_stat;
void yf_pcap_handler(u_char * user, struct pcap_pkthdr *hdr, u_char *data)
{
    //printf(“pkt len is %d\n”, hdr->caplen);
    //packet process code here
    pcap_stats((pcap_t *)user, &pkt_stat);
    printf(“pkt recv %d\n”, pkt_stat.ps_recv);
    printf(“pkt drop %d\n”, pkt_stat.ps_drop);
    printf(“pkt if drop %d\n”, pkt_stat.ps_ifdrop);
    return;
}
int main(int argc, char *argv[])
{
    pcap_t *pcap_desc;
    char errbuf[PCAP_ERRBUF_SIZE];
    int promisc = 1;
    int pcap_timeout = 1024; //milliseconds
    int snaplen = 65535;
    char *device = NULL;
    printf(“libpcap version is: %s\n”, pcap_lib_version());
    device = pcap_lookupdev(errbuf);
    if (device != NULL) {
        printf(“find a device: %s\n”, device);
    } else {
        printf(“ERR: %s”, errbuf);
    }
    pcap_desc = pcap_open_live(device, snaplen, promisc, pcap_timeout, errbuf);
    if(pcap_desc == NULL){
        printf(“%s\n”, errbuf);
        return;
}
    pcap_loop(pcap_desc, -1, (pcap_handler) yf_pcap_handler, (u_char *)pcap_desc);
    pcap_close(pcap_desc);
    return;
}
```