title: chrome 修改http头部扩展
date: 2015-01-05 16:59:25
tags: chrome
categories: 安全
---

渗透测试小伙伴的专用修改http request headers chrome 插件用起来已经相当顺手了。
github 源代码如下：https://github.com/chenyoufu/modify-http-headers

首先第一次运行会直接进入到options界面，这里可以为感兴趣的http 头部设置值，而且还可以添加自定义的http 头部。也可以删除默认的http 头部。
可以填写host过滤，只对自己填写的host生效。如果不填写，那么对所有的请求都会生效。

目前默认关注的头部为user-agent，x-forward-for，cookie，referer四个。
只有在setting中勾选了复选框，前端popup页面才会显示当前头部的信息。
popup根据options的设置，对感兴趣的http header的值可以选择浏览器默认，也可以选择自己随便填写。
只有选中custom才可以编辑当前头部的值。
可以即兴在前端更改值，更改后直接会在本地化存储。
对自定义的头部和默认的头部一视同仁，都可以修改，删除。
<!-- more -->
来点截图：
默认第一次进来的时候就是这个样子了。

![](http://7sbqk1.com1.z0.glb.clouddn.com/youfu_blog_mhh0.png)

四个默认头部的值都是空的，这里就可以填入自己搞到的cookie了，如果不想再见到cookie这个头部，那么可以直接点击删除当前头部，那么cookie再也不会来烦你了。

![](http://7sbqk1.com1.z0.glb.clouddn.com/youfu_blog_mhh1.png)

添加一个自定义的头部试一试啊。。。添加一个名字叫做hiyoufu的头部，值也是hiyoufu。

![](http://7sbqk1.com1.z0.glb.clouddn.com/youfu_blog_mhh2.png)

点击一下三文鱼的小图标，看看前端显示什么。。。自定义的头部hiyoufu已经显示出来了，我们再点击一下cookie后的custom，然后输入一个值。

![](http://7sbqk1.com1.z0.glb.clouddn.com/youfu_blog_mhh3.png)

访问一下百度看看能不能生效。。。下图显示生效了，cookie的值变成了刚才输入的Ilovehiyoufu，顺便还增加了一个request的头部hiyoufu，值也是hiyoufu。

![](http://7sbqk1.com1.z0.glb.clouddn.com/youfu_blog_mhh4.png)

所有的修改都是本地持久化存储的，下次再使用就不用麻烦设置了，俗称记忆功能。
