title: 女青年学Python
date: 2014-12-09 21:56:33
tags: python
categories: python
---

在味全工作4年后，立志进入IT圈的大龄女青年开始了自己的Python之旅。
在看书第二个晚上后照着书敲出了下面的代码并成功运行，而且理解了程序中的import含义，while 循环，if else 条件判断概念， 顺便对良好的编码风格有了一定的认识。
最最关键的是学习计算机之前就对计算机指令有一个理性的认识，她能够认识到所有的鼠标键盘操作最后会转化成一条条的硬件指令，她也不会对在一个linux下用终端啪啪啪敲打命令的行为产生任何的崇拜感，并且觉得这和用鼠标操作没有任何本质的区别，都是输入指令让计算机执行而已。
<!-- more -->
我觉得就是上面这个feeling就可以甩很多不懂计算机的女生几条街了。目前我可以感觉到，此女子如果一步一步踏实的走下去，三年后IT圈又多了一个有竞争力的IOS女程序媛。
``` python
__author__ = 'wu yuan'

import random
secret = random.randint(1,99)
guess = 0
tries = 0
print "AHOY! I'm dread Wu yuan, and I have a secret"

while guess != secret and tries < 6:
    guess = input("What's yer guess?")
    if guess < secret:
        print "Too low, ye scurvy dog!"
    elif guess > secret:
        print "Too high, landlubber!"
    tries += 1

if guess == secret:
    print "OH, Ye got it! Found my secret, ye did!"
else:
    print "No more guesses, Better luck next time, matey!"
    print "The secret number was", secret
```
在这里推荐一本无任何编程基础的入门书：
《Hello World！ Computer Programming for Kids and Other Beginners》
这本书适合8-88岁任何想学习编程的书，书中图文并茂的解释了很多书中没有的编程要素基本概念。比如，什么是编程？什么是变量？变量名字是什么？作者下了很大的功夫去研究如何教会自己的孩子编程，并且把这个方法贡献给全世界。
虽然是Python语言，如果理解了这本书中的内容，学习其他类似语言只是语法上的差异了。
这本书的中文翻译名字有点怪怪的《父与子的编程之旅：与小卡特一起学Python》。

此书学习完之后的奖励是一台MacBook Air或MacBook Pro。
那个时候此女子就要正式进入IOS开发的世界了。

