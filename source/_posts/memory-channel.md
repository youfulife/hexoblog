title: memory channel
date: 2014-12-14 10:21:00
tags: dpdk
categories: 编程
---
dpdk的示例程序中有个选项要求指定memory channel个数，之前没有怎么接触过这个东西，经过一番google后对这个概念有了基本的了解。


最关心的是linux有没有个命令能够直接输出这个信息，就和lscpu一样，输出cpu的基本信息，很不幸,， 去/sys和/proc下找了一圈发现没有，还按照网上说的安装了一个lshw的小工具，也没有直观的打印出memory channel的信息。
直接的既然没有，间接的肯定能得到，只能祭出dmidecode神器了。


通过man 8 dmidecode可以看到不同类型的硬件的号码，其中有一个是37  Memory Channel，运行命令dmidecode -t 37发现没有任何输出，空欢喜一场。本来以为是操作系统centos6.4比较老，内核版本比较低的原因，于是在最新的centOS 7上试了一把，还是没有信息。

当然既然dmidecode留了这个interface，后面肯定会通过这个选项直接直观的打印出来的。


下面是通过dmidecode的Memory信息来间接的查看当前的Channel信息，以及内存处于哪几个Channel上，是否对称。
<!-- more -->
查看channel个数信息：
``` bash
[root@localhost src]# dmidecode -t 17 | grep Bank
Bank Locator: NODE 0 CHANNEL 0 DIMM 0
Bank Locator: NODE 0 CHANNEL 0 DIMM 1
Bank Locator: NODE 0 CHANNEL 0 DIMM 2
Bank Locator: NODE 0 CHANNEL 1 DIMM 0
Bank Locator: NODE 0 CHANNEL 1 DIMM 1
Bank Locator: NODE 0 CHANNEL 1 DIMM 2
Bank Locator: NODE 0 CHANNEL 2 DIMM 0
Bank Locator: NODE 0 CHANNEL 2 DIMM 1
Bank Locator: NODE 0 CHANNEL 2 DIMM 2
Bank Locator: NODE 0 CHANNEL 3 DIMM 0
Bank Locator: NODE 0 CHANNEL 3 DIMM 1
Bank Locator: NODE 0 CHANNEL 3 DIMM 2
Bank Locator: NODE 1 CHANNEL 0 DIMM 0
Bank Locator: NODE 1 CHANNEL 0 DIMM 1
Bank Locator: NODE 1 CHANNEL 0 DIMM 2
Bank Locator: NODE 1 CHANNEL 1 DIMM 0
Bank Locator: NODE 1 CHANNEL 1 DIMM 1
Bank Locator: NODE 1 CHANNEL 1 DIMM 2
Bank Locator: NODE 1 CHANNEL 2 DIMM 0
Bank Locator: NODE 1 CHANNEL 2 DIMM 1
Bank Locator: NODE 1 CHANNEL 2 DIMM 2
Bank Locator: NODE 1 CHANNEL 3 DIMM 0
Bank Locator: NODE 1 CHANNEL 3 DIMM 1
Bank Locator: NODE 1 CHANNEL 3 DIMM 2
```
查看channel上插的内存条信息
``` bash
[root@localhost src]# dmidecode -t 17 | grep Size
Size: 4096 MB
Size: No Module Installed
Size: No Module Installed
Size: 4096 MB
Size: No Module Installed
Size: No Module Installed
Size: 4096 MB
Size: No Module Installed
Size: No Module Installed
Size: 4096 MB
Size: No Module Installed
Size: No Module Installed
Size: 4096 MB
Size: No Module Installed
Size: No Module Installed
Size: 4096 MB
Size: No Module Installed
Size: No Module Installed
Size: 4096 MB
Size: No Module Installed
Size: No Module Installed
Size: 4096 MB
Size: No Module Installed
Size: No Module Installed
```
这两个信息结合起来就可以看到当前的内存channel信息了。
结合上面两个就可以看出cpu node0上插了4个4G的内存，DIMM0，Channel 0,1,2,3。
node1上也是一样。
