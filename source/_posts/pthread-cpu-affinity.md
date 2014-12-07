title: pthread和cpu affinity头文件冲突
date: 2014-12-07 12:02:35
categories: linux
tags: linux
---
为了提高多线程程序的性能，有时候需要将线程绑定到固定的cpu core上。在这个过程中一不小心就会产生编译的问题，但是明明头文件都定义了，却依然编译通不过。不巧我就遇到了，google也基本搜不到这个问题的解决方案，没办法，只能自己解决了。
下面这个程序就会出现这种问题：
<!--more-->
``` c
#include <stdio.h>
#include <pthread.h>
#define __USE_GNU
#include <sched.h>
void mybind_cpu(int cpu_id)
{
cpu_set_t mask;
//! sched_setaffinity
CPU_ZERO(&mask);
CPU_SET(cpu_id, &mask);
if (sched_setaffinity(0, sizeof(cpu_set_t), &mask) < 0) {
printf(“Error: cpu id %d sched_setaffinity\n”, cpu_id);
printf(“Warning: performance may be impacted \n”);
}
return;
}
void test_thread(void *cpu_id)
{
int cpuid = (int)(long)cpu_id;
mybind_cpu(cpuid);
printf(“test success!\n”);
return;
}
int main(int argc, char **argv)
{
int res;
void *thread_result;
pthread_t test_thread_ctrl;
pthread_create(&test_thread_ctrl, NULL, (void *)test_thread, (void *)0);
res = pthread_join(test_thread_ctrl, &thread_result);
(void)res;
return 0;
}
```
运行编译命令，编译无法通过：
```bash
gcc test.c -g -Wall -lpthread -o test
```
错误如下：
```
test.c: In function ‘mybind_cpu’:
test.c:10:5: warning: implicit declaration of function ‘CPU_ZERO’ [-Wimplicit-function-declaration]
CPU_ZERO(&mask);
^
test.c:11:5: warning: implicit declaration of function ‘CPU_SET’ [-Wimplicit-function-declaration]
CPU_SET(cpu_id, &mask);
^
test.c:12:5: warning: implicit declaration of function ‘sched_setaffinity’ [-Wimplicit-function-declaration]
if (sched_setaffinity(0, sizeof(cpu_set_t), &mask) < 0) {
^
/tmp/cc0z5zPj.o: In function `mybind_cpu’:
/home/youfu/test.c:10: undefined reference to `CPU_ZERO’
/home/youfu/test.c:11: undefined reference to `CPU_SET’
collect2: error: ld returned 1 exit status
```
可以看到提示一些变量没有定义， 但是明明include了头文件<sched.h>，而且前面添加了#define __USE_GNU，莫非这些宏定义是在别的头文件中吗？
打开/usr/include/sched.h文件， 发现只要#define __USE_GNU，那么就会定义相关的宏，sched.h文件相关定义的源码如下：
``` c
#ifdef __USE_GNU
/* Access macros for `cpu_set’. */
# define CPU_SETSIZE __CPU_SETSIZE
# define CPU_SET(cpu, cpusetp) __CPU_SET_S (cpu, sizeof (cpu_set_t), cpusetp)
# define CPU_CLR(cpu, cpusetp) __CPU_CLR_S (cpu, sizeof (cpu_set_t), cpusetp)
# define CPU_ISSET(cpu, cpusetp) __CPU_ISSET_S (cpu, sizeof (cpu_set_t), \
cpusetp)
# define CPU_ZERO(cpusetp) __CPU_ZERO_S (sizeof (cpu_set_t), cpusetp)
# define CPU_COUNT(cpusetp) __CPU_COUNT_S (sizeof (cpu_set_t), cpusetp)
# define CPU_SET_S(cpu, setsize, cpusetp) __CPU_SET_S (cpu, setsize, cpusetp)
# define CPU_CLR_S(cpu, setsize, cpusetp) __CPU_CLR_S (cpu, setsize, cpusetp)
# define CPU_ISSET_S(cpu, setsize, cpusetp) __CPU_ISSET_S (cpu, setsize, \
cpusetp)
# define CPU_ZERO_S(setsize, cpusetp) __CPU_ZERO_S (setsize, cpusetp)
# define CPU_COUNT_S(setsize, cpusetp) __CPU_COUNT_S (setsize, cpusetp)
# define CPU_EQUAL(cpusetp1, cpusetp2) \
__CPU_EQUAL_S (sizeof (cpu_set_t), cpusetp1, cpusetp2)
# define CPU_EQUAL_S(setsize, cpusetp1, cpusetp2) \
__CPU_EQUAL_S (setsize, cpusetp1, cpusetp2)
# define CPU_AND(destset, srcset1, srcset2) \
__CPU_OP_S (sizeof (cpu_set_t), destset, srcset1, srcset2, &)
# define CPU_OR(destset, srcset1, srcset2) \
__CPU_OP_S (sizeof (cpu_set_t), destset, srcset1, srcset2, |)
# define CPU_XOR(destset, srcset1, srcset2) \
__CPU_OP_S (sizeof (cpu_set_t), destset, srcset1, srcset2, ^)
# define CPU_AND_S(setsize, destset, srcset1, srcset2) \
__CPU_OP_S (setsize, destset, srcset1, srcset2, &)
# define CPU_OR_S(setsize, destset, srcset1, srcset2) \
__CPU_OP_S (setsize, destset, srcset1, srcset2, |)
# define CPU_XOR_S(setsize, destset, srcset1, srcset2) \
__CPU_OP_S (setsize, destset, srcset1, srcset2, ^)
# define CPU_ALLOC_SIZE(count) __CPU_ALLOC_SIZE (count)
# define CPU_ALLOC(count) __CPU_ALLOC (count)
# define CPU_FREE(cpuset) __CPU_FREE (cpuset)
/* Set the CPU affinity for a task */
extern int sched_setaffinity (__pid_t __pid, size_t __cpusetsize,
const cpu_set_t *__cpuset) __THROW;
/* Get the CPU affinity for a task */
extern int sched_getaffinity (__pid_t __pid, size_t __cpusetsize,
cpu_set_t *__cpuset) __THROW;
#endif
```
好的，没办法，通过gcc的预编译选项来查看到底预编译的代码是什么， 在预处理生成的文件中有没有相关定义，运行下面命令生成预处理文件test.i
``` bash
gcc -E test.c -g -Wall -lpthread -o test.i
```
预编译生成的文件非常大，大概有2000多行，找到其中sched.h相关的部分，发现预处理没有把#define __USE_GNU相关的宏定义包含进来：
``` c
extern int sched_setparam (__pid_t __pid, const struct sched_param *__param)
__attribute__ ((__nothrow__ , __leaf__));
extern int sched_getparam (__pid_t __pid, struct sched_param *__param) __attribute__ ((__nothrow__ , __leaf__));
extern int sched_setscheduler (__pid_t __pid, int __policy,
const struct sched_param *__param) __attribute__ ((__nothrow__ , __leaf__));
extern int sched_getscheduler (__pid_t __pid) __attribute__ ((__nothrow__ , __leaf__));
extern int sched_yield (void) __attribute__ ((__nothrow__ , __leaf__));
extern int sched_get_priority_max (int __algorithm) __attribute__ ((__nothrow__ , __leaf__));
extern int sched_get_priority_min (int __algorithm) __attribute__ ((__nothrow__ , __leaf__));
extern int sched_rr_get_interval (__pid_t __pid, struct timespec *__t) __attribute__ ((__nothrow__ , __leaf__));
# 125 “/usr/include/sched.h” 3 4
```
第一感觉这个肯定不是gcc的bug， 这就说明__USE_GNU这个宏定义不知到为什么没有起作用。
只能是pthread.h在作怪，打开/usr/include/pthread.h， 重大发现：
``` c
#include <features.h>
#include <endian.h>
#include <sched.h>
#include <time.h>
#include <bits/pthreadtypes.h>
#include <bits/setjmp.h>
#include <bits/wordsize.h>
```
在pthread.h中也include 了<sched.h>，但是没有定义__USE_GNU，所以后面的include <sched.h>就没有作用了。
其实从预编译生成的文件就可以看出来， pthread相关的定义是在最后，但是明显我们是pthread.h是在sched.h前面的，这个也可以说明我们自己的c文件中的sched.h没有被包含进来，而是包含别的头文件中的sched.h
解决方案就很简单了，将pthread.h的定义放到sched.h的后面就可以了。
``` c
#include <stdio.h>
#define __USE_GNU
#include <sched.h>
#include <pthread.h>
```
``` bash
gcc test.c -g -Wall -lpthread -o test
./test
test success!
```
或者是直接去掉#include <sched.h>, 如下：
``` c
#include <stdio.h>
#define __USE_GNU
#include <pthread.h>
```
打完收工！