---
title: 将带参数的某一个站点代理到另一个站点的某一个页面
date: 2019-05-30 18:06:21
tags: nginx
categories: nginx相关
---
## 前言
通过 {% post_link nginx-301 %} 我们知道可以先建一个全新的域名，然后将其反向代理到另一个站点的某一个页面(地址栏url不能变)。结果现在还有一个更复杂的需求，就是有一个新的域名 foo.at(真实域名换成foo)， 然后后面有带参数，比如  foo.at/112233 ，然后转发代理到另一个站点的另一个页面，比如 www.foo.com/en/action?code=112233 ，其中 code 112233 就是 foo.at 要带的参数。
## 实作一
还是一样用 nginx 反向代理的方式：刚开始是这样子：
<!--more-->
```html
[kbz@centos156 ~]$ cat /etc/nginx/sites/foo.at.conf
server {
    listen 80;
    server_name foo.at;
    return 301 https://foo.at;
}

server {
    listen 443 ssl;
    server_name foo.at;
    include ssl/ssl_foo.at.conf;
    location / {
        proxy_pass   https://www.foo.com/en/action;
    }
}
```
然后测试一下， 输入 https://foo.at 就可以看到有反向代理过去了。页面也显示正确。
这时候输入 https://foo.at/11223 后面加上路径了。 这时候就出问题了， 发现参数没有带过去。所以这时候我们要调整一下，把参数带过去：
```html
[kbz@centos156 ~]$ cat /etc/nginx/sites/foo.at.conf
server {
    listen 80;
    server_name foo.at;
    return 301 https://foo.at;
}

server {
    listen 443 ssl;
    server_name foo.at;
    include ssl/ssl_foo.at.conf;
    location / {
     try_files $uri $uri/ /?$args;
        proxy_pass   https://www.foo.com/en/action;
    }
}
```
加上这一句就会把参数带过去了，所以就可以了。
## 备案的问题
还有一个问题，就是 foo.at 没有在国内的域名备案，所以如果访问 http 的话，就会出现这个页面：
![1](1.png)
所以在备案之前（备案要一个月左右）要用 https 才行。 foo.at 因为在 godaddy 买的，所以不能在国内备案，所以最好可以转到腾讯云。不过后面发现，原来不放国内服务器（之前放的是广东那一台），就不会有备案的问题了。 但是放国外的服务器，又可能出现国内访问国外，解析有问题的情况。所以最好的情况，就是放香港那一台服务器，国内，国外都可以解析。 所以就没有这个问题啦。我真的是个小机灵鬼。
所以就把 foo.at 放到香港那一台服务器了。www.foo.com 还是放原来的服务器。
## 最后调整
最后调整了一下结构为：
```html
server {
	listen 80;
	listen 443 ssl http2;
	server_name foo.at;
	include ssl/ssl_foo.at.conf;

	access_log /var/log/nginx/foo.at.access.log main;
	access_log /var/log/nginx/foo.at.4xx-5xx.log combined if=$loggable;


	if ($scheme = http){
		return 301 https://$server_name$request_uri;
	}
	location / {
		resolver 8.8.8.8;	
		proxy_pass https://www.foo.com/$cookie_lang/action;	
	}
}
```