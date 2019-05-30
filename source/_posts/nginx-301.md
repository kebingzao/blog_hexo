---
title: 将某一个站点代理到另一个站点的某一个页面
date: 2019-05-30 17:44:48
tags: nginx
categories: nginx相关
---
## 前言
之前有个需求，就是要做一个专门的页面用来跳转到我们的一个app的 ios 的 test flight 的页面，而且要单独一个域名，比如 ios.example.com 这种域名。
## 实作
页面很简单，就建立在官网的那个项目上，代码如下： testFlight.html
<!--more-->
```html
<!DOCTYPE html>
<html>
<head lang="en">
    <meta charset="UTF-8">
    <title>IOS Test Flight</title>
    <script>
        window.location.href = "https://testflight.apple.com/join/NIxxxxx"
    </script>
</head>
<body>

</body>
</html>
```
既然页面写好了，再把这个站点(ios.example.com)给配置好, 然后再在这个站点所在服务器的 nginx 的配置文件上，将指向代理到官网的这个页面：
```html
[kbz@centos156 sites]$ cat ios.example.com.conf
server {
    listen 80;
    server_name  ios.example.com;
    return 301 https://ios.example.com;
}

server {
    server_name ios.example.com;
    listen 443 ssl http2;
    include ssl/ssl_example.com.conf;
       
    location / {
        proxy_pass   https://www.example.com/testFlight.html;
    }
}
```
然后默认也走 https, 然后就是通过 nginx 代理转发到 官网的这个页面就行了 https://www.example.com/testFlight.html。
这样子其实浏览器地址栏，会经历三层跳转： 
```html
ios.example.com -> www.example.com/testFlight.html -> testflight.apple.com/join/NIxxxxx
```

