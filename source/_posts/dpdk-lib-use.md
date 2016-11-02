title: dpdk 移植到开源软件
date: 2015-01-26 14:54:53
tags: dpdk
categories: 编程
---
dpdk 自己提供了一套Makefile的写法，当需要使用dpdk来开发自己的应用程序或库时，可以按照dpdk makefile的固定套路来编写自己的Makefile。
官方文档描述的非常清晰。
http://www.dpdk.org/doc/guides/prog_guide/build_app.html

有的时候不是从头来写，而是要把dpdk移植到一个本身也有着复杂Makefile的开源软件中，这样就会产生一个问题：到底是服从哪一边的书写规范？
现在给出一个移植dpdk到开源软件的一些套路：

dpdk 编译后会生成一系列的lib库供外部程序使用，这个里面有一些坑需要注意。dpdk里面网卡的模块用到了gcc 构造函数 扩展。使得各种各样的Intel 网卡可以在main函数运行之前就注册到Tail queue中。
这样在初始化函数中直接就可以通过遍历列表来对网卡相关的数据结构进行操作。
<!-- more -->
但是gcc在链接时检查如果没有明确引用某个.o文件中的符号，那么默认是不会链接这个.o文件的。这样就导致虽然指定了静态库，但是通过nm命令在最终的可执行文件中依然找不到相关的符号。

http://stackoverflow.com/questions/1202494/why-doesnt-attribute-constructor-work-in-a-static-library

The linker only loads the modules it must (i.e. what you actually reference implicitly or explicitly), unless you force it.
That is both a good thing, and unfortunate. It makes the link process faster and prevents code bloat. It normally does not cause problems either, because normally you reference all modules that are needed in some way, and what you don’t reference isn’t needed. Normally.
But it is also the cause for this silent failure: The linker never loads the module containing your constructor/destructor functions and never even looks at it. That is because you never actually call these functions.
You can force the linker to include the module by explicitly linking with the object file that corresponds to the source containing the constructor/destructor functions.
In other words, if the constructor/destructor functions are in foo.c, add -l/path/to/foo.o to your linker’s options, or simply pass foo.o on the commandline to the linker (as an extra argument).
In either case, by doing so, you explicitly tell the linker to consider this module, and doing so will make it find the constructor functions and properly call them.
An alternative (which does not require you to play with the commandline) may be putting a variable or a function (which does not necessarily do anything) into the same source file as your constructor/destructor functions and calling that one from any source file in the main program.
This will also force the linker to load the containing module (where it will find the constructor functions).

解决方案：
http://industriousone.com/topic/how-use-whole-archive-gcc

##**开源软件Makefile修改步骤：**##

将dpdk编译生成的静态库全部包含进来
```
ifeq ($(strip $(DPDK_ENABLE)), yes)
    LIBDPDK =  $(ROOT_DIR)/dpdk/build/lib
    LIBS += $(LIBDPDK)/libethdev.a
    LIBS += $(LIBDPDK)/librte_mbuf.a
    LIBS += $(LIBDPDK)/librte_pmd_af_packet.a
    LIBS += $(LIBDPDK)/librte_pmd_i40e.a
    LIBS += $(LIBDPDK)/librte_pmd_vmxnet3_uio.a
    LIBS += $(LIBDPDK)/librte_sched.a
    LIBS += $(LIBDPDK)/librte_eal.a
    LIBS += $(LIBDPDK)/librte_kvargs.a
    LIBS += $(LIBDPDK)/librte_mempool.a
    LIBS += $(LIBDPDK)/librte_pmd_bond.a
    LIBS += $(LIBDPDK)/librte_pmd_ixgbe.a
    LIBS += $(LIBDPDK)/librte_port.a
    LIBS += $(LIBDPDK)/librte_table.a
    LIBS += $(LIBDPDK)/librte_cfgfile.a
    LIBS += $(LIBDPDK)/librte_hash.a
    LIBS += $(LIBDPDK)/librte_meter.a
    LIBS += $(LIBDPDK)/librte_pmd_e1000.a
    LIBS += $(LIBDPDK)/librte_pmd_ring.a
    LIBS += $(LIBDPDK)/librte_power.a
    LIBS += $(LIBDPDK)/librte_timer.a
    LIBS += $(LIBDPDK)/librte_malloc.a
    LIBS += $(LIBDPDK)/librte_pmd_enic.a
    LIBS += $(LIBDPDK)/librte_pmd_virtio_uio.a
    LIBS += $(LIBDPDK)/librte_ring.a
    LIBS += $(LIBDPDK)/librte_lpm.a
    LIBS += $(LIBDPDK)/librte_ip_frag.a
    LIBS += $(LIBDPDK)/librte_cmdline.a
    LIBS += $(LIBDPDK)/librte_acl.a

else
endif
```
生成可执行文件：

