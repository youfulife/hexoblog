title: ixgbe tcp stream RSS
date: 2014-12-07 15:02:54
categories: driver
tags: [linux, driver]
---

目前的网卡都提供了multi queue的功能， 为了提高I/O能力， 在多核cpu上可以打开RSS功能，使同一条流的数据包hash到同一个rx queue中， 将每一个rx queue绑定到一个cpu core上， 这样就可以多个cpu core并行的处理数据包， 使得处理能力大大的提高。
<!-- more -->
但是根据Intel 82599手册提供的算法， 发现目前的算法只能将同一个tcp会话中同一个方向的数据包hash到同一个rx queue中， 而另一个方向的数据包会hash到不同的rx queue中， 这对后面应用层的分析会造成很大的cache trashing。
下面是Intel 82599手册提供的RSS 算法：
```
function ComputeRSSHash(Input[], RSK)
  ret = 0;
  for each bit b in Input[] do
    if b == 1 then
        ret ˆ= (left-most 32 bits of RSK);
    end if
    shift RSK left 1 bit position;
  end for
end function
```
为了提高性能， Intel将这个算法通过硬件实现， 因此想要通过修改算法而得到一个对称的RSS几乎是不可能的， 但是RSK是在驱动中可以控制的， 在driver第一次load的时候可以指定，所以可以通过更新RSK来得到一个对称的hash结果。
下面是韩国人发表的一篇paper中提到的算法
http://www.ndsl.kaist.edu/~shinae/papers/TR-symRSS.pdf
根据文章中提供的算法， 在函数ixgbe_configure_rx()中， 做如下修改：
``` c
#if 0
static const u32 seed[10] = { 0xE291D73D, 0x1805EC6C, 0x2A94B30D, 0xA54F2BEC, 0xEA49AF7C, 0xE214AD3D, 0xB855AABE, 0x6A3E67EA, 0x14364D17, 0x3BED200D};
#endif
static const u32 seed[10] = { 0x6d5a6d5a, 0x6d5a6d5a, 0x6d5a6d5a, 0x6d5a6d5a, 0x6d5a6d5a, 0x6d5a6d5a, 0x6d5a6d5a, 0x6d5a6d5a, 0x6d5a6d5a, 0x6d5a6d5a};
```
还需要关闭掉rss type， 仅仅保留IPV4，否则分片的IP packet没法办啊
```c
/* Perform hash on these packet types */
mrqc |= IXGBE_MRQC_RSS_FIELD_IPV4; //we only use ipv4
#if 0
mrqc |= IXGBE_MRQC_RSS_FIELD_IPV4_TCP
| IXGBE_MRQC_RSS_FIELD_IPV4_UDP
| IXGBE_MRQC_RSS_FIELD_IPV6
| IXGBE_MRQC_RSS_FIELD_IPV6_TCP
| IXGBE_MRQC_RSS_FIELD_IPV6_UDP;
#endif
}
```
这样修改完成后， 经过测试发现
同一个tcp会话中同一个方向的数据包hash到同一个rx queue中， 然后在用户层每一个rx queue绑定到一个cpu core，测试发现基本上这样网卡会将数据包均匀分布到开启的rx队列上。
