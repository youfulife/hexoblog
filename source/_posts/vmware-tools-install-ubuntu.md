title: Ubuntu 虚拟机安装 vmware-tools
date: 2015-09-06 21:59:59
tags: linux
categories: 编程
---
开启unity模式，和主机之间共享复制黏贴等，必须安装vmware-tools。

在vmware player的管理菜单栏点击安装vmware-tools，会自动加载虚拟光驱，在桌面上也会显示一个加载的光盘。双击打开所在的文件夹，一般是/media/cdrom0目录。

将里面的VMwareTools-9.9.2-2496486.tar.gz copy到home目录下，解压缩，进入压缩后的目录:
```
    cp /media/cdrom0/VMwareTools-9.9.2-2496486.tar.gz ~/
    tar -xzvf VMwareTools-9.9.2-2496486.tar.gz
    cd vmware-tools-distrib/
```

运行安装命令，加上-d选项表示全部采用默认值，要不然得一下一下的选择yes或者no
```
   ./vmware-install.pl -d
```
现在已经安装好了，但是现在还是没有用的，因为没有配置。
运行配置脚本，对vmware-tools的功能进行配置，一般用默认选项就可以了。
```
    /usr/bin/vmware-config-tools.pl
```
配置完成后重启虚拟机即可，现在unity的功能已经可以用了，和主机之间剪切板也可以共享了。
