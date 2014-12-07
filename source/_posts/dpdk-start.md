title: dpdk start
date: 2014-12-07 22:20:37
tags: driver
categories: dpdk
---
dpdk版本：dpdk-1.7.0
下载地址：http://www.dpdk.org/browse/dpdk/snapshot/dpdk-1.7.0.tar.gz
操作系统：Centos 6.4
内核版本：2.6.32 x86_64
编译过程：tar -xzvf  dpdk-1.7.0.tar.gz

<!-- more -->

``` bash
cd  dpdk-1.7.0
make install T=x86_64-native-linuxapp-gcc
```
加载内核模块：
``` bash
modprobe uio
insmod x86_64-native-linuxapp-gcc/kmod/igb_uio.ko
```

绑定网卡：
```bash
python tools/igb_uio_bind.py --bind=igb_uio [device0] [device1]...
```
首先通过脚本看看本机有哪些可用的网卡，包括网卡当前的状态：
``` bash
python tools/igb_uio_bind.py --status
```
输出如下：
``` bash
Network devices using IGB_UIO driver
====================================
<none>

Network devices using kernel driver
===================================
0000:02:00.0 'I350 Gigabit Network Connection' if=eth0 drv=igb unused=igb_uio *Active*

Other network devices
=====================
0000:83:00.0 '82599EB 10-Gigabit SFI/SFP+ Network Connection' unused=ixgbe,igb_uio
0000:83:00.1 '82599EB 10-Gigabit SFI/SFP+ Network Connection' unused=ixgbe,igb_uio
```
从上面的输出可以可以看到有两个intel 10G网口并没有绑定到任何驱动。
下面通过命令把0000:83:00.0绑定到igb_uio模块：
``` bash
python tools/igb_uio_bind.py --bind=igb_uio 0000:83:00.0
```
绑定完成后再通过status参数来查看状态发现0000:83:00.0网口已经绑定到igb_uio上了。
``` bash
Network devices using IGB_UIO driver
====================================
0000:83:00.0 '82599EB 10-Gigabit SFI/SFP+ Network Connection' drv=igb_uio unused=ixgbe

Network devices using kernel driver
===================================
0000:02:00.0 'I350 Gigabit Network Connection' if=eth0 drv=igb unused=igb_uio *Active*

Other network devices
=====================
0000:83:00.1 '82599EB 10-Gigabit SFI/SFP+ Network Connection' unused=ixgbe,igb_uio
```
到目前为止，dpdk的内核模块已经安装完成，并且绑定了特定的网卡。
设置环境变量：
``` bash
 export RTE_SDK=/root/dpdk-1.7.0
 export RTE_TARGET=x86_64-native-linuxapp-gcc
```
使用 hugepage：
``` bash
mkdir -p /mnt/huge
mount -t hugetlbfs nodev /mnt/huge
echo 64 > /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages
echo 64 > /sys/devices/system/node/node1/hugepages/hugepages-2048kB/nr_hugepages
```
在cpu node0和node1上分别预留出64个2M大小的pages供dpdk的程序使用。
运行helloworld:
``` bash
cd examples/helloworld/
make
./build/helloworld -c f -n 4 --socket-mem=64
```
可以看到打印出下面信息：
``` bash
...
EAL: Requesting 32 pages of size 2MB from socket 0
EAL: TSC frequency is ~1995194 KHz
EAL: Master core 0 is ready (tid=56c31800)
EAL: Core 3 is ready (tid=4fffa700)
EAL: Core 2 is ready (tid=509fb700)
EAL: Core 1 is ready (tid=513fc700)
hello from core 1
hello from core 2
hello from core 3
hello from core 0
```
参数说明：
-c f: 指定运行的core mask，16进制，0xf = 0b1111 的意思是运行在core 0,1,2,3
-c 和 -n参数是必须的
–socket-mem=64 指定仅仅分配64M的内存在socket 0上。
如果不指定–socket-mem 默认就是分配预留的全部page。
如果想同时在socket0和socket1上分配内存，可以中间用逗号隔开，但是两个数字中间不能有空格，如果指定0表示当前的socket上不分配内存。
比如：
想在socket0上分配32M，想在socket1上分配16M内存，每个pagesize是2M，可以通过
sock-mem来指定。
``` bash
./build/helloworld -c f -n 4 --socket-mem=32,64
```
输出如下:
``` bash
...
EAL: Requesting 16 pages of size 2MB from socket 0
EAL: Requesting 32 pages of size 2MB from socket 1
...
```