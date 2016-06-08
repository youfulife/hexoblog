title: python property setter 多参数
date: 2015-12-23 17:18:31
tags: python
categories: python
---

python property的好处就不多说了，从[廖雪峰博客](http://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000/001386820062641f3bcc60a4b164f8d91df476445697b9e000)中摘抄过来一段：
```
@property广泛应用在类的定义中，可以让调用者写出简短的代码，同时保证对参数进行必要的检查，这样，程序运行时就减少了出错的可能性。
```

这里补充一下使用property setter 时如何传递多个参数进去。答案就是传递一个元组进去。然后在setter 函数中再拆分元组，进行多个参数的处理。

<!-- more -->

```python

import collections


class Test(object):

    def __init__(self):
        pass

    @property
    def age(self):
        return self.__age

    @age.setter
    def age(self, value):
        if value > 20:
            self.__age = value
        else:
            raise ValueError('too young, too simple...')

    @property
    def fuck(self):
        return self.__fuck

    @fuck.setter
    def fuck(self, value):
        if isinstance(value, collections.Iterable):
            self.__fuck = 'fucking ' + ''.join(value)
        else:
            self.__fuck = value

    def __getattr__(self, item):
        return None

t = Test()
t.age = 20
print t.age
t.fuck = ('g', 'f', 'w')
print t.fuck
```

输出如下：
```
None
fucking gfw
```

最近在深入学习python，希望半年后能够成为一个高级python程序员。
