---
title: 将 tls 加密级别调整到 tls 1.2 版本
date: 2021-06-21 13:39:46
tags: 
- nginx
- security
categories: web安全
---
## 前言
之前有 researcher 给我们发邮件说我们的一些站点的 tls 加密还支持 tls 1.0 和 tls 1.1 的协议。 这个是很不安全的。 

![1](1.png)

而且会导致站点的评分会被降低， 可以用 [ssltest 站点来测试](https://www.ssllabs.com/ssltest/analyze.html),  而从安全性来评估的话，因为 tls 1.0 和 tls 1.1 的加密方式已经可以被破解了， 所以很容易招到攻击，比如以下攻击方式:
- [BEAST Attack](https://raymii.org/s/tutorials/HTTP_Strict_Transport_Security_for_Apache_NGINX_and_Lighttpd.html)
- [CRIME Attack](https://wiki.mozilla.org/SecurityEngineering/Public_Key_Pinning)
- [Heartbleed](https://raymii.org/s/articles/HTTP_Public_Key_Pinning_Extension_HPKP.html)
- [FREAK Attack](https://www.ssllabs.com/ssltest/)

所以不管是从站点评分上， 还是安全性考虑上， 我们都应该剔除掉 tls 1.0 和 tls 1.1 的加密方式，只采用 tls 1.2 的加密方式。 之所以为啥是 tls 1.2 而不是 tls 1.3， 也是因为 tls 1.2 是最广泛应用的。
<!--more-->
## 具体实操
从项目用到的 tls 加密来看， tls 加密的方式，主要是分为两种，一个是短链接的 https， 一种是长连接的 wss。 接下来具体分析都应该怎么处理。

当然在设置之前， nginx 和 openssl 的版本要检查一下，太低的版本是不行的，比如 openssl 要 1.0.1 及以上的版本才会支持 tls 1.2。

## https 接口的方式
因为我们的项目都是用 nginx 做 web server 的， 所以如果要指定 tls 1.2 版本的话，只需要在各项目的 nginx 的配置加上对应的配置。

比如原来是这样子的, 三种都支持
```text
ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
```

后面要改成只支持 1.2 版本:
```text
ssl_protocols TLSv1.2;
```

但是具体实操，发现没有那么简单，还要配合对应的 cipher suite 才行。 比如我们原先的配置是这样子的:
```text
ssl_session_timeout         5m;
ssl_protocols               TLSv1 TLSv1.1 TLSv1.2;
ssl_ciphers                 EECDH+CHACHA20:EECDH+CHACHA20-draft:EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256:EECDH+3DES:RSA+3DES:!MD5;
ssl_prefer_server_ciphers   on;
```

可以看到当前是支持 1.0 和 1.1 的。 所以后面这个文件要改成

```text
ssl_session_timeout         5m;
ssl_protocols               TLSv1.2;

#支持TLSv1.2 版本的高强度加密
ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA:ECDHE-RSA-AES128-SHA:RSA-AES256-SHA:RSA-AES128-SHA;

ssl_prefer_server_ciphers   on;
```

从配置上来说，确实是只支持 tls 1.2， 但是由于加密算法中 cipher suite 有一些还是属于原来 tls 1.0 或者 tls 1.1 的算法，这就导致虽然我将 `ssl_protocols` 设置为 1.2 了， 但是因为加密算法的配置有一些是 1.0 或者 1.1 支持的加密算法， 也是属于不安全的加密算法，比如这个 `RSA-AES128-SHA`, 所以客户端还是认为你还是支持 1.0 版本。

所以 `ssl_protocols` 要配合 `ssl_ciphers` 两个一起修改，才算正确修改， 所以本例应该修改为:
```text
ssl_session_timeout         5m;
ssl_protocols               TLSv1.2;

#支持TLSv1.2 版本的高强度加密
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;

ssl_prefer_server_ciphers   on;
```
然后一定要测试一下是否真的支持 (后面会讲到怎么测试支持的 tls 版本)。

而且由于这个加密算法其实在不同的平台上是不一样，所以如果用到的话，最好每个客户端平台都要测一下，然后得到适合你们产品的 tls 1.2 所能支持的 cipher suite。 还是以本例来说， 上面的加密算法，其实不支持在 windows 7 的 tls 1.2 加密。 还要再补上这三个加密算法:
```text
DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256
```
所以最后 cipher suite 为:
```text
# 这个是 pc 要求加入的，可以兼容 windows 7 的 tls 1.2
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256;
```

然后在 nginx 改完之后， 记得要 reload 一下 nginx。

## wss 接口的方式
同样是 wss， 但是因为有不同的语言来实现， 这边就单独说。

### golang
我们很多 golang 项目都是走长链接的 wss 服务，这一块配置也比较简单，原来是这样子的:
```text
tlsConfig := &tls.Config{}
tlsConfig.MinVersion = tls.VersionTLS10
tlsConfig.MaxVersion = tls.VersionTLS12
```
后面改成:
```text
tlsConfig := &tls.Config{}
tlsConfig.MinVersion = tls.VersionTLS12
tlsConfig.MaxVersion = tls.VersionTLS12
```
如果想要更详细一点， 想要设置加密套件的话 cipher suite, 可以类似于这样子设置 (不过基本上上面就够用了):
```text
cfg := &tls.Config{
        MinVersion:               tls.VersionTLS12,
        CurvePreferences:         []tls.CurveID{tls.CurveP521, tls.CurveP384, tls.CurveP256},
        PreferServerCipherSuites: true,
        CipherSuites: []uint16{
            tls.TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
            tls.TLS_RSA_WITH_AES_256_GCM_SHA384,
        },
    }
```

### python
还有几个项目是用 python 做长连接的，然后用的是 `tornado.web` 库， 所以直接指定 ssl version 即可:
```text
app.listen(ssl_port_, ssl_options={
        "certfile": crt_,
        "keyfile": key_,
        "ssl_version": ssl.PROTOCOL_TLSv1_2,
    })
```

### 其他
也有用到一些第三方库的，比如 coturn 之类的， 也可以在配置里面设置 (一般配置里面都有):
```text
cipher-list="EECDH+AESGCM:EDH+AESGCM"
ec-curve-name=secp384r1
dh2066
no-tlsv1
no-tlsv1_1
```

## 验证
当我们配置完了， 接下来最主要的就是验证我们配置的对不对， 有两种方式可以验证。

### 1. curl 直接验证
可以用 curl 的指令来验证站点的 tls 是否支持某个版本， 格式如下
```text
curl -I -v --tlsv{version} --tls-max {version} https://{site}:{port}
```
也就是说，我如果要检验某一个服务是否支持 tls 1.0。那么就是:
```text
[kbz@centos156 ~]$ curl -I -v --tlsv1.0 --tls-max 1.0 https://foo.com/
*   Trying xx.xx.xx.xxx:443...
...
...
* Closing connection 0
curl: (35) Cannot communicate securely with peer: no common encryption algorithm(s).
```
如果不支持就是这样子的， 如果是 1.1 版本，那么 curl 指令就是:
```text
curl -I -v --tlsv1.1 --tls-max 1.1 https://foo.com/
```
如果是要测 wss 长连接，一般都要加上端口号，而且协议要改成 https，而不是 wss， 比如:
```text
curl -I -v --tlsv1.1 --tls-max 1.1 https://forward.foo.com:9011
```
如果是支持的协议的话，那么是有回执的，比如支持 tls 1.2 :
```text
[kbz@centos156 ~]$ curl -I -v --tlsv1.2 --tls-max 1.2 https://foo.com/
*   Trying xx.xx.xx.xx:443...
* Connected to foo.com (xx.xx.xx.xx) port 443 (#0)
* Initializing NSS with certpath: sql:/etc/pki/nssdb
*  CAfile: xxx
*  CApath: xxx
* loaded libnssckbi.so
* ALPN, server accepted to use h2
* SSL connection using TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
* Server certificate:
*   subject: CN=*.foo.com,OU=Domain Control Validated
*   start date: Oct 27 05:04:13 2020 GMT
*   expire date: Nov 28 05:04:13 2021 GMT
*   common name: *.foo.com
*   issuer: CN=Go Daddy Secure Certificate Authority - G2,OU=http://certs.godaddy.com/repository/,O="GoDaddy.com, Inc.",L=Scottsdale,ST=Arizona,C=US
* Using HTTP2, server supports multi-use
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* Using Stream ID: 1 (easy handle 0x1809100)
> HEAD / HTTP/2
> Host: foo.com
> user-agent: curl/7.77.0
> accept: */*
> 
* Connection state changed (MAX_CONCURRENT_STREAMS == 128)!
< HTTP/2 200 
HTTP/2 200 
< server: openresty
server: openresty
< date: Fri, 18 Jun 2021 05:37:25 GMT
date: Fri, 18 Jun 2021 05:37:25 GMT
< content-type: text/html; charset=UTF-8
content-type: text/html; charset=UTF-8
< content-length: 562
content-length: 562
< last-modified: Fri, 26 May 2017 05:21:02 GMT
last-modified: Fri, 26 May 2017 05:21:02 GMT
< etag: "5927bb3e-232"
etag: "5927bb3e-232"
< accept-ranges: bytes
accept-ranges: bytes

< 
* Connection #0 to host foo.com left intact
```

所以测试结果就是要 `tls1.0`, `tls1.1`, `tls1.3` 都是失败， 只有 `tls 1.2` 才是成功的，这样子才符合预期。

### 2. tls checker 站点测试
线上也有很多 tls checker 站点，他们也是可以的
- [TLS checker](https://www.cdn77.com/tls-test)
- [SSL Server Test](https://www.ssllabs.com/ssltest/analyze.html)

如果是有端口的(比如 wss)， 后面跟上端口即可。不需要加协议，因为默认都是走 https

![1](2.png)

---

参考文档:
- [How To Configure Nginx to use TLS 1.2 / 1.3 only](https://www.cyberciti.biz/faq/configure-nginx-to-use-only-tls-1-2-and-1-3/)
- [NGINX enable only TLS v1.2](https://serverfault.com/questions/1025568/nginx-enable-only-tls-v1-2)
- [SSL Ciphers](https://curl.se/docs/ssl-ciphers.html)
- [cipherlist](https://syslink.pl/cipherlist/)
- [Strong SSL Security on nginx](https://raymii.org/s/tutorials/Strong_SSL_Security_On_nginx.html#toc_1)







