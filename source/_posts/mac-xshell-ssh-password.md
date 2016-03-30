title: Mac下iTerm2 代替 Xshell
date: 2016-03-24 13:34:52
tags: mac
---

mac 下通过配置 iTerm2 让它具有xshell免密码登陆的功能。iTerm2 本身并没有提供这个功能，不过
可以借助第三方工具来实现。

### 下载sshpass工具

[下载地址](http://heanet.dl.sourceforge.net/project/sshpass/sshpass/1.05/sshpass-1.05.tar.gz)
### 编译安装sshpass
```
tar -xzvf sshpass-1.05.tar.gz
cd sshpass-1.05
./configure
make
sudo make install
```
### 配置 sshpass
sshpass 通过配置文件读取密码，只要把密码写道文本文件中就可以。比如我有个内网的机器root的密码是123456，那么直接将123456写到某个文本文件中即可，目前本机的配置文件放在~/sshpass/offline文件中。
```
123456
```
### 配置iTerm2
在iTerm2->Profiles->General下新建一个profile。
在Command选项中选择Command，并填写下面内容：
```
/usr/local/bin/sshpass -f /Users/youfu/sshpass/offline ssh -p22 root@192.168.10.37
```
其中/Users/youfu/sshpass/offline中保存了root@192.168.10.37的密码123456

这样通过Command + o快捷键可以呼出Profiles面板，选择要连接的主机，不需要输入密码了。

而且通过这样配置后，在同一个标签下Command + d快捷键分屏的时候会自动登录到远程的机器上。
