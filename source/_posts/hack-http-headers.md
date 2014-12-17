title: hack http headers
date: 2014-12-17 16:30:20
tags: chrome
categories: 安全
---
从零到chrome插件正式发布，历时一月有余。靠着不停的F5和F12，终于可以正常使用了，目前仅剩一下扫尾的美化和代码清理工作啦。

一个月的时间熟悉了chrome扩展开发流程，html+css+javascript基础知识，jquery+bootstrap基本用法。现在对网页的布局，简单效果的各种实现慢慢的有了那么一点点的感觉。不好捉摸，但是隐隐嗅出那么一丝丝的味道来了。

hack http headers扩展界面参考了live http headers扩展，少量的js和html代码直接就拿过来用了。主要的改变如下：
	> * jquery和bootstrap使用的是截止到目前为止最新的版本，jquery-2.1.1和bootstrap-3.2.0。
	> * 增加了http 头部的修改，以及对显示结果的过滤，添加了request发起的时间。
	> * 修复了live http headers有时request和response不对应的一个bug。
	> * 去掉了live http headers中的raw 功能。

计划中下一版本将会增加的功能：

	> 1. cookies修改过后本地持久化选项。
	> 2. 增加自定义http header功能。(目前版本不能通过注入\r\n的方式增加头部或者设置多个相同头部)。
	> 3. 增加针对单个tab的选项。
	> 4. 增加更多的过滤规则，比如可以根据refer过滤抓取和显示，可以仅仅显示post的请求。
	> 5. 增加对post内容的修改？(这个主要看chrome接口是否会开放出来)

这个项目的github主页如下：
https://github.com/chenyoufu/hack-http-headers
欢迎小伙伴们提供改进的建议以及合理的需求。

下载地址：http://pan.baidu.com/s/1qWJNXV2

由于google在chrome稳定版中禁用了chrome官方商店之外的应用，但是要想上传到google chrome商店必须支付5美元的开发者费用，但是google又禁用了大陆用户使用google钱包，搞了半晚上都没有搞定。

所以建议小伙伴们使用测试版的chrome进行操作。