---
title: nginx 通过 301 跳转将旧页面的 seo 权重转移到新页面上
date: 2021-05-18 15:16:42
tags: nginx
categories: nginx相关
---
## 前言
前段时间我们的官网上了新版本， 有将一些对 seo 不友好的旧页面的路由变成了新页面的路由。 比如将 `bizHome.html` 换成 `business/`, 但是其实内容是一样的。

刚开始为了保证旧页面不失效，在构建的时候，是将新页面和旧页面的内容都构建成一样的。 但是这样子一来， seo 的权重就被割裂了。

所以为了保证新页面的 seo 权重能够继承旧页面的权重。就要换成将旧页面的访问全部变成 301 的永久跳转到新页面上。

## nginx 进行 301 跳转
如果要进行 301 永久性跳转的话，是没办法在前端层面上进行的。 只能在服务端上进行， 而我们用的是 webserver 是 nginx， 所以要在 nginx 上面配置。

假设我们这次要调整的页面有以下:
<!--more-->

|旧网址|301跳转到新网址|
|---|---|
|`https://www.foo.com/get.html`|`https://www.foo.com/download/foo-personal/`| 
|`https://www.foo.com/get.html?init=business`|`https://www.foo.com/download/foo-business/`|
|`https://www.foo.com/pricing/?init=biz`|`https://www.foo.com/pricing/foo-business/`|
|`https://www.foo.com/pricing/?init=personal`|`https://www.foo.com/pricing/foo-personal/`|	
|`https://www.foo.com/pricing/`|`https://www.foo.com/pricing/foo-personal/`|
|所有 pt-pt 的网址都要 301 跳转到 pt-br|-|

而且我们的页面都是有小语种，也就是重定向之后，小语种的路由要在，比如 `/en/get.html` 301 跳转之后就是 `/en/download/foo-personal`

## 使用 rewrite 来进行重定向
nginx 设置 301 重定向有好几种， 直接 `return 301 url` 最简单。 不过因为我们这边会涉及到多语言路径也要带过去， 所以会有正则表达式匹配的需求，所以就用 rewrite 来处理。

rewrite 语法：
```text
rewrite regex replacement[flag];
```
rewrite 是实现 URL 重定向的重要指令，他根据regex(正则表达式)来匹配内容跳转到 replacement，结尾是 flag 标记

他可以应用的位置是 `server`, `location`, `if` 这几个内容块里面。 

他的应用 regex 的参数其实是 nginx 的内置变量 `$uri`, 也就是 url 的 path， 不包含传递的参数和锚点。

比如 `https://www.foo.com/a/b/index.html?name=1&age=2#hello` 这个 url，那么应用 regex 的 url 就是 `/a/b/index.html`

举个例子，像我们之前做多语言的时候，如果这个多语言路径不存在的话，那么就会重定向到非多语言的路径去。
```text
if (!-e $request_filename) {
  rewrite ^/([A-Za-z0-9_-]+)/(.*) /$2 permanent;
}
```
这边就会用到这个 rewrite 语法， 具体的表现就是，如果我请求 `https://www.foo.com/ru/download/` 这个页面的时候，如果这个页面不存在， 那么就会 301 重定向到 `https://www.foo.com/download/`

而且最后的这个 `flag` 的值 `permanent` 表示跳转行为为 301 的永久重定向。 `flog` 有以下值:
- `last` : 本条规则匹配完成后继续向下匹配新的location URI规则
- `break`: 本条规则匹配完成后终止，不在匹配任何规则
- `redirect`: 返回302临时重定向
- `permanent`: 返回301永久重定向

## 具体实操
针对上面的这几个旧页面， 处理一下 301 重定向。 因为这边涉及到 location 规则的匹配，具体可以看: {% post_link nginx-location %}

### 1. 首先是针对 get.html 的处理
处理逻辑如下:
```text
if ($request_uri ~* "get.html\\?init=business") {
    rewrite ^/(.*)get.html$ /$1download/foo-business/ permanent;
}
if ($request_uri ~* "get.html") {
    rewrite ^/(.*)get.html$ /$1download/foo-personal/ permanent;
}
```
这边注意一个细节， 重定向的逻辑 都在 if 判断的块里面， 是因为只有这些特殊的页面才需要重定向。 而判断的 `$request_uri` 也是 nginx 的内置变量， 不过这个跟 `$uri` 变量不一样，他还包含后面的请求参数， 只不过他是不能被修改的。

