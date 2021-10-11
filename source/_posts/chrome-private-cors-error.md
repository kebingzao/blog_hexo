---
title: chrome 94 之后 http 远程站点请求 ip 地址报 CORS 错误
date: 2021-10-11 11:55:24
tags: Chrome 
categories: 前端相关
---
## 前言
前段时间，有用户反馈 chrome 最新版本，无法在我们的远程站点连接本地的设备。 后面查了一下， 发现控制台有报了这个错误:
```text
Access to script at 'http://192.168.197.81:8888/sdctl/comm/ping/?product=intl&des=1&callback=_jqjsp&_1633681289908=' from origin 'http://example.com' has been blocked by CORS policy: The request client is not a secure context and the resource is in more-private address space `private`.
```
而且有些用户的 chrome 更奇葩， 请求 google 机器人校验的验证码的 api.js 也报了这个错:
```text
index.html:1 Access to script at 'https://www.recaptcha.net/recaptcha/api.js' from origin 'http://example.com' has been blocked by CORS policy: The request client is not a secure context and the resource is in more-private address space `local`.
```

## 排查
后面查了一下， 是 chrome 新版本 94 的问题， 他不允许线上远端的站点请求 本地local 的地址，比如 ip 地址，所以会报 CORS 错误。 

但是为啥我们的一些 chrome 94 没有问题， 猜测应该是 A/B 测的原因。
<!--more-->

https://developer.chrome.com/blog/private-network-access-update/  这个是他们的更新 log， 他们认为 远程站点请求 local 资源是不安全的， 有过多次被 CSRF 攻击的情况， 所以要禁止这种行为，才会报这个 CORS 的错。

![](3.png)

而且在 2022 年 5 月：Chrome 102 推出稳定版。弃用试验结束。Chrome 会阻止来自公共、非安全上下文的所有私有网络请求。 到时候， http 协议下的远程站点就真的无法访问本地私有专用网络了。

> 同时我在我的本地地址试了一下， 发现不会有这个问题。 说明确实是远程站点才会有。 本地 host 不受影响，直接在导航栏请求 ip 地址也正常。

