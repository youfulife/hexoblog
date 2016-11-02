title: elasticsearch bulk 插入性能测试
date: 2016-01-28 10:48:51
tags: ELK
categories: 编程
---

近期对es 集群进行了一次bulk 插入的性能测试，发现在bulk条数比较小的时候，es写入性能挺低的。
暂时还没有搞清楚，为什么bulk的条数对写入性能影响那么大。
初步怀疑是es集群网络状态交换过多。

测试工具： Apache AB 工具
测试URL：  http://192.168.10.84:9200/_bulk

测试持续时间： 60s

### es集群配置：

*es data node*
```
192.168.10.81     8core   14G   800G    es-81
192.168.10.82     8core   14G   800G    es-82
192.168.10.180    4core   16G   500G    es-180
192.168.10.182    4core   16G   500G    es-182
192.168.10.187    4core   16G   500G    es-187
```
*es master node*
```
192.168.10.83    4core   8G   500G    es-83
```
*es client node*
```
192.168.10.84    4core   8G   500G    es-84
192.168.10.85    4core   8G   500G    es-85
```
### es 索引配置：
```
"index": {
    "creation_date": "1453824012161",
    "refresh_interval": "5s",
    "number_of_shards": "5",
    "number_of_replicas": "1",
    "uuid": "OfgXNBFYQsynTKGg9qNBCw",
    "version": {
      "created": "2000099"
    }
  }
}

"mappings": {
  "_default_": {
    "dynamic_templates": [
      {
        "string_fields": {
          "mapping": {
            "index": "not_analyzed",
            "omit_norms": true,
            "type": "string"
          },
          "match_mapping_type": "string",
          "match": "*"
        }
      }
    ],
    "_all": {
      "enabled": false
    }
  }
}
```
### 测试结果：

1、每次bulk的doc数量分别为1、10、1000、10000递增

2、使用的concurrency数也分别为1、10、100

3、发送内容中所有的域都是整数，并没有字符串字段

4、测试时是开启了keepalive的

```
#bulk-1		
concurrency		Requests	docs/sec
1      			3.14 [#/sec]	3.14
10     			17.01 [#/sec]	17.01
100    			39.31 [#/sec]	39.31

#bulk-10		
concurrency		Requests	docs/sec
1			      1.42 [#/sec]	14.2
10	     		5.27 [#/sec]	52.7
100	    		12.21 [#/sec]	122.1

#bulk-100		
concurrency		Requests	docs/sec
1      			1.65 [#/sec]	165
10     			4.78 [#/sec]	478
100   			12.33 [#/sec]	1233

#bulk-1000		
concurrency		Requests	docs/sec
1		      	1.18 [#/sec]	1180
10		     	5.18 [#/sec]	5180
100	    		8.78 [#/sec]	8780

#bulk-10000		
concurrency		Requests	docs/sec
1		      	0.59 [#/sec]	5900
10	     		2.23 [#/sec]	22300
100		    	error	        error
```

一开始是怀疑es自身的http服务器性能太低。于是通过java的API直接使用TransportClient二进制协议每次也是插入一条数据，速率也是3~5左右。
所以基本排除了是http服务器本身性能的影响。

目前猜想是每次一条数据，插入太频繁，es集群网络状态交换过于频繁导致。

看了一下logstash的默认配置，bulk的缓存条数是500, 当然，不满500的话会定时刷新。
