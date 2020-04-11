---
title: 给站点添加 DNS CAA 保护
date: 2020-04-11 14:15:27
tags: security
categories: web安全
---
## 前言
前段时间有位好心的白客有发了一封邮件过来，内容如下：
```text
Hi Team,
    
    This time I found this vulnerability in https://www.example.com/
    
    Vulnerability Type: Missing Certificate Authority Authorization Rule
    
    Description:
    
    Certificate Authority Authorization (supported by LetsEncrypt and other CAs) allows a domain owner to specify which Certificate Authorities should be allowed to issue certificates for the domain. All CAA-compliant certificate authorities should refuse to issue a certificate unless they are the CA of record for the target site. This helps reduce the threat of a bad guy tricking a Certificate Authority into issuing a phony certificate for your site.
    
    The CAA rule is stored as a DNS resource record of type 257. You can view a domain's CAA rule using a DNS lookup service:
    
    
    https://dns.google.com/query?name=example.com&type=A&dnssec=true    
     
     
    Vulnerable Location: https://caatest.co.uk/example.com
     
    ...
```
简单的来说，就是我们的站点并没有添加 DNS CAA 保护。 具有被中间人劫持的风险。 不过确实是这样子，那时候确实并没有为我们的站点添加 CAA 的 DNS 记录，这个是他的站点检测截图：

![png](1.png)
<!--more-->
## 什么是 CAA 记录？
证书颁发机构授权（全称Certificate Authority Authorization，简称CAA）为改善PKI（Public Key Infrastructure：公钥基础设施）生态系统强度，减少证书意外错误发布的风险，通过DNS机制创建CAA资源记录，从而限定了特定域名颁发的证书和CA（证书颁发机构）之间的联系。

譬如说，你的站点已经启用了 [HSTS](https://en.wikipedia.org/wiki/HTTP_Strict_Transport_Security)，甚至已经被[固化到了浏览器内部](https://hstspreload.org/)。但是一个中间人仍然劫持了你的连接。你走了 HTTPS 协议？没问题，我也搞到了一个你所访问域名的 SSL 证书。要知道 SSL 连接使用的证书是服务端决定的，但是这个证书未必就是真正的域名所有人申请的。虽然普通人未必能搞到不属于你的域名的证书，但是证书颁发机构就不一样——虽然基于信誉的原因他们不太可能会这样做，但是没有任何外在的保护防止他们颁发并非由域名所有人申请的证书。

简而言之，一个有效的证书未必就是域名所有人申请的证书。就好比普通人造不出能过验钞机的假钞，但是再强大的验钞机也不能阻止造币厂监守自盗(尤其是合法的造币厂还有好几家)。这时就需要一个更上层的服务去验证这个“有效的证书”的合法性，这就是 DNS CAA 的作用。

[DNS Certification Authority Authorization](https://zh.wikipedia.org/wiki/DNS%E8%AF%81%E4%B9%A6%E9%A2%81%E5%8F%91%E6%9C%BA%E6%9E%84%E6%8E%88%E6%9D%83)（DNS证书颁发机构授权，简称 CAA）是一项借助互联网的域名系统（DNS），使域持有人可以指定允许为其域签发证书的数字证书认证机构（CA）的技术。它会在 DNS 下发 IP 的同时，同时下发一条资源记录，标记该域名下使用的证书必须由某证书颁发机构颁发。

比如我们的站点使用了 godaddy 的证书，我可以同时使用 CAA 技术标记 我们的官网域名 www.xxx.com 使用的 SSL 证书由 godaddy 颁发，这样就可以（在一定程度上）解决上面所述的问题。

## 在 DNS 服务器启用 CAA 记录
首先不是所有的 DNS 服务器都支持 CAA 资源记录，具体可以到这个站点去查: [谁支持 CAA](https://sslmate.com/caa/support), 刚好我们用的 AWS 的 router 53 是有支持的。所以就可以直接在上面配置就行了。

![png](2.png)

## CAA资源记录详解
CAA记录可以控制单域名SSL证书的发行，也可以控制通配符证书。当域名存在CAA记录时，则只允许在记录中列出的CA颁发针对该域名（或子域名）的证书。

在域名解析配置中，咱们可以为整个域（如example.com）或者特定的子域（如subzone.example.com）设置CAA策略。当为整域设置CAA资源记录时，该CAA策略将同时应用于该域名下的任一子域，除非被已设置的子域策略覆盖。

### CAA记录格式
根据规范（[RFC 6844](https://www.rfc-editor.org/rfc/rfc6844.txt)），CAA记录格式由以下元素组成：
```text
CAA <flags> <tag> <value>
```
解释:
- **CAA** -> DNS资源记录类型
- **flags** -> 认证机构限制标志
- **tag** -> 证书属性标签
- **value** -> 证书颁发机构、策略违规报告邮件地址等

#### flags
flags 定义为0~255无符号整型，取值：
```text
Issuer Critical Flag：0

1~7为保留标记
```
#### tag
tag 定义为US-ASCII和0~9，取值：
```text
CA授权任何类型的域名证书（Authorization Entry by Domain） : issue

CA授权通配符域名证书（Authorization Entry by Wildcard Domain） : issuewild

指定CA可报告策略违规（Report incident by IODEF report） : iodef

auth、path和policy为保留标签
```
#### value
value 定义为八位字节序列的二进制编码字符串，一般填写格式为：
```text
[domain] [";" * 参数]
```
### CAA资源记录示例
当需要限制域名 example.com 及其子域名可由机构 letsencrypt 颁发不限类型的证书，同时可由 Comodo 颁发通配符证书，其他一律禁止，并且当违反配置规则时，发送通知邮件到 example@example.com 。

配置如下（为便于理解，二进制Value值已经过转码，下同）：
```text
example.com.  CAA 0 issue "letsencrypt.org"
example.com.  CAA 0 issuewild "comodoca.com"
example.com.  CAA 0 iodef "mailto:example@example.com"
```
如果子域名有单独列出证书颁发要求，例如：
```text
example.com.  CAA 0 issue "letsencrypt.org"
alpha.example.com.  CAA 0 issue "comodoca.com"
```
那么，因子域策略优先，所以只有Comodo可以为域名alpha.example.com.颁发证书。

## 具体实作
知道规则了，那么就很简单了，直接去 router 53 添加一个 CAA 记录, 并且将 CA 指定为 godaddy 就行了:

![png](3.png)

## 检测
可以通过这个站点来检测: [DNS CAA Tester](https://caatest.co.uk/airdroid.com)

![png](4.png)

## 总结
通过这种方式就可以为你的站点更加的安全了。

## 参考
- [给你的站点添加 DNS CAA 保护](https://segmentfault.com/a/1190000011097942)
- [正确设置DNS CAA记录，提升HTTPS站点安全](https://www.trustauth.cn/wiki/24242.html)
- [SSL证书颁发机构将对域名强制CAA检查，到底什么是CAA？CAA记录详解](http://www.cloudxns.net/Support/detail/id/3065.html)




