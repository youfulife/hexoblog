title: python 之旅
date: 2016-07-06 22:34:17
tags: python
categories: 编程
---

“If you don’t know how compilers work, then you don’t know how computers work. If you’re not 100% sure whether you know how compilers work, then you don’t know how they work.” — Steve Yegge

最近在用python做一些项目，有公司相关的告警平台，基于elastic alert和skyline。

这里要吐槽一下，elastic alert的代码真是烂到家了，一点都不pythonic，不出意外应该是一个python新手在边学边写。相比之下skyline的代码看起来就舒服多了，至少是pep8没有警告的，而且代码质量挺高的，新手可以作为入门的代码教程。

还有晚上自己用python写的一个计算器，支持 + - * / () 正负号，当然是照着大神的[Let’s Build A Simple Interpreter博客](https://ruslanspivak.com)一步一步学习来的。这个系列的文章入门编译原理真的非常适合，唯一的缺点就是更新太慢，平均2个月才有一篇新的文章。作者一直强调```重复是成功之母```，学习编程也和我们高中时候背课文一样，一次不行来两遍，两边不行来三遍，熟能生巧。

周日之前要完全不参考现有的任何代码，从头再实现一遍计算器。

最近在用skyline时发现python的程序部署起来真的很费劲，尤其是底层依赖C语言的一些库，比如NumPy等科学计算的库，还有跨平台的数据包依赖，用virtual env根本解决不了。和小伙伴讨论了一下，决定告警平台用go语言重写，当然得先用python实现一个简单的能够上线撑半年的。

个人体验，python适用于编程语言入门，快速实现简单原型，运维自动化部署，测试脚本，逻辑简单性能要求不高的场景。
