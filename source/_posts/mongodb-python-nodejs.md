title: mongodb python 和 nodejs 基于时间查询
date: 2015-12-22 14:04:18
tags: mongodb
categories: mongodb
---

mongodb 使用过程中经常遇到查询或者聚合某一段时间内的数据。
基于时间段的查询通常有两种方式，一种是通过_id字段自带的时间属性来对比，一种是在document中添加一个Date类型的字段。
mongodb 中存储的时间格式是UTC的时间，需要不同的客户端自行和本地时间进行同步。

一般情况下，我们会直接调用不同语言的mongodb 库来执行mongodb的语句。下面是python和nodejs两种基于时间的查询。

## ObjectId By Python

基于_id的对比，我们根据当前的时间戳生成一个ObjectId的对象，然后和mongodb中存储的_id进行对比。

``` python

from bson.tz_util import utc
import time
import bson
import datetime


def dummy_id(timestamp):
    gen_time = datetime.datetime.fromtimestamp(timestamp, utc)
    print gen_time
    return bson.objectid.ObjectId.from_datetime(gen_time)

```
直接调用 dummy_id(time.time()) 既可以返回一个基于当前时间生成的ObjectId，这个ObjectId后面几位全部都是0。
<!-- more -->
所以直接使用下面语法：

```
{'$gt': dummy_id(start), '$lt': dummy_id(end)}
```
就可以查询到大于等于start，小于end的纪录，而不用考虑边界效益。为什么是大于等于start，因为mongodb自动生成的_id后面几位都不是0。

这里值得注意的是要自己处理和utc之间的差别，python并不会帮忙处理。

## CreatedAt By Python

首先需要在document中存在Date格式的字段，这里取名为CreatedAt。
很多时候我们查询某个时间段的数据都需要对时间取整，基于timestamp获取时间格式首先就要对时间戳进行取整，比如查询某一个天
的所有数据，后面的小时，分钟，秒都变成0。

``` python

import time
import datetime


def ts_round(tms, interval):
    return tms - tms % interval


def date_round(tms):
    gen_time = datetime.datetime.utcfromtimestamp(tms)
    return gen_time

interval = 60 * 60
start = ts_round(time.time() - interval, interval)
end = ts_round(time.time(), interval)
match = {'$gte': date_round(start), '$lt': date_round(end)}

```

上面的代码是查询最近1小时的数据，比如现在是3点05分，那么这个数据查询的是2点到3点的数据。

## ObjectId and CreatedAt By Nodejs

nodejs不需要自己处理UTC时间，直接把当前时间戳传递过去就行，估计Nodejs的mongodb客户端内部做了转换。

```javascript
var objectIdWithTimestamp = function (timestamp) {
  if (typeof(timestamp) == 'string') {
    timestamp = new Date(timestamp);
  }
  var hexSeconds = Math.floor(timestamp/1000).toString(16);
  var constructedObjectId = ObjectId(hexSeconds + "0000000000000000");
  return constructedObjectId;
}

var ts_round = function (tms, interval) {
  return tms - tms % interval;
}

var default_match = function(key) {
  var start = ts_round(Date.now() - interval, interval);
  var end = ts_round(Date.now(), interval);
  logger.info("start: " + new Date(start));
  logger.info("end: " + new Date(end));

  if(key == 'createdAt') {
    return { createdAt: {
      $gte: new Date(start),
      $lt: new Date(end)
    }};
  }
  if(key == '_id') {
    return { _id: {
      $gte: objectIdWithTimestamp(start),
      $lt: objectIdWithTimestamp(end)
    }};
  }
};

```

这个里面最大的坑就是对于utc时间戳的处理。由于对nodejs不是很熟悉，异步加同步操作hold不住，而且nodejs出错了并不会主动退出。
这样就需要捕获所有的异常，否则会一只占用着mongodb的连接，后台用cron运行nodejs脚本一个周发现光nodejs进程最后就有1000多个连接，生生把mongodb拖到死。

用了这个异常捕捉函数发现并没有用，异常发生时并没有主动退出，不清楚什么原因，估计是用了logger的问题？

```javascript
process.on('uncaughtException', function (err) {
    logger.error('Error: %s', err.message);
    process.exit(1);
});

```

还是python大法好。

python的完整代码在github上：
https://github.com/chenyoufu/shadow

主要是插件式开发，离线处理，mongodb相关的聚合语法。
