---
title: 记一次 nginx 转发代理 https 出现 502 的情况
date: 2022-08-16 20:35:49
tags: nginx
categories: nginx相关
---
## 前言
前段时间我们官网要做一个活动页面， 但是活动页面是用另一个活动页域名， activity.example.com, 但是运营人员需要对外展示的落地页是以官网 www 的域名来处理，所以这时候就会需要在官网的 nginx 指向那边进行页面的代理转发:
```sql
location  /promo/student-discount {
    resolver 8.8.8.8;
    proxy_pass https://activity.example.com/promo/student-discount;
}
```

但是实测的过程中， 却发现代理转发的时候，报了一个 502 的错误
```sql
2022/08/16 11:58:22 [error] 2293#0: *213338285 SSL_do_handshake() failed (SSL: error:1408F10B:SSL routines:ssl3_get_record:wrong version number) while SSL handshaking to upstream, client: 14.xxx.1.86, server: www.example.com, request: "HEAD /promo/student-discount HTTP/1.1", upstream: "https://13.xxx.xxx.101:443/promo/student-discount", host: "www.example.com"
2022/08/16 11:58:22 [warn] 2293#0: *213338285 upstream server temporarily disabled while SSL handshaking to upstream, client: 14.xxx.1.86, server: www.example.com, request: "HEAD /promo/student-discount HTTP/1.1", upstream: "https://13.xxx.125.101:443/promo/student-discount", host: "www.example.com"
```

看了一下，应该是 nginx 在进行代理请求的时候，就报错了， 应该是 ssl 的握手的错误 `SSL_do_handshake()`
<!--more-->
## 分析
因为我这个活动页程序是部署在 aws 的 s3 上的，并且使用 cloudfront 来做 cdn。 所以去查了一下 cloudfront 的文档，确实有对应的文档:

