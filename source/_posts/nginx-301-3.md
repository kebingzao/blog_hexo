---
title: 将A站点代理到B站点的某一个页面，但是其他页面还是在B站点
date: 2019-05-31 11:05:16
tags: nginx
categories: nginx相关
---
## 前言
最近又双叒遇到了一个比较奇怪的需求，就是有一个站点A: we.foo.com , 当用户输入这个站点的时候，要变成官网的登录页：www.foo.com/signup/, 地址栏不能是官网的url，而是要变成， we.foo.com/signup/， 但是其他相关外链的请求，还是要跳到官网去， 只有登录页是变成 we.foo.com。
## 解决
通过 {% post_link nginx-301 %} 我们知道，我们可以通过 nginx 代理转发，将首页的请求都代理转发到官网的登录页面：
```html
location / {
        proxy_pass  https://www.foo.com/signup/;
    }
```
<!--more-->
但是这样一来，其他的页面也会是 we.foo.com 地址开头，而不是官网开头的地址。所以还得在进行一层判断，如果不是 signup 路径的，那么就 301 重定向到官网对应的路径。所以不能直接粗暴的进行代理转发，因为这样子，无论是什么请求，都会全部到官网的登录页面去。而且考虑到nginx的代理转发，其实也是经历过一层中转，效率肯定没有直接取资源文件快。
所以就直接将 we.foo.com 的 nginx 指向 跟 www.foo.com 所在服务器同一台。这样就可以直接取源文件了。 具体 nginx 配置如下：
```html
server {
    server_name  we.foo.com;
    listen       443 ssl http2;
    include      ssl/ssl_foo.com.conf;
    include      conf.d/base.conf;

    access_log   /var/log/nginx/we.foo.com.access.log main;
    access_log   /var/log/nginx/we.foo.com.4xx5xx.log combined if=$loggable;
    error_log    /var/log/nginx/we.foo.com.error.log warn;

    root     /data/wwwroot/www.foo.com;
    index    index.html;

    add_header   strict-transport-security "max-age=31536000";
    add_header   X-Frame-Options "SAMEORIGIN";

    location = / {
        return 301 https://$server_name/signin/;
    }
    location ~* /signin {
            try_files $uri $uri/ @www;
    }
    location / {
            return 301 https://www.foo.com$request_uri;
    }
    location @www {
            return 301 https://www.foo.com$request_uri;
    }
}
```
这边解析试一下，
1. 首先将 root 资源目录指向 www.foo.com 的资源目录
2. 添加两个头部，一个是**strict-transport-security**，即 HSTS，告诉浏览器只能通过 HTTPS 来访问，一个是**X-Frame-Options**，防止被第三方的域名恶意用 iframe 嵌套
3. 接下来就是路由了，这个顺序还不能乱
    1. 如果请求根目录的话，直接 301 重定向到对应的注册页面 signup/
    2. 如果是包含 signup 路径的路由，那么就判断资源文件是否存在，如果存在，就返回资源文件，如果不存在，就到 @www 规则，也就是 301 重定向到官网对应的页面
    3. 如果既不是根目录请求，也不是包含 signup 路径的请求，那么就301 重定向到官网对应的页面
    4. @www 就是定义的一个路由规则，由上面调用

这样就可以实现我们的要求了。几个场景：
1. 如果是直接输入 we.foo.com 那么就会重定向到 we.foo.com/signup/ 页面
2. 如果是输入 we.foo.com/signup/ 就直接这个页面了（因为资源文件存在）
3. 如果是输入 we.foo.com/signupxxxx/ 就重定向到 www.foo.com/signupxxxx/， 然后这个文件官网也没有找到，再次重定向到 www.foo.com/404.html
4. 如果是输入 we.foo.com/download/ 就重定向到 www.foo.com/download/

