title: 实时告警系统之jepl DSL开发
date: 2016-11-08 11:12:28
tags: 编译原理
categories: 编程
---

![](http://7teb9r.com1.z0.glb.clouddn.com/lsbasi_part7_tree_terminology.png)

近半年业余时间会时不时的看一些编译原理的基础知识。这两个周终于派上用场了。

目前在开发一个json流实时处理的告警系统，需要开发一个DSL来让用户输入告警需要的metric表达式和过滤条件。考虑到运维人员大部分都擅长SQL，决定采用类SQL的方式来和用户交互一些告警相关的信息。这样可以避免web界面显示各种各样的单选框和复选框之类让人生畏的东西，同时还可以保证告警的几个模块元素复用，类似于一个个的小组件，设置一次后就可以通过托拉拽排列组合生成各种各样的告警表达式和过滤条件。

后台全部采用GO语言开发，数据源是一条条的json数据，DSL的作用就是对一条条的json数据进行过滤和聚合，然后生成时间序列的一个个数据点。基本等同于sql的select的一个子集，只有where子句，group by，limit之类的全都不需要。

目前我把这个DSL叫做jepl(JSON Event Process Language)，其实类似于spark streaming sql的一个前端，但是支持的特性肯定比spark sql要少得多。

一般定义好文法后开发一个DSL有两条路，一条是基于lex && yacc之类的工具通过文法自动生成tokenizer和grammar parser以及interpreter，各种语言都有自己的一套类似的工具。一种是自己手工写各个组件，手工写一般也是照着LL的固定套路来。考虑到后期需要进行错误的定制化输出以及语法检查，代码高亮等各种需求，最终还是决定手撸一把。

目前已经开发的差不多了，一开始自己手写了词法分析和语法分析部分，不过质量很烂，后来发现influxql的代码质量相当不错，于是废弃掉了自己的代码，直接基于influxql的代码修改，大部分还是influxql的源代码，不过做了大量的删减和一小部分的修改，还有一些对外接口封装，bug和正确性验证的东西要修改，已经可以输入一个sql语句和一段json数据，输出时间序列的数据点了。后期会投入很大一部分精力在上面。

不得不说对于一个新手来说做这个东西还是有点耗费脑力的，不过这也是在补习大学欠下的债，基本的DSL设计和实现应该是大学本科编译原理课程的基本要求吧。

总结一下近半年来看过的编译原理相关的经典书籍和文章

**[Let’s Build A Simple Interpreter系列](https://ruslanspivak.com/lsbasi-part1/)**

这个系列的文章非常好，里面作者手绘的插图非常可爱，简单易懂，也是我入门看的第一个系列的教程，使用python语言来讲解，从简单的四则运算表达式讲起，后续到复杂的四则运算表达式，再到后面的DSL的解析器，每一个步骤很清晰，而且也有github配套源代码。里面的计算器相关的代码我从头到尾手写了三遍，每隔一段时间感觉要忘记了的时候就从头来一遍。我推荐给了每一个想入门编译原理的新人。

**[Handwritten Parsers & Lexers in Go](https://blog.gopheracademy.com/advent-2014/parsers-lexers/)**

这个文章是Influxql的其中一个开发者写的一个简单sql解析的入门文章，只有词法分析和语法分析，套路和influxql的开发是一样的，也有配套的github源代码。

**[Rob Pike on lexical scanning](https://www.youtube.com/watch?v=HxaD_trXwRE)**

这个是Go语言的作者的一个演讲，将如何用go语言开发一个字符串模版系统的词法分析器，通常的模式是parser调用lexer来一个一个的token解析，但是go语言天生的channel和goroutine特性可以让lexer和parser彻底分离开，中间通过channel来通信。

**[编程语言实现模式](https://book.douban.com/subject/10482195/)**

这本书是我看过的非常好的一个编译原理的入门书，说实话，比龙书容易入门多了，里面不但讲各种模式而且循序渐进的讲了为什么需要这种模式，这种模式比上一种的好处是什么，坏处是什么，解决了哪些问题。这本书需要读三遍。
上面一些博客的通用缺点是直接给出了这本书中的某一模式的代码实现，包括influxql的代码，而这本书就是为各种博客的代码提供了理论的支持，并且抽象出了通用的规则。
