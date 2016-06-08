title: centOS6.4搭建xv6学习环境
date: 2014-12-10 13:48:00
tags: xv6
categories: 操作系统
---
前两天在centos 6.4服务器上折腾了一会儿xv6，xv6是MIT教学用的操作系统内核，总共6000多行代码，操作系统该有的基本都有了。
跟着MIT给出的课程表一步一步来，折腾完后就可以看linux内核了。
课程表：http://pdos.csail.mit.edu/6.828/2014/schedule.html
xv6主页：http://pdos.csail.mit.edu/6.828/2014/xv6.html
相关tools：http://pdos.csail.mit.edu/6.828/2014/tools.html
说白了，现在就是在还大学欠下的债。
首先要安装qemu，能让xv6在上面运行起来。
<!-- more -->
``` bash
git://git.qemu-project.org/qemu.git
sudo yum install glibc-devel.i686
cd qemu
./configure --disable-kvm  --target-list="i386-softmmu x86_64-softmmu"
make
sudo make install
```
OK，qemu目前已经安装到系统目录了，通过命令看看安装了哪些东西:
```
[youfu@localhost qemu]$ ls /usr/local/bin/qemu*

/usr/local/bin/qemu      /usr/local/bin/qemu-io     
/usr/local/bin/qemu-system-x86_64
/usr/local/bin/qemu-ga   /usr/local/bin/qemu-nbd
/usr/local/bin/qemu-img  /usr/local/bin/qemu-system-i386
```

下载xv6代码，编译，运行：
```
git clone git://pdos.csail.mit.edu/xv6/xv6.git
cd xv6
make
qemu-system-x86_64 -smp 1 -parallel stdio -hdb fs.img xv6.img -m 512 -curses
```
可以看到xv6已经在上面跑起来了。
添加一个符号连接：
```
sudo ln -s /usr/local/bin/qemu-system-x86_64 /usr/local/bin/qemu
```

Lab1：http://pdos.csail.mit.edu/6.828/2014/labs/lab1/
Lab1中还提供了一个内核的初始化模板，叫JOS，目前只是运行起来了，但是还没有去深入的了解。
```
git clone http://pdos.csail.mit.edu/6.828/2014-jos.git
make
make qemu-nox
```
内核启动之后会看到下面信息：
```
Booting from Hard Disk...
6828 decimal is XXX octal!
entering test_backtrace 5
entering test_backtrace 4
entering test_backtrace 3
entering test_backtrace 2
entering test_backtrace 1
entering test_backtrace 0
leaving test_backtrace 0
leaving test_backtrace 1
leaving test_backtrace 2
leaving test_backtrace 3
leaving test_backtrace 4
leaving test_backtrace 5
Welcome to the JOS kernel monitor!
Type 'help' for a list of commands.
K>

```
刚刚迈出了第一步，感觉要学习的东西太多了，差距太大。
