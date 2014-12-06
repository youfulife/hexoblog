title: linux查看多线程运行状态

date: 2014-12-06 17:53:38

categories: linux

tags: linux

---


### **pstack** 

如果知道运行的程序的名字，用pgrep name来得到进程的pid，然后运行：

``` bash
pstack pid
```
也可以直接运行
``` bash
pstack $(pgrep process_name)
```
就可以打印出当前进程所有线程的执行栈，里面有当前执行到哪一个函数， 函数的调用关系等，可以写一个shell脚本定时打印。
### **pstree**
这个命令可以看到当前运行的线程树， 运行
``` bash
pstree -p pid
```
则只看进程号为pid的线程树
### **top**
如果线程设置了cpu亲和性， 上面这两个命令是看不出来线程运行在哪个cpu的，之前一直用ps命令后面加一大堆各种参数， 打印格式来看， 实在是记不住， 每次都要去翻笔记， 后来发现top命令是可以的， 而且可以实时查看。
``` bash
top -p $(pgrep process_name)
```
然后输入H， 表示根据线程信息来显示， 再输入f，进入field界面， 输入j，表示显示last use cpu信息。
可以看到有一个P选项， 上面就是线程最后一次所运行的cpu了， 可以观察一段时间来判断我们设置的cpu affinity是否起作用。