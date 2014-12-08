title: libpcap install
date: 2014-12-08 17:56:37
tags: libpcap
categories: linux
---

CentOS 6.4 默认自带的是libpcap-1.4.0，虽然截止到目前主要是一些编译和系统支持的升级，但是测试性能还是安装最新版本libpcap较为稳妥
http://www.tcpdump.org/release/libpcap-1.5.3.tar.gz
解压，编译，安装
``` bash
tar -xzvf libpcap-1.5.3.tar.gz
cd libpcap-1.5.3
./configure
make
sudo make install
```
安装完成后动态库和静态库默认安装在/usr/local/lib/目录下，如下：
``` bash
Apr 21 14:00 libpcap.a
Apr 21 14:00 libpcap.so -> libpcap.so.1
Apr 21 14:00 libpcap.so.1 -> libpcap.so.1.5.3
Apr 21 14:00 libpcap.so.1.5.3
```
头文件安装在/usr/local/include/下：
``` bash
Apr 21 14:00 /usr/local/include/pcap-bpf.h
Apr 21 14:00 /usr/local/include/pcap.h
Apr 21 14:00 /usr/local/include/pcap-namedb.h
```
/usr/local/include/pcap目录下:
``` bash
Apr 21 14:00 bluetooth.h
Apr 21 14:00 bpf.h
Apr 21 14:00 ipnet.h
Apr 21 14:00 namedb.h
Apr 21 14:00 pcap.h
Apr 21 14:00 sll.h
Apr 21 14:00 usb.h
Apr 21 14:00 vlan.h
```
平时只需要用到/usr/local/include/pcap.h头文件就够了。
为了系统能够找到新安装的libpcap库，需要在系统的目录中添加一行配置：
vim打开/etc/ld.so.conf，在最下面添加下面一行/usr/local/lib，保存退出。
运行命令
``` bash
sudo ldconfig
```
更新缓存。

OK，现在已经可以用新安装的libpcap来进行编程了。
珍爱生命，为了不每次gcc的时候通过-I和-L指定头文件和动态库的目录，编写一个Makefile，以后修改完源代码后直接make就可以了，省时省事，童叟无欺。
Makefile源代码如下：
``` bash
LIBS = -lpcap
CFLAGS = -I/usr/local/include
LDFLAGS = -L/usr/local/lib
all: pcap_demo
%.o: %.c
gcc -c $(CFLAGS) $<
pcap_demo: pcap_demo.o
gcc -o $@ $< $(LDFLAGS) $(LIBS)
clean:
rm -f *.o pcap_demo
```
下一步就是编写pcap_demo.c来进行libpcap的性能测试了。