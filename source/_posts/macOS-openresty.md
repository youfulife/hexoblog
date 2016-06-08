title: Mac下安装openresty
date: 2016-01-05 11:32:45
tags: openresty
categories: 软件安装
---

最新Mac OSX 10.11.1 下安装 openresty 有一点小小的麻烦，记录一下：

第一步根据官网安装需要的依赖：

```bash
brew install pcre openssl
```

其实最后我们会发现这样安装的openssl并没有什么用。
configure 过程中会遇到 openssl找不到的错误，需要手动指定openssl 源码包。

```bash
wget https://www.openssl.org/source/openssl-1.0.2e.tar.gz
tar -xzvf openssl-1.0.2e.tar.gz
```
链接的时候会发现找不到pcre的动态库，需要手动置顶prce的动态库。

最终成功configure和make的命令如下：

```
./configure --prefix=/opt/openresty --with-openssl=../openssl-1.0.2e --with-cc-opt='-I/usr/local/Cellar/pcre/8.37/include/' --with-ld-opt='-L/usr/local/Cellar/pcre/8.37/lib' -j5

make -j5

sudo make install
```