```
exec: exec.o
    $(CC) -o $@ $< $(LDFLAGS) -lrt -Wl,--whole-archive $(LIBS) -Wl,--no-whole-archive

```
-Wl,—whole-archive $(LIBS) -Wl,—no-whole-archive 选项告诉gcc，把$(LIBS)中的所有符号(不管有没有显式调用)通通链接到可执行文件中。
当然，这样会使得最终生成的可执行文件变大。

##编译过程中遇到的问题：##
###1. 编译报错 RTE_PKTMBUF_HEADROOM 等没有定义：###

```
../../../dpdk/build/include/rte_mbuf.h:648: error: ‘RTE_PKTMBUF_HEADROOM’ undeclared (first use in this function)
../../../dpdk/build/include/rte_mbuf.h:648: error: (Each undeclared identifier is reported only once
../../../dpdk/build/include/rte_mbuf.h:648: error: for each function it appears in.)
In file included from ../../src/rx_dpdk.c:68:
../../../dpdk/build/include/rte_ethdev.h: At top level:
../../../dpdk/build/include/rte_ethdev.h:203: error: ‘RTE_ETHDEV_QUEUE_STAT_CNTRS’ undeclared here (not in a function)
../../src/rx_dpdk.c:93: error: ‘RTE_MAX_ETHPORTS’ undeclared here (not in a function)
../../src/rx_dpdk.c: In function ‘l2fwd_main_loop’:
../../src/rx_dpdk.c:237: error: ‘RTE_LOG_LEVEL’ undeclared (first use in this function)
../../src/rx_dpdk.c: In function ‘rx_dpdk’:
../../src/rx_dpdk.c:518: error: ‘RTE_PKTMBUF_HEADROOM’ undeclared (first use in this function)
```

没有在源文件中#include ，example下的c文件中(比如l2fwd)也没有include这个头文件，但是能够编译通过。
是因为在mk中通过include指令包含了，如下所示。
```
./target/generic/rte.vars.mk:CFLAGS += -include $(RTE_SDK_BIN)/include/rte_config.h

```
我们没有按照dpdk的makefile格式来写, 自然需要自行在源文件中添加，或者在自己的Makefile中也用include指令。

##2. 编译时 srand48 和lrand48 函数未定义：##

```
In file included from ../../src/rx_dpdk.c:66:
dpdk/dpdk-1.8.0/x86_64-native-linuxapp-gcc/include/rte_random.h: In function ‘rte_srand’:
dpdk/dpdk-1.8.0/x86_64-native-linuxapp-gcc/include/rte_random.h:63: warning: implicit declaration of function ‘srand48’
dpdk/dpdk-1.8.0/x86_64-native-linuxapp-gcc/include/rte_random.h: In function ‘rte_rand’:
dpdk/dpdk-1.8.0/x86_64-native-linuxapp-gcc/include/rte_random.h:80: warning: implicit declaration of function ‘lrand48’
```
解决方案：
‘srand48’函数是在stdlib.h中定义的，而且搜索了一下发现dpdk包含了stdlib.h这个头文件，而且自己在程序中最开始也include了这个头文件。
甚至手动加上这两个宏定义 USE_SVID || defined USE_XOPEN还是不行。在源文件开头(include 语句前面)加上这两行：
```
extern void srand48 (long int __seedval);
extern long int lrand48 (void);
```
或者直接在Makefile中指定
```
CFLAGS     += -include /usr/include/stdlib.h
```
搜索了一下，发现系统中有好多的stdlib.h，只有/usr/include/stdlib.h下面才有srand48和lrand48的定义。

##3. 链接时函数引用错误:

```
/../../dpdk/dpdk-1.8.0/x86_64-native-linuxapp-gcc/lib/librte_ring.a
../../../dpdk/dpdk-1.8.0/x86_64-native-linuxapp-gcc/lib/librte_eal.a(eal_timer.o): In function `rte_eal_timer_init':
eal_timer.c:(.text+0x19c): undefined reference to `clock_gettime'
eal_timer.c:(.text+0x1d6): undefined reference to `clock_gettime'
collect2: ld returned 1 exit status
```
在链接时添加-lrt 库

```
$(CC) -o $@ $< $(LDFLAGS) -lrt -l...
```
