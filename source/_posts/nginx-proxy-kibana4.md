title: nginx 反向代理kopf和kibana4
date: 2016-02-01 16:40:53
tags: elk
---

在公网通过nginx 反向代理 来访问配置在内网的es集群中的es 插件kopf，head以及kibana，marvel。

其中es集群是2.0.0版本，kibana是4.2.0版本。

nginx 配置如下：

```
http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;
    upstream kibana {
        server 10.160.0.49:5601;
    }
    upstream es {
        server 10.160.0.49:9200;
    }

    server {
        listen       8359;
        server_name  localhost;

        location = / {
            proxy_pass http://es;
            valid_referers ~/_plugin/kopf/;
            if ($invalid_referer) {
                return 403;
            }
        }

        location ~ /(app/kibana|bundles|elasticsearch|marvel|app/marvel) {
            proxy_pass http://kibana;
        }

        location ~ /(_plugin|_template|_nodes|_stats|_cluster|_aliases) {
            proxy_pass http://es;
        }
    }
}
```

这样就可以在浏览器中访问内网的es集群中的插件了， 比如kopf

http://nginx_public_ip:8359/_plugin/kopf/#!/cluster

访问kibana:

http://nginx_public_ip:8359/app/kibana

访问marvel:

http://nginx_public_ip:8359/app/marvel

还可以进一步加上简单的用户名密码验证功能。只能说nginx 太强大了。
