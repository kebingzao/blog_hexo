---
title: web 安全之 - 开启HSTS让浏览器强制跳转HTTPS访问
date: 2020-05-25 10:55:57
tags: security
categories: web安全
---
## 前言
前段时间一位好心的白帽子发了一封邮件过来:
```html
Hi team,
    I found another potential bug on your site.
    Description
    The application fails to prevent users from connecting to it over unencrypted connections. An attacker able to modify a legitimate user's network traffic could bypass the application's use of SSL/TLS encryption, and use the application as a platform for attacks against its users. This attack is performed by rewriting HTTPS links as HTTP so that if a targeted user follows a link to the site from an HTTP page, their browser never attempts to use an encrypted connection. The sslstrip tool automates this process.
    
    To exploit this vulnerability, an attacker must be suitably positioned to intercept and modify the victim's network traffic. This scenario typically occurs when a client communicates with the server over an insecure connection such as public Wi-Fi, or a corporate or home network that is shared with a compromised computer. Common defences such as switched networks are not sufficient to prevent this. An attacker situated in the user's ISP or the application's hosting infrastructure could also perform this attack. Note that an advanced adversary could potentially target any connection made over the Internet's core infrastructure.
```
大意就是说我们的站点虽然有强制 https 请求，但是并没有添加 `Strict-Transport-Security`，所以有可能会遭受 SSL 剥离攻击。 建议我们赶紧加上。 

