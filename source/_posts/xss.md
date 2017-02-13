title: 挖了一个xss小洞
date: 2017-02-13 17:16:02
tags: xss
categories: web安全
---

距离上次挖xss漏洞已经1年多了，新的一年又开始了，去招聘网站上逛了逛，顺手挖到了一个国内著名招聘网站的xss小洞。

从alert的poc验证到可以真正利用加载外部js花了不少的时间，不过还是没有绕过chrome的反射型xss检测，但是在firefox下是没问题的。

首先说说遇到的几个坑：

1. https的环境下如果遇到http的外部请求，firefox默认会屏蔽掉，所以xss的利用平台需要支持https访问。
2. 目前市面上公开的xss 测试平台基本都是http的，所以需要自己搭建一个https的xss利用平台。
3. 输入框有长度限制，后来发现只是在web前端做的字符限制，后台没有做进一步检查。
4. 绕过了一些网站后台的过滤，比如\/等好多特殊字符都过滤掉了，得对利用代码进行编码绕过。

主要记录一下在ubuntu 14.04上如何从0搭建一个https的xss利用平台。


1. 安装openresty和php5以及php5-fpm，其中openresty安装在/opt下面。
新建/opt/xsstest目录，这个目录作为xsstest的专门目录
```
cd /opt/xsstest
cp -r /opt/openresty/nginx/conf .
cp -r /opt/openresty/nginx/html .
mkdir logs
```
修改nginx的配置文件, 大部分都是采用默认配置即可
```
worker_processes auto;
user  www-data;
daemon off;

events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    keepalive_timeout  65;

    server {
        listen       80;
        server_name  localhost;
        root           bxss;

        location / {
            index  index.html index.htm;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
        }

        location ~ \.php$ {
            fastcgi_pass unix:/var/run/php5-fpm.sock;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            include        fastcgi_params;
        }
    }

    # HTTPS server

    server {
        root           bxss;
        listen       443 ssl;
        server_name  localhost;

        ssl_certificate     /opt/xsstest/ssl/ip.crt;
        ssl_certificate_key /opt/xsstest/ssl/ip.key;

        ssl_session_cache    shared:SSL:1m;
        ssl_session_timeout  5m;

        ssl_ciphers  HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers  on;

        location / {
            index  index.html index.htm;
        }
        location ~ \.php$ {
            fastcgi_pass unix:/var/run/php5-fpm.sock;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            include        fastcgi_params;
        }
    }
}
```
2. 选择一个xss平台，这里推荐https://github.com/firesunCN/BlueLotus_XSSReceiver
这个平台不需要mysql相关的依赖，数据是保存在本地的，自己用很好。
git clone 下来之后重命名为 bxss，放到 /opt/xsstest/ 目录下。并且按照帮助文档做相关的配置，之后修改整个目录的权限为777.

3. 生成ssl证书。
http://www.liaoxuefeng.com/article/0014189023237367e8d42829de24b6eaf893ca47df4fb5e000
或者在xsstest目录下直接下载脚本
```
wget https://raw.githubusercontent.com/michaelliao/itranswarp.js/master/conf/ssl/gencert.sh
```
然后运行这个脚本就可以了。如果没有域名就在输入域名的时候直接输入外部ip地址。创建完成证书后根据提示将crt和key放到xsstest/ssl/ 目录下。

4. 安装supervisorctl来控制nginx服务器的状态。配置文件如下：

```
[program:xsstest]
startsecs = 3
autostart = true
autorestart = true
startretries = 3
command = /opt/openresty/nginx/sbin/nginx -p /opt/xsstest -c conf/nginx.conf
```

这个样子配置好的还有一个问题，就是里面的default.js会默认加一个/，这个问题可以修改nginx的配置文件路径，也可以直接把xss平台的js脚本修改掉就可以成功接受。
