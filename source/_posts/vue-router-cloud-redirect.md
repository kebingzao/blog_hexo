---
title: vue-router history 路由模式的后端配置
date: 2022-08-17 20:24:47
tags: 
- nginx
- aws
- cos
categories: nginx相关
---
## 前言
前段时间有用 vue3 + vue-router 做了一个前端的单页面应用程序， 那时候为了让路由看起来更美观一点，使用了 vue-router 的 history 模式 (还有一种就是 hash 模式)。
```text
const router = createRouter({
  history: createWebHistory(),
  routes
})
```
虽然 `history`和 `hash` 都是利用浏览器的两种特性实现前端路由，`history` 是利用浏览历史记录栈的API实现，`hash` 是监听 `location` 对象 `hash` 值变化事件来实现。

但是 history 模式因为是直接走 url 的 pathname 中，所以首次访问或者刷新的时候，都会请求到后端的服务器的。 所以后端的服务器是要适配路由的。 

这时候因为是单页面应用程序，所以适配路由其实是全部回到首页，简单的来说，就是在请求路由 404 的情况下， 请求首页 index.html 就行了。
<!--more-->
## nginx 配置
nginx 有一个 try_files 的语法，就是按指定的 file 顺序查找存在的文件，并使用第一个找到的文件进行请求处理。 所以就可以设置为:
```text
location / {
        root /data/test/wwwroot/test.example.com;
        index index.html;
        try_files $uri $uri/ /index.html;
    }
```

其实就是先找 `$uri` 路径的文件， 如果找不到，就去找 `$uri/index.html`, 再找不到，就直接返回 `/index.html`, 让首页去统一解析。

## 腾讯云 COS
正常情况下，静态站点可能要走 CDN 的， 以国内腾讯云的 COS 云存储来说，应该这样子配置:
1. 当找不到文件的时候， 返回 index.html
2. 将 404 错误码转为 200

所以后台配置应该是:

![1](1.png)

## AWS S3 CloudFront
如果是用的是 AWS 的 S3 进行静态资源存储，然后通过 CloudFront 走 CDN 的话，也是差不多配置
1. 在 S3 的路由规则中对 403 规则进行重定向到首页 (对于 S3 来说，如果路径找不到，是返回 403(禁止) 而不是 404)
2. 如果只做第一步的话，就会出现页面是成功返回了，但是状态码还是 403， 所以我们还需要在 CloudFront 中，将 403 转为 200

![1](2.png)

这个 HostName 其实就是站点的域名(`test.example.com`)

![1](3.png)

最后要记得修改 CloudFront 的时候，要在后台刷新 CloudFront 的缓存。

关于 S3 和 CloudFront 的重定向规则，可以参考: {% post_link aws-s3-cloudfront-rule-redirection %}
