title: pycharm中配置github
date: 2014-12-26 16:30:24
tags: python
categories: 编程
---
pycharm 作为python的开发利器，谁用谁知道，没有最好只有更好。
pycharm支持各种版本管理工具，支持github。
但是pycharm本身不带有客户端，需要调用系统的git.exe来进行github repository的管理。

##1. 安装git.exe
官方网站：http://git-scm.com/
安装git.exe的时候有个选项是安装完成后设置git的环境变量，默认的不是这个选项，在一路next的过程中要注意停一下，把这个选上。
安装完成后在cmd中直接运行git，如果出现帮助信息，说明一切顺利。

##2. 设置pycharm
在pycharm中，选择 **VCS菜单—>checkout from version control—>GitHub**。
根据提示填写邮箱，密码，要check出来的repository。
然后pycharm就创建了一个你的repository name的project。
在这个project下创建的文件，修改的代码都可以push到github上。

总的来说，pycharm提供的版本管理功能非常强大，顺带着diff工具也非常好用。