## 通过修改 chrome 配置来避免
后面通过查了一下 [Chrome CORS error on request to localhost dev server from remote site](https://stackoverflow.com/questions/66534759/chrome-cors-error-on-request-to-localhost-dev-server-from-remote-site), 发现要解决的话， chrome 有个配置可以解决。
1. 打开 tab 页面 `chrome://flags/#block-insecure-private-network-requests`
2. 将其 `Block insecure private network requests` 设置为 `Disabled`, 然后重启就行了， 这样子就相当于把这个功能禁用掉

![](6.png)

这样子就相当于禁用了。 效果是可以的，也就不会再报这个错误了。

## 使用 deprecation trial
虽然通过修改 chrome 配置项是可以的， 但是这样子毕竟对用户不太友好，而且也不是每个用户遇到问题都会反馈。 所以还是要有一个解决方案的。

这一篇 blog: [Private Network Access update: Introducing a deprecation trial](https://developer.chrome.com/blog/private-network-access-update/) 有给了三种途径解决。

{% blockquote Accessing private IP addresses https://developer.chrome.com/blog/private-network-access-update/ %}
Accessing private IP addresses
If your website needs to issue requests to a target server on a private IP address, then simply upgrading the initiator website to HTTPS does not work. Mixed Content prevents secure contexts from making requests over plaintext HTTP, so the newly-secured website will still find itself unable to make the requests. There are a few ways to solve this issue:

1. Upgrade both ends to HTTPS.
2. Use WebTransport to securely connect to the target server.
3. Reverse the embedding relationship.
{% endblockquote %}

不过他给的解决方案其实就是要升级为 https。 但是我们的 local server 是 ip 地址， 如果远程站点升级为 https， 那么将无法访问这个 ip 地址了。 而如果将这个 ip 地址也自定义一个 ssl 证书的话，又需要首次访问的时候， 提醒用户去设置允许， 所以只能保持用 http 协议。

同时有给了我们一种现阶段的兼容方式 `deprecation trial`， 可以一直用到 `2022-05-17` 。

{% blockquote Register for the deprecation trial https://developer.chrome.com/blog/private-network-access-update/ %}
Register for the deprecation trial
First, register for the "Private Network Access from non-secure contexts" trial using the web developers console, and obtain a trial token for each affected origin. Then configure your web servers to attach the origin-specific Origin-Trial: $token header on responses. Note that this header need only be set on main resource and navigation responses, and then only when the resulting document will make use of the deprecated feature. It is useless (though harmless) to attach this header to subresource responses.

Since this trial must be enabled or disabled before a document is allowed to make any requests, it cannot be enabled through a <meta> tag. Such tags are only parsed from the response body after subresource requests might have been issued. This presents a challenge for websites not in control of response headers, such as github.io static websites served by a third party.
{% endblockquote %}

其实就是去 chrome 的 develop 后台，针对这个功能申请一个 token， 然后再这个 web server 的回执中将这个 token 作为 `Origin-Trial` 头部返回。
> 这边要注意，这边的 web server 是你的远程站点的 web server， 而不是你请求的那个 local 资源的 web server。 这个坑我也踩过

正常有两种方式来用这个 token

![](7.png)

1. 当做 web server 的 response `Origin-Trial` 头部返回
2. 设置成 web server 的 meta 标签

不过 blog 文章里面说， meta 标签不适于本次这个例子。 因此只能用 response 头部返回的情况
> 刚开始我还不信邪， 还真尝试了用 meta 标签的方式， 最后证明文章说的没错，确实不适用于本例

### 申请 token
所以就去这边申请 [chrome origin trials](https://developer.chrome.com/origintrials/#/trials/active)

![](8.png)

找到这个项

![](9.png)

然后点击进去， 点击注册 按钮

![](10.png)

可以看到这种兼容模式， 只能用到 `2022-05-17`, 也就是这个 chrome 101 版本出来之后，就没有用了。

然后填一下， 这时候就注册好了。

![](11.png)

这边注意几个点
1. 这个 token 是有有效期的， 看起来应该是 55 天的有效期， 快到期的时候，要点击 renew 按钮续期
2. renew 按钮，刚申请的时候，是点击不了的。

后面发现 有个 三角形的 感叹号。 我怀疑第一次要生效的话， 得先 feedback 一下才行。 并且 renew 按钮是点不了的。

所以就点击 feedback 按钮， 进行了填写。 果然填写了之后， 感叹号不见了。 renew 按钮也可以点击了。

![](12.png)

接下来我们就可以用这个 token 了。

### 在远程站点的 response 中返回 Origin-Trial 头部
因为这个站点， 国内有用 网宿的加速， 国外用的是 s3 的 cloudfront cdn。 所以要配两个地方。 之前添加 `x-frame-options` 头部的时候，就处理过了。 具体看: {% post_link  cloudfront-add-x-frame-options %}

#### 网宿添加头部

![](13.png)

在网宿后台这样子配， 然后测试一下。

```text
[kbz@centos156 logs]$ curl -I http://example.com/index.html
HTTP/1.1 200 OK
Date: Mon, 11 Oct 2021 03:28:08 GMT
Content-Type: text/html
Content-Length: 7065
Connection: keep-alive
x-amz-id-2: InmgJo+NreFhy3ipzrvWRxNXTalwg0eRgXVSS35TJbfzkxFQsfb7CBVQJv4klP2Pw+j0b8ZC1Aw=
x-amz-request-id: 5NGX6B1PWYF6HW04
Last-Modified: Tue, 28 Sep 2021 11:34:17 GMT
ETag: "e42f39a00a7eaa61f93ba5b582c72d28"
Server: AmazonS3
X-Via: 1.1 jfzhdx97:4 (Cdn Cache Server V2.0), 1.1 PS-FOC-01iNb26:10 (Cdn Cache Server V2.0)
X-Ws-Request-Id: 6163af48_PS-FOC-01wFw27_14483-32709
X-Frame-Options: SAMEORIGIN
Origin-Trial: AlzXLEBW+....ZCIsImV4cGlyeSI6MTY1Mjc3NDQwMH0=
```


#### cloudfront 添加头部
还是在 添加 `x-frame-options` 头部的时候， 直接添加一行， 然后重新部署一个新版本，最后应用到对应的 cloudfront 就行了

![](14.png)

然后测试一下:
```text
[kbz@gosrv-other ~]$ curl -I http://example.com/index.html
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 7065
Connection: keep-alive
Last-Modified: Tue, 28 Sep 2021 11:34:17 GMT
Accept-Ranges: bytes
Server: AmazonS3
X-Frame-Options: SAMEORIGIN
Origin-Trial: AlzXLEBW+...yeSI6MTY1Mjc3NDQwMH0=
X-Edge-Origin-Shield-Skipped: 0
Date: Mon, 11 Oct 2021 03:29:00 GMT
ETag: "e42f39a00a7eaa61f93ba5b582c72d28"
X-Cache: RefreshHit from cloudfront
Via: 1.1 7a21e9c0eca084f9537ebb23906ea9ff.cloudfront.net (CloudFront)
X-Amz-Cf-Pop: SFO20-C1
X-Amz-Cf-Id: OTY0Aq5ejaUsl5-ZdZIfbEltgzwPEu6QM-eizShk-Jvg76RUYzfDVw==
```

说明都正常。

这时候 chrome 访问的时候， 终于正常了， 不会再报错误了。

## 后续注意续期
在 `2021-12-03` 之前，要对这个 token 进行续期。就是点击 renew 按钮。

## 真正的解决方案
到明年的 `2022-05-17` 这个临时方案也不行了。 那么有没有可以解决的方式??

从 chrome 的策略来看， http 下访问私有专用网络， 大方向应该还是不行的。 因为这个东西确实不安全， 所以报了多个 CSRF 的安全漏洞。 所以能升 https， 还是要升级到 https。

如果到那时候，还想要在 http 协议下访问私有专用网络，那么就只能启用他的 policies， 具体可以看 [Enable policies](https://developer.chrome.com/blog/private-network-access-update/#policies):
1. [InsecurePrivateNetworkRequestsAllowed](https://chromeenterprise.google/policies/#InsecurePrivateNetworkRequestsAllowed)
2. [InsecurePrivateNetworkRequestsAllowedForUrls](https://chromeenterprise.google/policies/#InsecurePrivateNetworkRequestsAllowedForUrls)

其实就是要将这个私有网络设置为 chrome 这个用户的访问白名单， 以 windows 为例，就是要这样子设置:
```text
Software\Policies\Google\Chrome\InsecurePrivateNetworkRequestsAllowedForUrls\1 = http://www.example.com:8080
Software\Policies\Google\Chrome\InsecurePrivateNetworkRequestsAllowedForUrls\2 = [*.]example.edu
```

但是这样子不现实。 所以可能在 `2022-05-17` 之后， http 协议的远程站点，就不能再请求本地的专用网络了。包括 ip 形式的， localhost 形式的。

---

## 参考资料
- [Origin Trials Guide for Web Developers](https://github.com/GoogleChrome/OriginTrials/blob/gh-pages/developer-guide.md)
- [Private Network Access update: Introducing a deprecation trial](https://developer.chrome.com/blog/private-network-access-update/)
- [How to register for an origin trial](https://developer.chrome.com/blog/origin-trials/)
- [Private Network Access](https://wicg.github.io/private-network-access/)
- [Chrome CORS error on request to localhost dev server from remote site](https://stackoverflow.com/questions/66534759/chrome-cors-error-on-request-to-localhost-dev-server-from-remote-site)








