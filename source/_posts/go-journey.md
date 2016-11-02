title: go 之旅
date: 2016-09-07 15:28:38
tags: go
categories: 编程
---
七月份的时候有过一个go语言的学习计划，目前已经使用go开发了线上的一个模块，主要是将数据从kafka读出来，进行处理之后存储到elasticsearch，目前来看还是很稳定的，开发效率也比C语言高很多，最近两个月来断断续续的基本都是使用golang来编程，边学变做项目，做完项目又看书再学。

目前这个小项目用到的一些技术点和模块主要有toml，json，regex，http，kafka，goroutine，channel。
下面主要是学习heka的代码如何嵌入一个lua虚拟机到go语言中。

推荐一下个人觉得非常好的go的学习资料：

[go语言入门](https://zengweigang.gitbooks.io/core-go/content/index.html)
[go语言圣经](https://docs.ruanjiadeng.com/gopl-zh/index.html)
[go语言编程](https://book.douban.com/subject/11577300/)

目前前面两本过了一遍了，本月会找时间将《go语言编程》过一遍，每本书的代码和习题要跟着做一遍就可以了。自己的路线是看书，做项目，再回过头来从头看书，再做项目，如此三遍基本上可以说go语言就入门了。

目前使用的编辑器是Atom，通过配置一系列的go语言插件，不比任何的IDE差，支持函数跳转，标准库的函数也可以直接跳转，代码自动格式化，语法高亮，自动提示，可以直接运行单个go脚本，符号树展示，各种IDE有的都有，IDE没有的也有。

go1.7已经release了。使用过程中感觉最爽的就是go的goroutine和channel和select加上timer的组合使用，加一个定时器功能和线程就改几行代码完事，如果用c的话估计各种pthread，sleep，if之类的就得半天，而且go开发的代码感觉特别清晰，层次感很强。再加上reflect包的使用，很短的代码量就可以实现很强大的功能，做一些通用的框架还是非常不错的。

唯一不爽的是指针和非指针访问数据的方式相同，作为一个C语言党表示很不习惯。目前总共的有效go代码量还没超过1000行，等到10000行代码量的时候估计就可以在简历上说自己熟练掌握go语言了，这个时间点个人感觉应该会在2017年6月份。
