title: dpdk多线程管理控制机制
date: 2014-12-30 15:11:31
tags: dpdk
categories: 编程
---


![EAL Initialization in a Linux Application Environment](http://7sbqk1.com1.z0.glb.clouddn.com/youfu_blog_linuxapp_launch.png)

一图胜千言！rte_eal_init函数中根据输入的cpu core id通过pthread来创建所有子线程。
<!-- more -->
``` c
     RTE_LCORE_FOREACH_SLAVE(i) {

          /*
          * create communication pipes between master thread
          * and children
          */
          if (pipe(lcore_config[i].pipe_master2slave) < 0)
               rte_panic("Cannot create pipe\n");
          if (pipe(lcore_config[i].pipe_slave2master) < 0)
               rte_panic("Cannot create pipe\n");

          lcore_config[i].state = WAIT;

          /* create a thread for each lcore */
          ret = pthread_create(&lcore_config[i].thread_id, NULL,
                         eal_thread_loop, NULL);
          if (ret != 0)
               rte_panic("Cannot create thread\n");
     }
```
下面的两个dummy function确保所有的子线程都准备完毕了。
```
    /*
     * Launch a dummy function on all slave lcores, so that master lcore
     * knows they are all ready when this function returns.
     */
     rte_eal_mp_remote_launch(sync_func, NULL, SKIP_MASTER);
     rte_eal_mp_wait_lcore();
```

子线程和主线程进行通信的结构体如下，里面主要是两组pipe，这个地方没有弄明白，为什么不能用状态来进行同步，而非要搞两组pipe：
``` c
/**
 * Structure storing internal configuration (per-lcore)
 */
struct lcore_config {
     unsigned detected;         /**< true if lcore was detected */
     pthread_t thread_id;       /**< pthread identifier */
     int pipe_master2slave[2];  /**< communication pipe with master */
     int pipe_slave2master[2];  /**< communication pipe with master */
     lcore_function_t * volatile f;         /**< function to call */
     void * volatile arg;       /**< argument of function */
     volatile int ret;          /**< return value of function */
     volatile enum rte_lcore_state_t state; /**< lcore state */
     unsigned socket_id;        /**< physical socket id for this lcore */
     unsigned core_id;          /**< core number on socket for this lcore */
     int core_index;            /**< relative index, starting from 0 */
};
```
在主线程中调用下面的 rte_eal_remote_launch 函数，给处于wait状态的线程发送一个命令，让子线程运行函数f。
``` c
/*
* Send a message to a slave lcore identified by slave_id to call a
* function f with argument arg. Once the execution is done, the
* remote lcore switch in FINISHED state.
*/
int
rte_eal_remote_launch(int (*f)(void *), void *arg, unsigned slave_id)
{
     int n;
     char c = 0;
     int m2s = lcore_config[slave_id].pipe_master2slave[1];
     int s2m = lcore_config[slave_id].pipe_slave2master[0];

     if (lcore_config[slave_id].state != WAIT)
          return -EBUSY;

     lcore_config[slave_id].f = f;
     lcore_config[slave_id].arg = arg;

     /* send message */
     n = 0;
     while (n == 0 || (n < 0 && errno == EINTR))
          n = write(m2s, &c, 1);
     if (n < 0)
          rte_panic("cannot write on configuration pipe\n");

     /* wait ack */
     do {
          n = read(s2m, &c, 1);
     } while (n < 0 && errno == EINTR);

     if (n <= 0)
          rte_panic("cannot read on configuration pipe\n");

     return 0;
}
```
每一个子线程都是一个while(1)循环，初始化完成后处于WAIT状态，等待接收主线程的命令，收到运行函数的命令后，设置RUNNING状态，开始执行函数，执行完毕后设置FINISHED状态，等待接收下一次命令。
``` c
/* main loop of threads */
__attribute__((noreturn)) void *
eal_thread_loop(__attribute__((unused)) void *arg)
{
     char c;
     int n, ret;
     unsigned lcore_id;
     pthread_t thread_id;
     int m2s, s2m;

     thread_id = pthread_self();

     /* retrieve our lcore_id from the configuration structure */
     RTE_LCORE_FOREACH_SLAVE(lcore_id) {
          if (thread_id == lcore_config[lcore_id].thread_id)
               break;
     }
     if (lcore_id == RTE_MAX_LCORE)
          rte_panic("cannot retrieve lcore id\n");

     RTE_LOG(DEBUG, EAL, "Core %u is ready (tid=%x)\n",
          lcore_id, (int)thread_id);

     m2s = lcore_config[lcore_id].pipe_master2slave[0];
     s2m = lcore_config[lcore_id].pipe_slave2master[1];

     /* set the lcore ID in per-lcore memory area */
     RTE_PER_LCORE(_lcore_id) = lcore_id;

     /* set CPU affinity */
     if (eal_thread_set_affinity() < 0)
          rte_panic("cannot set affinity\n");

     /* read on our pipe to get commands */
     while (1) {
          void *fct_arg;

          /* wait command */
          do {
               n = read(m2s, &c, 1);
          } while (n < 0 && errno == EINTR);

          if (n <= 0)
               rte_panic("cannot read on configuration pipe\n");

          lcore_config[lcore_id].state = RUNNING;

          /* send ack */
          n = 0;
          while (n == 0 || (n < 0 && errno == EINTR))
               n = write(s2m, &c, 1);
          if (n < 0)
               rte_panic("cannot write on configuration pipe\n");

          if (lcore_config[lcore_id].f == NULL)
               rte_panic("NULL function pointer\n");

          /* call the function and store the return value */
          fct_arg = lcore_config[lcore_id].arg;
          ret = lcore_config[lcore_id].f(fct_arg);
          lcore_config[lcore_id].ret = ret;
          rte_wmb();
          lcore_config[lcore_id].state = FINISHED;
     }

     /* never reached */
     /* pthread_exit(NULL); */
     /* return NULL; */
}
```
相关函数的具体解释参考API文档：
http://www.dpdk.org/doc/api/rte__launch_8h.html
