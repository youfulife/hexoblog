title: 编程水平进阶之路
date: 2014-12-13 20:27:11
tags: linux
categories: 编程
---
下面这些都是从知乎或者一些其他网站收集的，觉得挺好，接下来的三年一个一个的攻坚，作为C语言进阶，应该读一些不是太大，而且编程风格良好的项目，还应该顺带学习一些底层的东西：

词法分析语法分析：
	* flex and Bison

网络编程：

	* redis 是单线程异步网络编程的范例，带注释的redis：huangz1990/annotated_redis_source · GitHub
	* nginx 多进程网络编程的巅峰，模块化
	* memcached 虽然是C++，但是C style的，多线程网络编程的巅峰
<!-- more -->
数据结构&数据库：

	* SQLite 数据理论的范例。要去读非合并源文件版的（为了方便编译器优化，有个单文件版的）。

大杂烩类型：

	* Coreutils - GNU core utilities 大多数Linux系统命令的实现
    * Python源代码（CPython，注意不是Cython）多少次遇到百思不得其解的问题，都可以从Python源码里找到答案。

底层：
    * xv6，6.828 / Fall 2014 熟悉底层，了解操作系统的实现，xv6文档齐全。

    * lua，The Programming Language Lua 学习虚拟机的实现和运行。

    * lcc，drh/lcc · GitHub，学习Parser，编译器，顺带一本书A Retargetable C Compiler (豆瓣)。