{% blockquote AmazonCloudFront https://docs.aws.amazon.com/zh_cn/AmazonCloudFront/latest/DeveloperGuide/http-502-bad-gateway.html %}
如果您使用自定义源并将 CloudFront 配置为要求 CloudFront 与源之间使用 HTTPS，则问题可能在于域名不匹配。在源上安装的 SSL/TLS 证书的公用名字段中包含一个域名，使用者备用名称字段中可能包含更多域名。（CloudFront 支持证书域名中的通配符。） 证书中必须有一个域名与下面的一个或两个值匹配：

您为分配中适用源的源域名指定的值。
- 如果您将 CloudFront 配置为将 Host 标头转发到您的源，则为该 Host 标头的值。有关将 Host 标头转发到源的更多信息，请参阅根据请求标头缓存内容。
- 如果域名不匹配，SSL/TLS 握手将失败，CloudFront 将返回 HTTP 状态代码 502（无效网关）并将 X-Cache 标头设置为 Error from cloudfront。

要确定证书中的域名是否与分配或 Host 标头中的 Origin Domain Name 匹配，可以使用在线 SSL 检查程序或 OpenSSL。
{% endblockquote %}

他的文档中也有提供用 openssl 的测试方式:

{% blockquote AmazonCloudFront https://docs.aws.amazon.com/zh_cn/AmazonCloudFront/latest/DeveloperGuide/http-502-bad-gateway.html %}
要帮助纠正来自 CloudFront 的 HTTP 502 错误，您可以使用 OpenSSL 尝试与源服务器建立 SSL/TLS 连接。如果 OpenSSL 无法建立连接，则可能表明源服务器的 SSL/TLS 配置出错。如果 OpenSSL 能够建立连接，它将返回有关源服务器证书的信息，包括证书的公用名称（Subject CN 字段）和使用者备用名称（Subject Alternative Name 字段）。

使用以下 OpenSSL 命令测试与源服务器的连接（将源域名 替换为源服务器的域名，如 example.com）：

`openssl s_client -connect origin domain name:443`

如果满足以下条件：
1. 您的源服务器支持具有多个 SSL/TLS 证书的多个域名
2. 您的分配已配置为将 Host 标头转发到源

然后，将 -servername 选项添加到 OpenSSL 命令中，如以下示例所示（将 CNAME 替换为分配中配置的 CNAME）：

`openssl s_client -connect origin domain name:443 -servername CNAME`
{% endblockquote %}

而我们的官网的原服务器，其实是有存在多个 tls 证书的域名的，比如:
- www.example.com
- example.at
- example.info

所以确实符合上述的第一点条件。

## 使用 openssl 实验

既然是 ssl 的握手错误，那就可以用 openssl 尝试握手一下:
```sql
[kbz@VM-16-9-centos ~]$ openssl version
OpenSSL 1.0.2k-fips  26 Jan 2017


[kbz@VM-16-9-centos ~]$ openssl s_client -connect activity.example.com:443
CONNECTED(00000003)
139703956457360:error:140770FC:SSL routines:SSL23_GET_SERVER_HELLO:unknown protocol:s23_clnt.c:794:
---
no peer certificate available
---
No client certificate CA names sent
---
SSL handshake has read 7 bytes and written 289 bytes
---
New, (NONE), Cipher is (NONE)
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
SSL-Session:
    Protocol  : TLSv1.2
    Cipher    : 0000
    Session-ID: 
    Session-ID-ctx: 
    Master-Key: 
    Key-Arg   : None
    Krb5 Principal: None
    PSK identity: None
    PSK identity hint: None
    Start Time: 1660726052
    Timeout   : 300 (sec)
    Verify return code: 0 (ok)
---
```

发现确实连证书都不给返回。 跟文章说的效果一样， 接下来我们加上 `-servername`:
```sql
[kbz@VM-16-9-centos ~]$ openssl s_client -connect activity.example.com:443 -servername activity.example.com
CONNECTED(00000003)
depth=3 C = US, O = "The Go Daddy Group, Inc.", OU = Go Daddy Class 2 Certification Authority
verify return:1
depth=2 C = US, ST = Arizona, L = Scottsdale, O = "GoDaddy.com, Inc.", CN = Go Daddy Root Certificate Authority - G2
verify return:1
depth=1 C = US, ST = Arizona, L = Scottsdale, O = "GoDaddy.com, Inc.", OU = http://certs.godaddy.com/repository/, CN = Go Daddy Secure Certificate Authority - G2
verify return:1
depth=0 CN = *.example.com
verify return:1
---
Certificate chain
 0 s:/CN=*.example.com
   i:/C=US/ST=Arizona/L=Scottsdale/O=GoDaddy.com, Inc./OU=http://certs.godaddy.com/repository//CN=Go Daddy Secure Certificate Authority - G2
 1 s:/C=US/ST=Arizona/L=Scottsdale/O=GoDaddy.com, Inc./OU=http://certs.godaddy.com/repository//CN=Go Daddy Secure Certificate Authority - G2
   i:/C=US/ST=Arizona/L=Scottsdale/O=GoDaddy.com, Inc./CN=Go Daddy Root Certificate Authority - G2
 2 s:/C=US/ST=Arizona/L=Scottsdale/O=GoDaddy.com, Inc./CN=Go Daddy Root Certificate Authority - G2
   i:/C=US/O=The Go Daddy Group, Inc./OU=Go Daddy Class 2 Certification Authority
 3 s:/C=US/O=The Go Daddy Group, Inc./OU=Go Daddy Class 2 Certification Authority
   i:/C=US/O=The Go Daddy Group, Inc./OU=Go Daddy Class 2 Certification Authority
---
Server certificate
-----BEGIN CERTIFICATE-----
```

发现确实是可以了。

## 解决
既然知道问题了，其实就很好解决，就是在 nginx 转发代理的时候，加上 `proxy_ssl_server_name on`
```sql
Syntax:	proxy_ssl_server_name on | off;
Default:	
proxy_ssl_server_name off;
Context:	http, server, location
This directive appeared in version 1.7.0.
```
> Enables or disables passing of the server name through TLS Server Name Indication extension (SNI, RFC 6066) when establishing a connection with the proxied HTTPS server.

事实上，只要是需要针对 SNI 返回 server name 的后端服务，都会需要 nginx 代理转发的时候携带这个配置。 所以最后改成:
```text
location  /promo/student-discount {
    resolver 8.8.8.8;
    proxy_ssl_server_name on;
    proxy_pass https://activity.example.com/promo/student-discount;
}
```

这样子就正常了。

---

参考资料:
- [CloudFront 与自定义源服务器之间的 SSL/TLS 协商失败](https://docs.aws.amazon.com/zh_cn/AmazonCloudFront/latest/DeveloperGuide/http-502-bad-gateway.html)
- [nginx反向代理https访问502](https://www.jianshu.com/p/999ac06e3934)
- [proxy_ssl_server_name](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_ssl_server_name)


