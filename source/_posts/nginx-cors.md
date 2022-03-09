---
title: nginx 配置 CORS
date: 2022-03-01 20:01:16
tags: nginx
categories: nginx相关
---
## 前言
关于 nginx 设置 CORS 跨域的事情，其实在我之前的一些文章都或多或少都有提到，比如:
- {% post_link page-redirect-not-change-url %}
- {% post_link xhr %}

但是都没有比较详细的代码，导致我有时候要写的时候，都要自己查自己写的文章，所以就自己写一个比较通用的范例， 后面用的时候，直接抄就行了。

至于 CORS 原理就不再多说了，网上到处都是。 这边给出的通用范例要符合以下标准
1. `Access-Control-Allow-Origin` 我不会设置为 `*`, 因为除了不安全之外，还会与设置携带 cookie 的这个 `Access-Control-Allow-Credentials` 有冲突
2. `Access-Control-Allow-Origin` 会有一个默认值，但是为了对开发环境更友好，也会有一个白名单的二级域名，只要在这个二级域名下的子域名，也都是可以符合

所以具体配置如下:
<!--more-->
```text
set $origin 'https://www.example.com';
if ($http_origin ~* ".example.com(:[\d]+)?$") {
    set $origin "$http_origin";
}

add_header Access-Control-Allow-Origin $origin;
add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
add_header Access-Control-Allow-Headers 'origin, content-type';
add_header Access-Control-Allow-Credentials 'true';
add_header Access-Control-Max-Age '86400';

if ($request_method = "OPTIONS") {
    return 200;
}
```

同时这边注意几个细节:
1. 对于请求的来源，只要是符合 `*.example.com` 的子域名(还允许最后面带 port 端口号)，那么就可以允许跨域，通过 `$http_origin` 我们可以得到 request 请求的 `origin` 头部
2. 对于 Methods 的允许，我们这边只允许 `GET, POST, OPTIONS` 这 3 个方法，除非你明确有用到 `PUT` 或者 `DELETE`, 否则不要加上去，一切以最小化使用原则
3. 对于 Headers 的允许，如果有自定义头部的话，也要补充到列表中，或者是有使用一些 auth 校验的，比如 http basic auth， 那么也要在这边加上 `Authorization` 这个头部，才能被允许
4. 对于 `Credentials`, 如果默认允许携带 cookie 的话，就为 true， 如果不需要的话，可以设置为 false
5. 对于是否要明确针对 options 请求进行捕获，并返回 200，我有用 chrome 试了一下， 好像没有加也是可以的， 不过为了完整性和更好理解，我这边还是手动将这个 options 的明确 200 返回加上去了。
6. 上面的这个设置可以放到 location 路由外面，这样子可以全局生效，如果只想对某个路由生效，可以单独写在这个路由 location 的代码块里面。
7. 可以加上 `Access-Control-Max-Age` 来对 `OPTIONS` 请求进行缓存，只要请求的 url 一致， 那么第二次浏览器就不会再发送预检请求了。具体可以看 {% post_link use-access-control-max-age %}







