title: lua coroutine 学习笔记
date: 2015-04-25 15:02:08
tags: lua
categories: 编程
---


最近刚刚接触lua，一步一步学习中，之前也没怎么接触过协程这个概念，花了一天时间看了两三遍lua programming 3 中的协程章节。
```
co = coroutine.create(function ()
    for i = 1, 10 do
        print("co", i)
        coroutine.yield()
    end
end)
```
程序会停留在coroutine.yield() 函数内部的某个地方阻塞住 (挂起)，再次调用
```
coroutine.resume(co)
```
程序会继续从上次阻塞(挂起)的地方继续执行，然后 coroutine.yield()函数会返回。
上面的例子中，for循环会继续执行。

类似于操作系统中进程执行的几种状态，这样程序员可以手动的(模拟内核）去控制程序的运行，挂起，结束等等，操作系统级别的线程（比如pthread）运行之后程序员是无法控制的，都是由操作系统来进行调度，由于时序或者调度造成的一些随机bug也比较难以调试。

上面程序的执行过程如下：
```
coroutine.resume(co) --> co 1
coroutine.resume(co) --> co 2
coroutine.resume(co) --> co 3
...
coroutine.resume(co) --> co 10
coroutine.resume(co) -- prints nothing

print(coroutine.resume(co))
--> false cannot resume dead coroutine
```
第11次resume的时候，程序从yield函数返回，然后for循环结束，并不会继续执行打印语句，整个coroutine 执行完毕，进入dead状态，所以第12次调用resume的时候就会失败。


程序员完全控制程序的切换，语言自动保存了需要的上下文状态，唤醒和挂起之间可以通过参数来进行通信，同步的方式实现异步， 不同coroutine的切换不用陷入到内核态，效率比较高，同一时间只有一个coroutine例程在执行。

lua coroutine 参数传递：

<!-- more -->

示例1：
```
co = coroutine.create(function (a, b, c) print("co", a, b, c + 2) end)
coroutine.resume(co, 1, 2, 3) --> co 1 2 5
```

在coroutine的主函数中，没有相应的yeild在等待，也就是说上下文还不处于yeild函数中，也不知道有没有yeild，那么第一次调用resume时，传进去的多余的参数（比如上面的1,2,3）就会传递给coroutine的function (a, b, c)函数。
最终打印出1， 2， 5。

coroutine.resume函数返回什么呢，在上面的例子中返回的是true，表示没有错误。

示例2：
```
> co = coroutine.create(function (a, b) coroutine.yield(a+b, a-b) end)
> coroutine.resume(co, 20, 40)
true     60     -20
> return coroutine.resume(co, 20, 40)
true
> return coroutine.resume(co1, 20, 40)
false     cannot resume dead coroutine
```

示例3：
```
> co1 = coroutine.create(function (a, b) coroutine.yield(a+b, a-b) return 5,12 end)
> return coroutine.resume(co1, 20, 40)
true     60     -20
> return coroutine.resume(co1, 20, 4)
true     5     12
> return coroutine.resume(co1, 20)
false     cannot resume dead coroutine
```
第一次resume额外的参数20, 40会传递给function (a, b), 然后遇到了yield函数，传递给yield函数的参数是60， -20，程序挂起，然后返回，可以看到resume的返回值为传递给yield函数的参数。
再次resume后，yield程序继续运行，协程程序结束，resume返回值为function本身的返回值，示例2中无返回值，示例3中的返回值为5， 12

示例4：

coroutine.yield函数的返回值：
```
> co = coroutine.create(function (a, b) print(coroutine.yield(a+b, a-b)) return 5,12 end)
> return coroutine.resume(co, 20, 4)
true     24     16                            // resume的返回值为 进入yield函数时，yeild的参数值
> return coroutine.resume(co, 20, 25)
20     25                                       // yield的返回值为本次调用resume激活程序时， resume的参数值
true     5     12                             // 协程程序结束时，function主函数的返回值
> return coroutine.resume(co, 20, 25)
false     cannot resume dead coroutine
```
分3步走：
1. 第一次程序执行resume，程序在yield函数中的某个地方挂起来了（但是并不阻塞在那里不动），同时第一次resume函数会返回。
2. 再次调用resume函数让程序再次从上次停止的地方(yield函数中的某个地方)运行，yield函数执行完毕后就会返回。
3. 程序从yield返回后一直往下执行，由于没有yield函数了，程序执行到end就结束了，状态变成dead状态。

第一次resume的返回值就是第一次调用yield函数时传递给yield的参数值，
第一次yield函数的返回值就是第二次调用resume函数时传递给resume的参数值。
第二次resume的返回值是function (a, b)的返回值。

整个过程总共调用了2次resume和一次yield函数。


示例5：
```
> co = coroutine.create(function (a, b) print(coroutine.yield(a+b, a-b)) print(coroutine.yield(34, 43)) return 5,12 end)
> return coroutine.resume(co, 20, 4)
true 24 16               // 第一次resume返回值
> print(coroutine.status(co))
suspended
> return coroutine.resume(co, 20, 25)
20 25                     //第一次yield返回值
true 34 43            // 第二次resume返回值
> return coroutine.resume(co, 3, 5, 7)
3 5 7               // 第二次yield返回值
true 5 12             // 第三次resume返回值
> return coroutine.resume(co, 3, 5, 7)
false cannot resume dead coroutine
>
```
