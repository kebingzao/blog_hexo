---
title: 配置Access-Control-Max-Age让服务端缓存options预检请求
date: 2018-05-05 18:20:44
tags: http
categories: php相关
---
之前在做企业版的管理后台的时候， 因为要做jwt的token校验， 所以前端将utoken作为一个头部项放到 ajax 的 header 头部里面。
因为多了一个自定义头部的原因，这时候前端的请求全部就变成了非简单请求了。
包括get请求和post请求，这样就导致每个请求都要先进行一次options的预请求处理。
导致会增加多余的http请求时间。
<!--more-->
后面查了一下资料：

{% blockquote mozilla https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Access-Control-Max-Age %}
The Access-Control-Max-Age 这个响应首部表示 preflight request  （预检请求）的返回结果
（即 Access-Control-Allow-Methods 和Access-Control-Allow-Headers 提供的信息） 可以被缓存多久。
{% endblockquote %}

---

返回结果可以用于缓存的最长时间，单位是秒。
在Firefox中，上限是24小时 （即86400秒）
而在Chromium 中则是10分钟（即600秒）。Chromium 同时规定了一个默认值 5 秒。
<font color=red>如果值为 -1，则表示禁用缓存，每一次请求都需要提供预检请求，即用OPTIONS请求进行检测。</font>

所以后面我们就针对这些有jwt校验的请求添加了这个头部：
{% codeblock lang:php %}
header('Access-Control-Max-Age: 86400');
{% endcodeblock %}

这样就会缓存预检头部了，可以减少http的连接时间。