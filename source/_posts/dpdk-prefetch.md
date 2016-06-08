title: dpdk prefetch 介绍
date: 2015-01-12 12:54:48
tags: [performance, dpdk]
categories: dpdk
---

prefetchXX指令的左右是提前将memory中的数据或指令放到Cache中，这样cpu下次用到这个数据的时候直接用就行了，不会出现cache miss。
CPU自己也会去根据局部性prefetch一些数据，但是效果并不能达到最佳，也称为硬件prefetch，不是程序员能够控制了的。
下面说的主要是能够自己控制的prefetch，也叫软件prefetch。

dpdk 在 rte_prefetch.h 中提供了一组prefetch指令的API，如下：

``` c
static void 	rte_prefetch0 (volatile void *p)
static void 	rte_prefetch1 (volatile void *p)
static void 	rte_prefetch2 (volatile void *p)
```
<!-- more -->
参数是一个内存地址，表示要prefetch到cache中数据的起始地址。每次prefetch一个 cache line 长度的数据，一般为64Bytes。
0-2 表示prefetch的局部性级别。prefetch 提高性能主要依据就是局部性原则。
prefetch0 级别最高，会将数据prefetch到L1，L2，L3 cache中。
prefetch1 级别稍微低一些，会将数据prefetch L2，L3 cache中。
prefetch2 级别再低一些，只将数据prefetch到L3 cache中。
具体详细解释可以看dpdk API帮助文档 http://www.dpdk.org/doc/api/rte__prefetch_8h.html

在现在的Intel多核架构下，每个core L1，L2是独占的，L3 cache在同一个物理cpu上是共享的。从L1-L3容量依次增大，速度也越来越慢。

局部性级别最低的是 prefetchnta, 意思就是没有局部性 (Non Temporal)。DPDK中并没有给出相关 API。
prefetchnta 会暗示cpu这个数据是一次性的，用完一次后续不会再用了。

上面几种prefetchXX的区别还有置换策略问题。
其中prefetch 0-2会遵循操作系统的cache替换策略，一般是LRU，所以会造成cache的污染。
prefetchnta 会将cache污染降低到最小。这个指令只将数据prefetch到某一层cache中的某一way中。不同的cpu型号可能不同。
![](http://7sbqk1.com1.z0.glb.clouddn.com/youfu_blog_prefetchnta.png)

prefetch 不是总能成功，这个指令只是对cpu的一个暗示(hint)，cpu会不会理睬要看心情。
所以就算失败，地址无效之类的也不会产生异常(exception)。
如果prefetch的时候发现数据已经在更高级的cache中了，那么prefetch指令不会做任何事情。

滥用prefetch 会影响性能，尤其是prefetch 0-2，如果替换出去的数据是有用的数据，进来的是没有用的数据，那么性能会变得很糟糕。而且会占用内存带宽。
保险起见，大部分情况下可以使用prefetchnta指令。

prefetch的步长问题比较难判定。需要结合具体应用场景具体分析，可以结合perf工具查看cache miss来选择一个合适的步长。

可以将prefetch和MOVNTQ指令结合使用。
MOVNTQ，使用WC(Write-combining)模式，跳过缓存直接写内存。不会对cache造成影响，那些写入内存长时间不用的数据就可以用这个指令来优化。比如抓包程序。

在linux下可以使用gcc提供的build_in Function。具体使用方法粘帖如下：
``` c
void __builtin_prefetch (const void *addr, ... ) [Built-in Function]
```
This function is used to minimize cache-miss latency by moving data into a cache
before it is accessed. You can insert calls to __builtin_prefetch into code for
which you know addresses of data in memory that is likely to be accessed soon. If
the target supports them, data prefetch instructions are generated. If the prefetch is
done early enough before the access then the data will be in the cache by the time it
is accessed.
The value of addr is the address of the memory to prefetch. There are two optional
arguments, rw and locality. The value of rw is a compile-time constant one or zero;
one means that the prefetch is preparing for a write to the memory address and zero,
the default, means that the prefetch is preparing for a read. The value locality must
be a compile-time constant integer between zero and three. A value of zero means
that the data has no temporal locality, so it need not be left in the cache after the
access. A value of three means that the data has a high degree of temporal locality and
should be left in all levels of cache possible. Values of one and two mean, respectively,
a low or moderate degree of temporal locality. The default is three.

``` c
for (i = 0; i < n; i++)
{
    a[i] = a[i] + b[i];
    __builtin_prefetch (&a[i+j], 1, 1);
    __builtin_prefetch (&b[i+j], 0, 1);
/* . . . */
}
```
Data prefetch does not generate faults if addr is invalid, but the address expression
itself must be valid. For example, a prefetch of p->next does not fault if p->next is
not a valid address, but evaluation faults if p is not a valid address.
Chapter 6: Extensions to the C Language Family 491
If the target does notsupport data prefetch, the address expression is evaluated if it
includes side effcts but no other code is generated and GCC does not issue a warning.
