---
title: 记一次因为 S3 bucket 删除而导致的子域名接管(subdomain takeover)的安全问题
date: 2021-06-21 18:22:44
tags: 
- aws
- s3
- security
categories: web安全
---
## 前言
前段时间有位好心的 researcher 发了一封邮件过来，说我们有一个域名存在 子域名接管(`subdomain tokeover`) 的安全缺陷， 让我们赶紧处理。

还附上了他的 POC 截图

![1](1.png)

后面查了一下，确实这个我们的域名被接管了， 里面的 `index.html` 是 researcher 的。
<!--more-->
## 子域名接管 (subdomain takeover)
子域名接管是指注册一个不存在的域名以获得对另一个域名控制权的过程。此过程最常见的情况如下：

1. 域名(例如 http://sub.example.com )将 CNAME 记录 用于另一个域(例如 http://sub.example.com CNAME http://anotherdomain.com)。
2. 在某个时间点，http://anotherdomain.com 到期，任何人都可以注册。
3. 由于没有从 http://example.com DNS 解析中删除CNAME记录，因此注册 http://anotherdomain.com 域名的任何人都可以完全控制 http://sub.example.com ，直到删除DNS记录为止。

子域接管的含义非常重要。通过使用子域名托管，攻击者可以从合法的域名发送网络钓鱼电子邮件，执行跨站脚本(XSS)或破坏与该域相关联的品牌的声誉。

子域接管不限于CNAME记录,NS，MX甚至A记录(均不受此限制）也将受到影响。

事实上很多提供域名服务的服务或多或少都会存在子域名接管漏洞，比如 Amazon S3， Heroku, Shopify, GitHub

## 排查分析
而本次出现的子域名接管的漏洞，也是属于 CNAME 子域接管，不过不是上述的域名过期，而是域名对应的 bucket 源被我们删除了， 但是因为 aws 创建 bucket 的公开域名是基于地区的，跟用户没关系。所以就算这个 bucket 被删掉了， 其他人就可以在同一个地区创建一个同样的 bucket， 这样子对外的公开域名就变成别人的 bucket 了。所以就指向这个 bucket 的内容。

简单的来说，之所以这个我们的域名会出现被别人接管的情况，主要是因为这个域名所对应的 aws S3 的 bucket 不见了，导致这个 researcher 只需要在他自己的 aws 账号创建一个跟这个域名一样的 bucket 的话， aws 的 router 53 DNS 解析，就会指到这个 researcher bucket， 所以就会出现这个明明是我们的域名，但是内容显示的是别人的文件。

> 当然这种情况只适合 http 协议，如果是 https 协议是不行的， 因为 hacker 没有这个域名的证书，如果是 https 访问就会出现证书问题

所以整个事件的流程是这样子的，首先要知道我们的这个域名的 dns 指向是:
```text
test.foo.com  cname   test.foo.com.s3-website-us-west-1.amazonaws.com
```
其中 `test.foo.com.s3-website-us-west-1.amazonaws.com` 就是这个 `test.foo.com` bucket 所对应的公开访问域名 (事实上只需要将这个 bucket 设置为静态网站托管，就会有一个公开的对外访问域名，不需要额外配置)

![1](2.png)

然后按照以下操作:
1. 将 `test.foo.com` 这个桶删掉。(因为之前要修复这个问题: {% post_link s3-bucket-index %})
> 这时候访问 `http://test.foo.com` 的话， 就会显示无法访问此网站，因为 bucket 源不见了

2. 这时候 hacker 创建一个 aws 账号，然后在他的 s3 上，创建了一个叫做 `test.foo.com` 的 bucket, 选择区域是美国西部 (`us-west-1`), 并且设置为静态网站托管，这时候因为 `test.foo.com.s3-website-us-west-1.amazonaws.com` 没人用了， 所以对外公开域名就是这个。

3. 这时候再访问 `http://test.foo.com` 的话，DNS 路由就会指向 `test.foo.com.s3-website-us-west-1.amazonaws.com` 这个域名， 然后这个域名就会指向 hacker 的这个 bucket 桶 `test.foo.com`， 所以访问成功，但是显示的却是别人的站点。

所以这个就是 S3 出现的因为删除 bucket 目录，但是没有将对应的 DNS 解析记录删除，导致这个子域名被接管的问题。

顺便说一下，Cloudfront 某种情况下，也会有子域名接管的问题，以上例来说，如果这个 bucket 是用 Cloudfront 来进行加速的话，这时候他的 DNS 记录就是:
```text
test.foo.com  cname  d123c12345.cloudfront.net
```
但是 Cloudfront 的回源地址还是这个 bucket ，也就是还是这个公开域名 `test.foo.com.s3-website-us-west-1.amazonaws.com`， 那也是一样会中招。

## 危害性
当然这种子域名接管漏洞危害性还是挺严重的，基本上 CVSS 3.0 的评分都是 high 的级别。 因为别人都接管你的域名了， 都是粘板上的鱼。

## 修复
最后是将这个删除 bucket 的对应的 DNS 记录也一起删除。 才算修复这个问题。

---
参考资料:
- [什么是子域名接管漏洞?](https://zhuanlan.zhihu.com/p/136694063)
- [Subdomain Takeover: Thoughts on Risks](https://0xpatrik.com/subdomain-takeover/)