最后 POC 附上 [hstspreload](https://hstspreload.org/) 检测截图

![png](6.png)

<!--more-->
## 分析
查看了一下，确实在请求的时候，并没有下发 `Strict-Transport-Security` 这个头部。 在说明危害性的时候，按照惯例，先科普一下什么是 HSTS。

## 启用 HTTPS 也不够安全
在解释什么是 HSTS, 先了解一个场景， 现在已经是全站 HTTPS 的年代了，但用户在访问某个网站的时候，在浏览器里却往往直接输入网站域名 ( 例如 www.example.com )，而不是输入完整的URL ( 例如https://www.example.com )，不过浏览器依然能正确的使用HTTPS发起请求。这背后多亏了服务器和浏览器的协作，如下图所示(服务器和浏览器在背后帮用户做了很多工作):

![png](7.png)

具体步骤就是:
1. 浏览器使用 http 协议请求到底 web server
2. web server 发现是 http 协议，就返回 301 状态码，并带上 https 协议的 url
3. 浏览器发现 301 状态，直接取 response 中的 Location 头部中的地址，然后直接跳转

当然在用户眼里看来，在浏览器里直接输入域名却依然可以用HTTPS协议和网站进行安全的通信，是个不错的用户体验。但是有一个安全隐患，由于在建立起HTTPS连接之前存在一次明文的HTTP请求和重定向（上图中的第1、2步），使得攻击者可以以中间人的方式劫持这次请求，从而进行后续的攻击，例如窃听数据，篡改请求和响应，跳转到钓鱼网站等。

![png](8.png)

具体步骤就是:
1. 浏览器发起一次明文HTTP请求，但实际上会被攻击者拦截下来
2. 攻击者作为代理，把当前请求转发给钓鱼网站
3. 钓鱼网站返回假冒的网页内容
4. 攻击者把假冒的网页内容返回给浏览器

这个攻击的精妙之处在于，攻击者直接劫持了HTTP请求，并返回了内容给浏览器，根本不给浏览器同真实网站建立HTTPS连接的机会，因此浏览器会误以为真实网站通过HTTP对外提供服务，自然也就不会向用户报告当前的连接不安全。于是乎攻击者几乎可以神不知鬼不觉的对请求和响应动手脚。

## 使用 HSTS 来解决
既然建立HTTPS连接之前的这一次HTTP明文请求和重定向有可能被攻击者劫持，那么解决这一问题的思路自然就变成了如何避免出现这样的HTTP请求。我们期望的浏览器行为是，当用户让浏览器发起HTTP请求的时候，浏览器将其转换为HTTPS请求，直接略过上述的HTTP请求和重定向，从而使得中间人攻击失效，规避风险。其大致流程如下(略过HTTP请求和重定向，直接发送HTTPS请求)：

![png](9.png)

具体步骤就是:
1. 用户在浏览器地址栏里输入网站域名，浏览器得知该域名应该使用HTTPS进行通信
2. 浏览器直接向网站发起HTTPS请求
3. 网站返回相应的内容

当然浏览器不可能自己去知道这个流程，肯定是有东西告诉它要这么做，这个就是 HSTS 了。

HSTS(HTTP Strict Transport Security)是国际互联网工程组织 IETF 发布的一种互联网安全策略机制。采用HSTS策略的网站将保证浏览器始终连接到该网站的HTTPS加密版本，不需要用户手动在URL地址栏中输入加密地址，以减少会话劫持风险。

HSTS 最早于2015年被纳入到 ThoughtWorks 技术雷达，并且在2016年的最新一期技术雷达里，它直接从“评估（Trial）”阶段进入到了“采用（Adopt）“阶段，这意味着ThoughtWorks强烈主张业界积极采用这项安全防御措施，并且ThoughtWorks已经将其应用于自己的项目。

HSTS 最为核心的是一个HTTP响应头（HTTP Response Header）。正是它可以让浏览器得知，在接下来的一段时间内，当前域名只能通过HTTPS进行访问，并且在浏览器发现当前连接不安全的情况下，强制拒绝用户的后续访问要求。

## HSTS 语法介绍
HSTS Header的语法如下：
```html
Strict-Transport-Security: max-age=expireTime [; includeSubDomains] [; preload]
```
- `max-age`，单位是秒，用来告诉浏览器在指定时间内，这个网站必须通过HTTPS协议来访问。也就是对于这个网站的HTTP地址，浏览器需要先在本地替换为HTTPS之后再发送请求。
- `includeSubDomains`，可选参数，如果指定这个参数，表明这个网站所有子域名也必须通过HTTPS协议来访问。
- `preload`，可选参数，一个浏览器内置的使用HTTPS的域名列表。

所以只要让你的服务器在返回给浏览器的响应头中，增加 `Strict-Transport-Security` 这个HTTP Header:
```html
Strict-Transport-Security: max-age=31536000; includeSubDomains
```
就可以告诉浏览器，在接下来的 31536000秒内（1年），对于当前域名及其子域名的后续通信应该强制性的只使用HTTPS，直到超过有效期为止。 完整的流程如下:

![png](10.png)

只要是在有效期内，浏览器都将直接强制性的发起HTTPS请求，但是问题又来了，有效期过了怎么办？其实不用为此过多担心，因为HSTS Header 存在于每个响应中，随着用户和网站的交互，这个有效时间时刻都在刷新，再加上有效期通常都被设置成了1年，所以只要用户的前后两次请求之间的时间间隔没有超过1年，则基本上不会出现安全风险。更何况，就算超过了有效期，但是只要用户和网站再进行一次新的交互，用户的浏览器又将开启有效期为1年的HSTS保护。

## HSTS 下浏览器强制拒绝不安全的连接
在没有 HSTS 保护的情况下，当浏览器发现当前网站的证书出现错误，或者浏览器和服务器之间的通信不安全，无法建立HTTPS连接的时候，浏览器通常会警告用户，但是却又允许用户继续不安全的访问。如下图所示，用户可以点击图中红色方框中的链接，继续在不安全的连接下进行访问。

![png](11.png)

理论上而言，用户看到这个警告之后就应该提高警惕，意识到自己和网站之间的通信不安全，可能被劫持也可能被窃听，如果访问的恰好是银行、金融类网站的话后果更是不堪设想，理应终止后续操作。然而现实很残酷，就我的实际观察来看，有不少用户在遇到这样的警告之后依然选择了继续访问。

不过随着HSTS的出现，事情有了转机。对于启用了浏览器 HSTS 保护的网站，如果浏览器发现当前连接不安全，它将仅仅警告用户，而不再给用户提供是否继续访问的选择，从而避免后续安全问题的发生。例如，当访问Google搜索引擎的时候，如果当前通信连接存在安全问题，浏览器将会彻底阻止用户继续访问Google，如下图所示:

![png](12.png)

## 使用 preload list 防止首次劫持风险
虽然 HSTS 可以很好的解决 HTTPS 降级攻击，但是对于 HSTS 生效前的首次 HTTP 请求或者当浏览器没有当前网站的 HSTS 信息的时候，依然无法避免被劫持。浏览器厂商们为了解决这个问题，提出了 HSTS Preload List 方案：内置一份可以定期更新的列表，对于列表中的域名，即使用户之前没有访问过，也会使用HTTPS协议。

目前这个 Preload List 由 Google Chrome 维护，Chrome、Firefox、Safari、IE 11和Microsoft Edge都在使用。如果要想把自己的域名加进这个列表，首先需要满足以下条件：
1. 拥有合法的证书(如果使用SHA-1证书，过期时间必须早于2016年)；
2. 将所有HTTP流量重定向到HTTPS；
3. 确保所有子域名都启用了HTTPS；
4. 输出HSTS响应头：
5. max-age不能低于18周(10886400秒)；
6. 必须指定includeSubdomains参数；
7. 必须指定preload参数；

即便满足了上述所有条件，也不一定能进入 HSTS Preload List，更多信息可以查看：https://hstspreload.org/。 

通过 Chrome 的 chrome://net-internals/#hsts 工具，可以查询某个网站是否在 Preload List 之中，还可以手动把某个域名加到本机 Preload List。

![png](13.png)

## HSTS 缺点
HSTS 并不是 HTTP会话劫持的完美解决方案。用户首次访问某网站是不受 HSTS 保护的。这是因为首次访问时，浏览器还未收到HSTS，所以仍有可能通过明文HTTP来访问。 如果用户通过 HTTP 访问 HSTS 保护的网站时，以下几种情况存在 SSL 剥离劫持的可能性：
1. 以前从未访问过该网站
2. 最近重新安装了其操作系统
3. 最近重新安装了其浏览器
4. 切换到新的浏览器
5. 切换到一个新的设备，如：移动电话
6. 删除浏览器的缓存
7. 最近没访问过该站并且max-age过期了

解决这个问题目前有两种方案：
1. 在浏览器预置HSTS域名列表，就是上面提到的 HSTS Preload List 方案。该域名列表被分发和硬编码到主流的Web浏览器。客户端访问此列表中的域名将主动的使用HTTPS，并拒绝使用HTTP访问该站点。

2. 将HSTS信息加入到域名系统记录中。但这需要保证DNS的安全性，也就是需要部署域名系统安全扩展。

### 还有其它可能存在的问题
由于HSTS会在一定时间后失效(有效期由max-age指定)，所以浏览器是否强制 HSTS 策略取决于当前系统时间。大部分操作系统经常通过网络时间协议更新系统时间，如Ubuntu每次连接网络时，OS X Lion每隔9分钟会自动连接时间服务器。攻击者可以通过伪造NTP信息，设置错误时间来绕过HSTS。

解决方法是认证NTP信息，或者禁止NTP大幅度增减时间。比如：Windows 8 每7天更新一次时间，并且要求每次NTP设置的时间与当前时间不得超过15小时。

## 支持 HSTS 的浏览器
目前主流浏览器都已经支持HSTS特性，具体可参考下面列表：
1. Google Chrome 4及以上版本
2. Firefox 4及以上版本
3. Opera 12及以上版本
4. Safari从OS X Mavericks起
5. Internet Explorer及以上版本

## CVSS 3.0 的评分
我们知道如果没有 HSTS 加持的话，就有可能会出现 SSL 剥离攻击，导致 HTTP 的会话被劫持。 那么他的危害性是属于哪一个级别呢，如果是按照 CVSS 3.0 的评分，大概是多少分？ (关于 CVSS, 可以看 {% post_link cvss %})

以 `Missing HTTP Strict Transport Security Policy`  可以在这两个站点找到对应的 CVSS 3.0 的评分
- [netsparker](https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/http-strict-transport-security-hsts-policy-not-enabled/) -> medium
- [tenable](https://www.tenable.com/plugins/was/98056) -> medium

![png](14.png)

评分的时候，危险级就是 medium。

## HSTS 部署
HSTS 策略只能在 HTTPS 响应中进行设置，网站必须使用默认的443端口；必须使用域名，不能是IP。因此需要把HTTP重定向到HTTPS，如果明文响应中允许设置HSTS头，中间人攻击者就可以通过在普通站点中注入HSTS信息来执行DoS攻击。
### 1. Nginx 上启用 HSTS
```html
add_header Strict-Transport-Security "max-age=63072000; includeSubdomains; preload";
```
启用之后要重启: 
```html
$ service nginx restart
```

### 2. Apache 上启用 HSTS
```html
Header always set Strict-Transport-Security "max-age=63072000; includeSubdomains; preload"
```
启用之后要重启: 
```html
$ service apche2 restart
```

### 3. IIS启用HSTS
要在IIS上启用HSTS需要用到第三方模块，具体可参考：https://hstsiis.codeplex.com/

### 4. 其他云平台
#### 腾讯云
如果是腾讯云，那么提个工单，让他们后台帮我们配就行了。
#### aws cloudfront
关于在 cloudfront 添加安全头部，可以参考之前写的这个: {% post_link cloudfront-add-x-frame-options %}, 只不过 node 的脚本换成:
```html
exports.handler = (event, context, callback) => {
    //Get contents of response
    const response = event.Records[0].cf.response;
    const headers = response.headers;
    //Set new headers
    headers['strict-transport-security'] = [{key: 'Strict-Transport-Security', value: 'max-age=63072000; includeSubdomains; preload'}];
    //Return modified response
    callback(null, response);
};
```
当然如果你的站点由于某些其他原因，也要允许 http 请求存在，那么配置的时候，就不能加上 `includeSubdomains` 和 `preload` 这个配置项。

## 开启之后的测试
假设我通过 nginx 开启了 HSTS 的配置:
```html
add_header Strict-Transport-Security "max-age=300";
```
最简单的测试当然是直接 curl 来测试:
```html
[kbz@centos156 ~]$ curl -I https://www.example.com
HTTP/1.1 200 OK
Server: openresty
Date: Wed, 27 May 2020 12:10:36 GMT
Content-Type: text/html
Content-Length: 30759
Last-Modified: Wed, 20 May 2020 07:10:44 GMT
Connection: keep-alive
Keep-Alive: timeout=30
Vary: Accept-Encoding
ETag: "5ec4d7f4-7827"
X-Frame-Options: SAMEORIGIN
Strict-Transport-Security: max-age=300
Accept-Ranges: bytes
```
当然也可以直接在站点实测一下完整流程，直接用 http 的协议输入, 可以看到有进行了一次 301 跳转：

![png](1.png)
 
这个是 nginx 配置的，如果是 http 的请求，那么就会进行 301 重定向到 https， 所以也可以看到 nginx 有两条记录，先是 301， 然后再是 200 :
```html
www.xxx.com 125.xx.xx.xxx "125.xx.xx.xxx" - - [25/May/2020:03:06:57 +0000] "GET / HTTP/1.1" "301" 182 "-" "Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.132 Safari/537.36" 0.000 477 1439 [-]
www.xxx.com 125.xx.xx.xxx "125.xx.xx.xxx" - - [25/May/2020:03:06:57 +0000] "GET / HTTP/2.0" "200" 6072 "-" "Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.132 Safari/537.36" 0.000 6269 56 [-]
```
这时候可以看到在 https 请求的 response，就会返回这个头部了:
```html
strict-transport-security: max-age=300
```
然后接下来我们再请求一次，还是依然输入 http 协议的网址，然后看看是什么效果:

![png](2.png)

可以看到还是有一个 3xx 开头的跳转，307 跳转到 https 的协议:

![png](3.png)

而且很神奇的是，这个 307 的跳转我并没有在 nginx 的配置上看到过，而且 nginx 的 access log 也只有 200 的状态值这一条，没有 307 跳转这一条。 所以答案就很明显了， 这个 307 跳转其实是浏览器自己处理的，并没有到达 nginx 的服务端层面。 其实就是服务端检测到这个域名有配置 HSTS, 所以就将 http 协议自动转为 https 协议，然后再发送到服务端，这也是为啥服务端只看到一条 access log 的原因，因为 http 协议根本就没有到达服务器。

## HSTS 缓存的清除
在测试 HSTS 的时候，我们需要经常对 HSTS 的配置进行清除，最常规的有两种:
1. 将 max-age 设置一个很短的时间，然后等待它过期，再测试
2. 清除浏览器缓存
3. 每次都开启一个隐身模式(无痕模式)的 tab 页

其中前两种都不方便，尤其是第二种，清除浏览器缓存，因为很多密码的记住密码的保存都放在缓存中，一旦清除了浏览器的整体缓存，这些密码后面又要重新输入了。 第三种倒是可以接受，不过每次测试都得重新再开一个窗口，不能一直用。 其实一些现代浏览器，比如 Chrome，Firefox 都有针对清除 HSTS 缓存的功能。
### 1. Chrome 清除 HSTS
1. 在 tab 页打开 chrome://net-internals/#hsts
2. 在 `Query HSTS/PKP domain` 输入要检测的域名，如果存在 HSTS 缓存，下面会列出来，如果不存在，下面就会变成 `Not found`

![png](4.png)

请注意，这是一个非常敏感的搜索。请只输入主机名，比如 www.example.com 或 example.com，不要输入任何相关的协议或路径。

3. 如果存在的话，到最下面的 `Delete domain security policies` 然后输入这个域名，最后点击 delete 按钮，就可以清除了

![png](5.png)

### 2. Firefox 清除 hsts
1. 关闭 Firefox 中所有打开的标签。
2. 利用键盘快捷键 Ctrl + Shift + H（Mac上为Cmd + Shift + H）打开完整的历史窗口。在以下步骤中，你必须使用到这一窗口或侧边栏。
3. 找到你想要为之删除HSTS设置的网站——如果需要，你可以在右上角搜索该网站。
4. 从项目列表中右键点击该网站，并点击忘记这个网站。这将会清除这个域名的HSTS设置（以及其他缓存数据）。
5. 重启Firefox并访问该网站。

## 总结
HSTS 从 web 安全的角度来看，还是非常有必要的，尤其是现在全站 HTTPS 的时代，更要防止这种因为首次请求 http 而被劫持的 SSL 剥离攻击。

---
参考资料:
- [HSTS详解](https://blog.csdn.net/u014311799/article/details/79037717)
- [开启HSTS让浏览器强制跳转HTTPS访问](https://www.cnblogs.com/luckcs/articles/6944535.html)
- [如何清除Chrome和Firefox中的HSTS设置？](https://www.racent.com/blog/clear-hsts-settings-chrome-firefox)
- [hstspreload](https://hstspreload.org/)




