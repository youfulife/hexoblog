title: linux stack 之旅
date: 2016-06-16 17:06:09
tags: ["内存", "一图胜千言"]
categories: 编程
---

最近一直在看[Gustavo Duarte](http://duartes.org/gustavo/blog/)内存相关的文章，写得太好了，读了一篇根本停不下来。
```C
int add(int a, int b)
{
  int result = a + b;
  return result;
}

int main(int argc)
{
  int answer;
  answer = add(40, 2);
}
```
假设在linux 32位平台的命令行下直接执行，[journey-to-the-stack](http://duartes.org/gustavo/blog/post/journey-to-the-stack/)中一步一步的给出栈的变化图谱。


还是一图胜千言啊。

![](http://7teb9r.com1.z0.glb.clouddn.com/callSequence.png)
