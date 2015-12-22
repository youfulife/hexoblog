title: crontab 秒级定时
date: 2015-12-22 16:29:11
tags: linux
categories: linux
---

crontab 默认定时最小只能到1分钟，如果想要15s定时执行一次脚本怎么办？
不想通过while True sleep 这种方式，因为这种方式还得写一个脚本定时检查后台程序是否存在，不存在还得重启。
虽然有monitor，supervisor等后台监控程序可以使用。但是如果能用crontab来实现秒级的定时任务就再好不过了。

下面给出一种方法，虽然有点ugly，但是可以正常工作，而且工作的也比较愉快。

```bash

* * * * * /foo/bar/your_script
* * * * * sleep 15; /foo/bar/your_script
* * * * * sleep 30; /foo/bar/your_script
* * * * * sleep 45; /foo/bar/your_script

```

上述配置就可以通过cron每15s执行一次了，当然如果是1s执行一次，那么需要来60行的配置。
