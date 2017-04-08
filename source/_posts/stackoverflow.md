title: 栈溢出学习入门总结
date: 2017-04-08 23:31:23
tags: security
categories: 安全
---
最近在入门二进制安全，第一个便是缓冲区溢出，有漏洞的代码大致长得下面这个样子。

```
int vuln() {
    char buf[80];
    int r;
    r = read(0, buf, 400);
    printf("\nRead %d bytes. buf is %s\n", r, buf);
    puts("No shell for you :(");
    return 0;
}
```
要想成功的利用漏洞，首选需要了解目前linux下针对栈溢出通用的一些安全措施。
CANARY, ASLR, NX
了解了上述三种保护机制之后，得找到绕过相关保护措施的方法。

## 基础知识

学习过程中必需熟练掌握的基础知识：

1. GDB相关使用技巧，常用命令
2. 能够看懂32位和64位下汇编语言
3. 32位和64位下函数调用过程，参数传递方式
4. 可执行程序ELF基础知识
5. 可执行程序虚拟内存布局，用户空间每一个字节都是干什么用的
6. 可执行程序动态链接过程，link_map struct, dl_runtime_resolve, PLT
7. 如何通过DynELF模块来查找libc中函数地址以及DynELF原理和实现
8. 通用gadgets查找，尤其是64位程序如何控制传递参数的寄存器

## 牛逼工具
工欲善其事，必先利其器。
掌握了相关原理之后，需要使用工具来提升效率，学习过程中经常使用的工具主要是下面一些：
gdb, file, ldd, strace, peda, pwntools, readelf, objdump, socat, ROPgadget

## 相关资料
自己在学习过程中参考的相关资料总结：
《深入理解计算机系统》 第三章 主要讲解32位和64位汇编和函数调用过程，其中第二版是32位，第三版是64位。

《程序员的自我修养》ELF和动态链接以及延迟绑定机制。
《64-bit-linux-stack-smashing-tutorial-part-3系列》
https://blog.techorganic.com/2016/03/18/64-bit-linux-stack-smashing-tutorial-part-3/

《一步一步学ROP系列》
https://github.com/zhengmin1989/ROP_STEP_BY_STEP
主要讲解如何绕过ASLR以及如何查找通用的gadgets。

《Linux (x86) Exploit Development 系列》
https://sploitfun.wordpress.com/2015/06/26/linux-x86-exploit-development-tutorial-series/

《stack-chk-fail 系列》
http://yunnigu.dropsec.xyz/2017/03/04/%E6%A0%88%E6%BA%A2%E5%87%BA%E4%B9%8B%E5%88%A9%E7%94%A8-stack-chk-fail/
主要讲解如何利用canary检查失败的机制leak信息

## 实战练手
目前主要是通过刷javios OJ上的pwn题目来检验自己学习的掌握程序。
https://www.jarvisoj.com/challenges
整个过程中的writeup都已经提交到github中了。
https://github.com/chenyoufu/writeups

## 下一步计划
堆溢出相关的漏洞利用学习。
