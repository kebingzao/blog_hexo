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

---

### 再次更新 -- options 缓存失效的情况 -- 2021-06-19
前段时间在跟进一个服务器性能并发问题的时候，发现这个项目每次在 get 请求的时候， 都会带上 options 的预请求， 导致原本只需要请求一次， 结果每次因为 options 的预请求， 导致每次都要变成请求两次了。 因此也导致服务器那边的并发数就上来了， 刚开始以为是 response 没有带上 `access-control-max-age` 头部， 后面发现是有的:
```text
access-control-max-age: 86400
```
有设置 options 的预请求缓存时间是一天。  那么为啥这个缓存会失效呢？

查看了一下完整的 get 请求:
```text
https://foo.com/user/getinfos?account_id=xxx&_t=2021-06-19T01:54:03.192Z
```
原来每次请求的时候， url 都会带上一个随机值 `_t`, 而这个 `_t` 的随机值就是导致 options 缓存失效的原因。

因为 <font color=red> Access-Control-Max-Age 的设置针对完全一样的url，如果 url 加上路径参数，其中一个url的 Access-Control-Max-Age 设置对另一个url没有效果的 </font>， 必须 url 是一样的， options 请求才会执行缓存效果， 而因为有 `_t` 的随机值存在，导致每次 get 的请求的 url 都是不一样，所以也就导致 options 缓存失效了。

回想了一下，之前之所以在 get 请求加上 `_t` 的随机值，是因为在某些 ie 浏览器上， get 请求如果 url 没有变的话， 会被浏览器本地缓存， 导致结果错误。 所以才在 get 请求后面加上 `_t` 随机值，但是这样子又会导致像 Chrome 或者 Firefox 的 options 缓存失效。

所以后面就修改为，只在 ie 浏览器上的 get 请求才加上 `_t` 随机值，其他浏览器就不加。 果然这样子修改完之后， options 请求就可以被正常的缓存处理了， 相当于并发减少了一半了。













