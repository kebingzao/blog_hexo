---
title: 强制站点使用 https
date: 2019-08-08 17:56:36
tags: 
- nginx
- aws
- 腾讯云
categories: 前端相关
---
## 前言
之前有个站点(foo.com)是同时允许 http 和 https 访问的。 后面策略调整了一下，不允许 http 访问了，只允许 https 访问，如果用户输入了 http 地址，然后就要强制跳转到 https。
## 前端的解决方式
如果啥都不改的话，前端的做法很简单，就是判断协议，如果是 http 协议的话，就重定向到 https 协议：
```javascript
<script type="text/javascript">
  if (location.protocol !== 'https:') {
    // Automatically jump to https
    location.href = location.href.replace('http:', 'https:');
  }
</script>
```
<!--more-->
不过这个有个问题，就是因为前端多了一层跳转，会导致页面的加载速度会更慢。而且还会有安全问题， 因为刚开始是 http 协议，所以有可能会出现 [DNS 劫持](https://baike.baidu.com/item/DNS%E5%8A%AB%E6%8C%81/6739044?fr=aladdin) 的情况。
## web server 的处理方式
所以最好的方式，就是通过 web server 去处理。 刚好我们的这个域名，有测试环境和生产环境，测试环境是用 nginx 做 web server。 而生产环境，又分为国内用腾讯云的 COS 的云存储，国外用 AWS 的 cloudfront 的方式去存储。接下来就分析一下这三种方式是怎么设置的。
![1](1.png)
## nginx 配置
逻辑很简单，如果是 http 80 端口的，就 301 重定向到 https：
```html
[kbz@centos156 verify]$ sudo cat /etc/nginx/foo.com.conf
server {
    server_name foo.com;
    listen 80;
    return 301 https://$server_name$request_uri;

}
server {
    server_name  foo.com;
    listen       443 ssl http2;
    include      ssl/ssl_foo.com.conf;
    include      conf.d/base.conf;

    access_log   /var/log/nginx/foo.com.access.log main;
    error_log    /var/log/nginx/foo.com.error.log warn;

    root         /data/wwwroot/foo.com;
    index        index.html home.html ;

    error_page 404 /404.html;
}
```
## 腾讯云 COS 配置
腾讯云 cos 配置强制 https，不在 cos bucket 里面配置，而是要在 【内容分发网络】 那个服务去配置。找到这个域名，【高级配置】-> 【HTTPS 配置】->【强制跳转HTTPS】:
![1](2.png)
这个需要时间来生效，大概 5 分钟左右。
## AWS cloudfront 配置
cloudfront 的配置也非常简单，直接找到这个 cloudfront 配置，【行为】->【创建行为】->【查看器协议策略】，选择【将 HTTP 重定向到 HTTPS】, 默认是 【HTTP 和 HTTPS】。这样就可以了。
![1](3.png)
这个需要时间来生效，大概 5 分钟左右。至于怎么测试，就跟之前一样，ping cloudfront ，然后得到 ip， 然后在 host 一下。