然后会在 rewrite 的正则里面将 多语言的路径提取出来，并放到新的路径里面。

而且替换后的也是 uri 部分， 后面的参数还是会跟原来的。也就是会将 `/en/get.html?name=1` 变成 `/en/download/foo-personal/?name=1`

### 2. 对 pricing 界面的处理
```text
if ($request_uri ~* "/pricing(/)?(index.html)?\\?init=biz") {
    rewrite ^/(.*)(pricing) /$1pricing/foo-business/ permanent;
}

if ($request_uri ~* "/pricing(/)?(index.html)?\\?init=personal") {
    rewrite ^/(.*)(pricing) /$1pricing/foo-personal/ permanent;
}

if ($request_uri ~* "/pricing(/)?(index.html)?$") {
    rewrite ^/(.*)(pricing) /$1pricing/foo-personal/ permanent;
}
```
逻辑跟上面差不多，不过需要注意一点的是，因为这个页面，有时候会省略 `index.html`, 有时候甚至连 `/` 都省略。 所以还要兼容以上两种情况， 所以用了两个 `?` 来做兼容。

### 3. 对 pt-pt 做处理
这个就很简单了，直接瞄准 `pt-pt` 的语言路径:
```text
if ($request_uri ~* "/pt-pt/") {
    rewrite ^/(pt-pt)(.*) /pt-br$2 permanent;
}
```
然后替换成固定的 `pt-br` 即可。

最后替换完之后，要做两步:
1. 执行 `nginx -t` 进行语法检查
2. 执行 `nginx -s reload` 重启 nginx， 让配置文件生效

## 测试
测试的话，很简单，用 curl 测试就行了。 使用参数 `-I`, 然后直接看 location 头部就行了
```text
[root@VM-156-200-centos ~]# curl -I https://www.foo.com/get.html
HTTP/1.1 301 Moved Permanently
...
Location: https://www.foo.com/download/foo-personal/
...

[root@VM-156-200-centos ~]# curl -I https://www.foo.com/zh-cn/get.html
HTTP/1.1 301 Moved Permanently
...
Location: https://www.foo.com/zh-cn/download/foo-personal/
...

[root@VM-156-200-centos ~]# curl -I https://www.foo.com/zh-cn/get.html?name=1
HTTP/1.1 301 Moved Permanently
...
Location: https://www.foo.com/zh-cn/download/foo-personal/?name=1
...

[root@VM-156-200-centos ~]# curl -I https://www.foo.com/zh-cn/get.html?init=business
HTTP/1.1 301 Moved Permanently
...
Location: https://www.foo.com/zh-cn/download/foo-business/?init=business
...

[root@VM-156-200-centos ~]# curl -I https://www.foo.com/pt-pt/
HTTP/1.1 301 Moved Permanently
...
Location: https://www.foo.com/pt-br/
...

[root@VM-156-200-centos ~]# curl -I https://www.foo.com/pricing/
HTTP/1.1 301 Moved Permanently
...
Location: https://www.foo.com/pricing/foo-personal/
...

[root@VM-156-200-centos ~]# curl -I https://www.foo.com/zh-cn/pricing
HTTP/1.1 301 Moved Permanently
...
Location: https://www.foo.com/zh-cn/pricing/foo-personal/
...


[root@VM-156-200-centos ~]# curl -I https://www.foo.com/zh-cn/pricing/index.html?init=biz
HTTP/1.1 301 Moved Permanently
...
Location: https://www.foo.com/zh-cn/pricing/foo-business/?init=biz
...
```

不过这边要注意一个细节，在使用 curl 的时候，如果参数后面有 `&` 要转义，不然会报错, 导致参数传递不全 (直接输入浏览器之所以不用，是因为浏览器帮我们自动转义了)
```text
[root@VM-156-200-centos ~]# curl -I https://www.foo.com/zh-cn/get.html?name=1&age=2
HTTP/1.1 301 Moved Permanently
...
Location: https://www.foo.com/zh-cn/download/foo-personal/?name=1
...
```
就会发现 `&` 后面的参数丢了。

这个是因为在使用 `curl` 的时候，参数的连接符 `&` 是要进行转义的， 所以加上转义符号就可以:
```text
[root@VM-156-200-centos ~]# curl -I https://www.foo.com/zh-cn/get.html?name=1\&age=2
HTTP/1.1 301 Moved Permanently
...
Location: https://www.foo.com/zh-cn/download/foo-personal/?name=1&age=2
...
```










