---
title: web 站点升级到 tls 1.2 支持
date: 2024-11-18 17:36:51
tags: 
- nginx
- security
- cloudfront
categories: web安全
---
## 前言
之前有出一版是针对 后端服务升级到 tls 1.2 支持的: {% post_link set-tls-12 %}

之前是考虑到客户端那边有些旧设备可能还在用，贸然从服务端那边直接将 tls 1.0 和 tls 1.1 去掉的话，可能会导致原先可以用的设备和客户端，突然就不行了，可能会对用户造成影响。所以对于客户端来说，我们采用了自然过渡的方式，慢慢淘汰掉旧设备，然后新的版本都会开始支持 tls 1.2.

但是对于 web 来说，不需要那么麻烦，早期在站点，不将 站点支持的 tls 1.0 和 tls 1.1 拿掉是因为有些 ie 的浏览器，比如 ie 11以下可能会受影响。 但是自从 2022-06， 微软已经宣布要抛弃掉 ie 系列浏览器了。 所以我们其实可以将一部分的 web 站点，慢慢的去掉 tls 1.0 和 tls 1.1 的支持了。

所以这次就补一个针对 web 站点的升级情况。 

我们的 web 站点部署和请求方式有几种:
1. 直接部署在服务器，然后用 nginx 做托管
2. 直接部署在云服务的云存储 ，然后走 CDN，其中包含 `aws s3 -> cloudfront` 和 `腾讯云 cos -> 腾讯云 cdn`
3. 部署在 aws s3，然后国内单独走加速线路的，比如国内走网宿加速，国外走 cloudfront 加速

接下来讲一下这几种方式

## nginx 方式
nginx 的方式很简单，跟 后端服务升级 是一样的配置，这边不再赘述:
```javascript
ssl_session_timeout         5m;
ssl_protocols               TLSv1.2;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256;
ssl_prefer_server_ciphers   on;
```
<!--more-->
## 第三方 cdn 后台操作
其他几个都是在云服务，要在各自的后台配置

### 1. aws cloudfront 调整为 tls 1.2
cloudfront 很简单，直接找到这一条的记录，然后编辑一下设置，找到安全策略，将原先的 `tlsv1.1_2016` 改成 `tlsv1.2_2018` 即可

![](1.png)

### 2. 腾讯云 cdn 动态加速调整为 tls 1.2
这个也很简单，进入到腾讯云后台，找到 `内容分发网络`, 再找到 `域名管理`, 找到所属域名的 `HTTPS 配置`, 然后将 tls 版本的 `1.0` 和 `1.1` 关掉即可，只保留 `1.2`

![](2.png)

### 3. 网宿加速调整为 tls 1.2
这个也很简单，进入到网宿金融加速后台，找到 `域名管理`, 找到所属域名的 `HTTPS 协议优化`, 然后将 tls 版本的 `1.0` 和 `1.1` 关掉即可，只保留 `1.2`

![](3.png)

## 验证
验证也很简单，一个是用第三方站点进行 ssl 安全验证，比如:
- [ssl labs](https://www.ssllabs.com/ssltest/analyze.html)
- [my ssl](https://myssl.com/)
- [SSL 测试](https://www.sslceshi.com/ssl_check/)

直接看 TLS Protocols 这一栏的支持就行了

还可以直接用 curl 来进行测试，比如站点调整完之后，tls 1.0 和 tls 1.1 是不行的，而 tls 1.2 是可以的，那么结果就是:
```text
curl -I -v --tlsv1.0 --tls-max 1.0 https://test.example.com/
* Host test.example.com:443 was resolved.
....
curl: (35) OpenSSL/3.0.13: error:0A0000BF:SSL routines::no protocols available
```
```text
curl -I -v --tlsv1.1 --tls-max 1.1 https://test.example.com/
* Host test.example.com:443 was resolved.
...
curl: (35) OpenSSL/3.0.13: error:0A0000BF:SSL routines::no protocols available
```

上面两个都要不行， 下面这个是可以的。

```text
curl -I -v --tlsv1.2 --tls-max 1.2 https://test.example.com/
* Host test.example.com:443 was resolved.

...

* TLSv1.2 (OUT), TLS handshake, Client hello (1):
*  CAfile: /etc/ssl/certs/ca-certificates.crt
*  CApath: /etc/ssl/certs
* TLSv1.2 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-AES128-GCM-SHA256 / X25519 / RSASSA-PSS
...
```

这样子就验证修改成功了。





