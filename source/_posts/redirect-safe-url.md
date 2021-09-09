---
title: web 安全之 - 登录之后的重定向 url 要有白名单机制
date: 2021-09-09 15:02:45
tags: 
- security
categories: web安全
---
## 前言
前段时间有一个好心的白帽子给我们反馈了一个安全问题，就是官网登录之后的重定向 url 存在安全隐患，导致有可能会泄露用户的 oauth token。

具体是这样子操作的, 因为我们的好多产品线，都是同一个地方做单点登录的(其实就是在官网), 然后登录之后，会根据不同的产品，然后在登录校验成功之后，返回对应产品线的 oauth token。 并拼接到 redirect url 后面。

正常情况下， 我们当然会在登录成功之后， 校验 redirect url 的合理性。 但是这个校验是有漏洞的:
```text
/^http[s]?:\/\/.+\.example.com/ig.test(redirectUrl)
```
因为只判断域名是否有包含 `.example.com` 域名。并没有去判断一定是要 `.example.com` 域名结尾的。 导致最后被 `biz.example.com.com` 这种冒充的域名给通过了。

所以我如果是黑客的话， 我只要搞一个类似的 `biz.example.com.com` 的域名， 然后将这个我构建出来的登录页面 url 发送给受害者，让受害者去登录， 那么这个 oauth token 就会带到我的站点上来了。

```text
https://my.example.com/en/signin/?redirect=https%3A%2F%2Fbiz.example.com.com
```
<!--more-->
## 优化
所以这个 正则表达式要改一下， 域名必须是 `.example.com` 结尾的。 后面要么什么都不带，要么后一个字符就只能带 `/` 或者 `:`, 前者是允许跟路径， 后置是允许带端口号。

所以后面就抽成一个公共方法:

```text
isSafeUrl: function(url) {
    if (!url) return false;

    url = decodeURIComponent(url);
    var has_protocol = /http[s]?:\/\//i.test(url);
    if (has_protocol) {
        return /^https?:\/\/[^/?#]*\.example\.com([\/:].*)?$/i.test(url);
    } else {
        // 以 / 开头
        return /^\//g.test(url);
    }
},
```

这样子就不会被绕过去了。

