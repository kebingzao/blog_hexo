---
title: node 使用 https 以及 nginx 端口转发
date: 2019-08-19 13:45:43
tags: 
- nodejs
- https
categories: node相关
---
## 前言
前段时间，项目内部有个工具站点，有运营同学向我反馈，有时候用着用着会出现这个页面：
![1](1.png)
这个页面只会在域名没有在国内备案的时候，用 http 访问的时候，才会出现。但是问题就在于其实我们的域名是有在国内备案的，但是一旦访问的目录层级深一点，比如 5 层，那么就有一定概率还是会出现这个页面。其实有反馈了，但是还是时不时会出现，要解决这种情况，还有三个方式：
1. 将站点放到国外去
2. 访问使用 https
3. 直接使用 ip:port 访问

<!--more-->
将站点放到国外去，其实不太方便，因为是内部项目，所以都是内网访问，数据库也都在内网。如果是使用 ip 访问的话，那么还要加上 node 的端口号（这个是一个node程序），也不方便。所以后面想了一下，还是加 https 最省事了。
## node 接入 https
因为这个程序是用 node 写的，之前只监听了 http，所以无法就是在加上监听 https 而已。代码非常简单：
```javascript
var https = require('https');
// 接下来开启 https
var options = {
    key: fs.readFileSync('./ssl/foo.com.key'),
    cert: fs.readFileSync('./ssl/foo.com.crt')
};
https.createServer(options, app).listen(app.get('httpPort'), function () {
    console.log('Https server listening on port ' + app.get('httpPort'));
});
```
这样子启动的时候，就可以监听 https 了
当然访问的时候，是要加上端口号的，比如： https://foo.com:4556/xxx/xxx 之类的。
## 使用 nginx 端口转发
上面其实已经用 node 实现 https 的监听，但是访问的时候，还要加上端口号，其实很不方便，所以最好的方式就是跟之前 http 的方式一样，对 https 默认的 443 端口进行端口转发：
```html
[kbz@centos156 foo.com]$ cat /etc/nginx/sites-enabled/foo.com.conf
upstream airlang_server {
    server 127.0.0.1:4555 fail_timeout=0;
}
server {
    listen       80;
    listen  443 ssl http2;
    server_name  foo.com;
    include      conf.d/limitip_lan.conf;

    index index.html index.htm index.php;

    access_log  /var/log/nginx/foo.com.access.log;
    error_log   /var/log/nginx/foo.com.error.log;

    location / {
                proxy_pass http://airlang_server;
                proxy_set_header  X-Real-IP  $remote_addr;
        proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```
好吧，配完之后才发现，既然都用 nginx 实现端口转发了，那干嘛还要配置 node 的 https 监听，直接将 nginx 的 80 和 443 的请求端口转发到 node 程序的 http 请求的 4555 端口就好啦。
根本不需要让 node 再去启动 https 服务，只需要启动 http 服务就行了，因为 nginx 会将浏览器请求的 https 请求代理转发到 node 的 http 请求。所以上面的 node 监听 https 其实是多余的，根本不需要，代码注释掉啦， nginx 果然强大 Orn。



