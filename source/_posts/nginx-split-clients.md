---
title: nginx 实现前端页面的 A/B 测
date: 2021-12-24 16:02:45
tags: nginx
categories: nginx相关
---
## 前言
前段之前产品有个需求， 想要对一个申请页面的修改进行 A/B 测试, 从而判断出是哪个页面的转化率会比较高。 所以需要前端人员针对原先的 申请页面 进行 A/B 测试。

简单的来说就是: 
```text
用户访问的还是同一个入口: mobile-device-management-free-trial/index.html
然后 A 用户 得到的是旧页面: mobile-device-management-free-trial-old/index.html (这边为了区分，将旧页面的路由改成 old)
然后 B 用户 得到的是新页面: mobile-device-management-free-trial-new/index.html

而且还会有多语言，比如: zh-cn/mobile-device-management-free-trial/index.html
然后 A 用户 得到的是旧页面: zh-cn/mobile-device-management-free-trial-old/index.html
然后 B 用户 得到的是新页面: zh-cn/mobile-device-management-free-trial-new/index.html
```
然后新旧页面的各自流量是一半。

## nginx ngx_http_split_clients_module
nginx 的 `ngx_http_split_clients_module` 可以满足这种需求，可以用来实现流量的分流 (当然也可以实现负载均衡)。 而且该模块是默认就载入的，不需要额外再载入。
<!--more-->
直接拿官方的案例, 具体看 [官方文档](https://nginx.org/en/docs/http/ngx_http_split_clients_module.html)
```text
http {
    split_clients "${remote_addr}AAA" $variant {
                   0.5%               .one;
                   2.0%               .two;
                   *                  "";
    }

    server {
        location / {
            index index${variant}.html;
```

在 nginx中，`split_clients` 执行过程如下：
1. 对设定的变量获取到的值执行 `Murmurhash2` 算法得到 32位整型哈希值，记为hash
2. 32位无符号整型的最大数字`2^32-1`，记为max，也就是最大值
3. 哈希数字与最大数字相除 `hash/max`，可以得到百分比`percent`
4. 配置指令中配置各个百分比范围对应的新变量值
5. 当percent落在配置的范围里时，新变量值就对应赋值给 `$variant`

> 各个百分比相加不能超过100%，* 表示匹配剩余的百分比，百分比可以为小数点后两位的小数

> 因为 `split_clients` 可以构造新的变量，通过这个特性，我们可以把该变量作为内部自定义变量用在很多地方, 本例就是将 ip 地址作为变量来分流，其他的比如 `cookie`, `user-agent` 都可以当做变量

所以本例就是 IP 地址加上 `AAA` 字符串会使用`MurmurHash2` 转换成数字。得出的数字再除以 `2^32-1` ， 得到一个百分比， 当这个百分比在前`0.5%`，那么`$variant`值为`.one`，相应的在`0.5-2.0%`区间的值为`.two`,其他的为空字符。

## 简单的 demo 来一个
```text
[root@VM-0-13-centos conf]# cat nginx.conf

worker_processes  1;

pid /var/run/nginx.pid;

events {
    worker_connections  1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    include             mime.types;
    default_type        application/octet-stream;

    # 根据内置变量变量${remote_addr}进行1:1分发，并将v1和v2的值赋予$version变量
    split_clients "${remote_addr}AAA" $version {
                   50%               v1;
                   *                 v2;
    }
    
    # v1版本服务
    server {
          listen 8081;
          location  / {
              return 200 "v1\n";
          }
    }
    # v2版本服务
    server {
          listen 8082;
          location  / {
              return 200 "v2\n";
          }
    }

    server {
        listen 80;
        location /apply.html {
            proxy_pass http://127.0.0.1/$version;
        }
        # v2版本转发
        location  /v2 {
            proxy_pass http://127.0.0.1:8082/;
        }
        # v1版本转发
        location  /v1 {
            proxy_pass http://127.0.0.1:8081/;
        }
    }
}
```
在配置中，我们利用 `split_clients` 指令对 `$remote_addr` 变量进行 hash 运算，并按 `1:1` 比例随机地将 `$version` 的值赋予 v1 和 v2 ，*表示剩余的比例，即 `1-50%`，这样就可以通过`$version` 的值进行流量分配

我们找两个 ip 地址试一下:
```text
[kbz@VM_16_7_centos ~]$ curl http://159.75.xx.xx/apply.html
v1


[root@VM-0-13-centos conf]# curl http://159.75.xx.xx/apply.html
v2
```

## 实现需求
我们的需求跟上面的 demo 有两个差别:
1. 最后要显示具体 新/旧 的页面的 (服务器上本来就有对应的站点页面了)
2. 我们是有多语言的，所以多语言在流量分配的时候，一定要带过去，不然就会丢掉了

最后的 `nginx.conf` 如下
```text
[root@VM-0-13-centos conf]# cat nginx.conf
worker_processes  1;
pid /var/run/nginx.pid;
events {
    worker_connections  1024;
}
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
    root         /data/server/wwwroot;    
    index        index.html index.htm index.php;
    # 分流，这边指向本地已存在的页面
    split_clients "${remote_addr}AAA" $ver {
        50%               mobile-device-management-free-trial-old/index.html;
        *                 mobile-device-management-free-trial-new/index.html;
    }

    server {
        listen       80;
        server_name  localhost;
        # 因为有多语言，所以这边要正则匹配这个统一的入口
        location ~* mobile-device-management-free-trial/index.html {
            # 并且要从 uri 中，将多语言提取出来，然后转发代理的时候，要带上这个多语言路径
            if ($uri ~* /([A-Za-z0-9_-]+)/mobile-device-management-free-trial/index.html) {
              set $lang $1;
            }
            # 这边转发的时候，要把多语言参数带上，不然就会变成都是没有语种的，而且代理分流的页面是本来就有的，所以也不需要再重设匹配路由，直接走默认的
            proxy_pass http://127.0.0.1/$lang/$ver; 
        }
        
        location / {
           # 如果文件不存在，有可能是该语种没有，所以去掉前一层路由，再重新请求
           if (!-e $request_filename) {
              rewrite ^/([A-Za-z0-9_-]+)/(.*) /$2 permanent;
           }
        }
    }
}
```
接下来测试一下，首先是本机测试:
```text
[root@VM-0-13-centos wwwroot]# curl  "http://127.0.0.1/mobile-device-management-free-trial/index.html"
<!DOCTYPE html>
<html>
<head></head>

<body>
this is page B
</body>
</html>

[root@VM-0-13-centos wwwroot]# curl  "http://127.0.0.1/zh-cn/mobile-device-management-free-trial/index.html"
<!DOCTYPE html>
<html>
<head></head>

<body>
this is page B with zh-cn lang
</body>
</html>
```
> 这个 B 其实就是新页面的内容，如果是 zh-cn 的新页面，那么内容上，就会有 zh-cn 的标记

然后再找另一台服务试试看 (因为是按 ip 地址请求的，所以同一个 ip 肯定是不会变的，所以要另找一台不一样 ip 的服务器):
```text
[kbz@centos156 ~]$ curl "http://159.75.xx.xxx/mobile-device-management-free-trial/index.html"
<!DOCTYPE html>
<html>
<head></head>

<body>
this is page A
</body>
</html>

[kbz@centos156 ~]$ curl "http://159.75.xx.xxx/zh-cn/mobile-device-management-free-trial/index.html"
<!DOCTYPE html>
<html>
<head></head>

<body>
this is page A with zh-cn lang
</body>
</html>
```

如果是 https 的，也是可以的:
```text
[root@VM-0-13-centos ~]# cat /usr/local/nginx/conf/nginx.conf

worker_processes  1;
pid /var/run/nginx.pid;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;

    root         /data/server/wwwroot;    
    index        index.html index.htm index.php;
    
    split_clients "${remote_addr}AAA" $ver {
        50%               mobile-device-management-free-trial-old/index.html;
        *                 mobile-device-management-free-trial-new/index.html;
    }

    server {
        listen       80;
        listen       443 ssl;
        server_name  localhost;
        server_name  test.example.com;
        ssl_certificate             ssl/now/example.com.crt;
        ssl_certificate_key         ssl/now/example.com.key;
        ssl_protocols               TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
        ssl_ciphers                 HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers   on;
        location ~* mobile-device-management-free-trial/index.html {
            if ($uri ~* /([A-Za-z0-9_-]+)/mobile-device-management-free-trial/index.html) {
              set $lang $1;
            }
            # 这个是用来进行域名解析
            resolver 8.8.8.8;
            proxy_pass https://test.example.com/$lang/$ver; 
        }
        
        location / {
           if (!-e $request_filename) {
              rewrite ^/([A-Za-z0-9_-]+)/(.*) /$2 permanent;
           }
        }
    }
}
```

## 线上测试
因为涉及到 ip 地址的测试，所以要有多个不同出口的服务器来测。

这边有一个网址可以测试多个国家的页面访问情况: [geopeeker](https://geopeeker.com/fetch/), 然后我这边的访问结果是

![](1.png)

可以看到有些国家是 A 页面， 有些国家是 B 页面， 比例各 50%， 同一个 url。 所以确实是实现了分流

---

参考资料:
- [Module ngx_http_split_clients_module](https://nginx.org/en/docs/http/ngx_http_split_clients_module.html)
- [Nginx通过split_client实现客户端分流](https://cloud.tencent.com/developer/article/1643133)
- [企业实战｜Nginx多策略流量分发](https://www.cnblogs.com/cheyunhua/p/12771992.html)




