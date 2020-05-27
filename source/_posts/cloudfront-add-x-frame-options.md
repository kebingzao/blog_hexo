---
title: web 安全之 - cloudfront 添加 x-frame-options 防止 Clickjacking
date: 2020-04-26 09:59:25
tags: cloudfront
categories: web安全
---
## 前言
前段时间一个好心的白帽子，发了一封邮件告知我们的某一个站点可以被其他第三方站点内嵌，会有点击劫持(Clickjacking)的风险, 事实上关于 Clickjacking 这个安全问题，之前我就在 {% post_link web-forbidden-iframe-embed %} 写过怎么处理了， 无非就是加上 `x-frame-options` 这个头部， 不过那时候出问题的那个站点是部署在我们的服务器的，是用 nginx 的，所以在 nginx 的配置上加上:
```php
add_header X-Frame-Options "SAMEORIGIN";
```
就可以了。 但是这次的站点比较特殊，所以他是有 CDN 加速的，而且是如果是在国内访问的话，用的 CDN 加速是网宿的， 国外访问的话， 用的 CDN 加速是 AWS 的 cloudfront。 当然无论是国内还是国外，获取的源都在 aws 的 S3 上。 所以如果要添加 `x-frame-options` 这个头部，那么无论是 网宿 还是 cloudfront 都要加。
<!--more-->

![png](1.png)

其实关于 CDN 添加头部的情景，早在 {% post_link www-history-13 %} 这篇文章就有讲过了，无论是 腾讯云的 CDN 还是 aws 的 cloudfront 都可以转发 CORS 头部 (在源 bucket 进行头部添加，然后对应的 CDN 服务进行头部转发)。 而 `x-frame-options` 这个头部原则上也可以实现在 源 bucket 上进行添加，然后由对应的 cloudfront 去转发。 不过这一次我没有用这种方式，而是用另一种方式。

## 网宿的处理
网宿的处理很简单，我们简单讲一下，就是去后台，添加头部，然后提交工单，等审核通过就行了。

![png](2.png)

然后生效了之后，就可以通过 `curl -I http://example.com` 来查看这个站点的的返回头部:
```php
[kbz@centos156 ~]$ curl -I http://example.com
HTTP/1.1 200 OK
Date: Sun, 26 Apr 2020 01:49:25 GMT
Content-Type: text/html
Content-Length: 6261
Connection: keep-alive
x-amz-id-2: TIvtawqX+hDmjxUPWdo3grl5oCb1dYxrh/eWstYYPHlk7c/T1YN/h/OWaVITPLnxoELAOXygl4M=
x-amz-request-id: 40306F4EEE5B59F5
Last-Modified: Tue, 21 Apr 2020 03:36:42 GMT
ETag: "3ffde77ec97042c69b8957660c0c2490"
Server: AmazonS3
X-Via: 1.1 uzhoudianxin70:1 (Cdn Cache Server V2.0), 1.1 jfzhdx145:4 (Cdn Cache Server V2.0)
X-Ws-Request-Id: 5ea4e8a5_jfzhdx145_20556-54723
X-Frame-Options: SAMEORIGIN
```
可以看到最后一行  `x-frame-options` 加上去了。

## cloudfront 的处理
cloudfront 添加这些安全头部是要结合 aws 的另一个服务 Lambda 来处理的，这边有一个很详细的教程，对着教程处理就行了: [使用Lambda @ Edge和Amazon CloudFront添加HTTP安全标头](https://aws.amazon.com/cn/blogs/networking-and-content-delivery/adding-http-security-headers-using-lambdaedge-and-amazon-cloudfront/?nc1=h_ls)。

接下来我们就按照教程来执行

### 1. 创建 Lambda Console
首先进入Lambda Console，通过从右上角的下拉列表中选择 US East（弗吉尼亚北部），确保自己位于美国东北1弗吉尼亚州地区。然后，我选择创建函数来创建一个新的Lambda函数。
 
![png](3.png)

### 2. 编辑代码
接下来进入函数， 编辑代码:
```php
exports.handler = (event, context, callback) => {
    //Get contents of response
    const response = event.Records[0].cf.response;
    const headers = response.headers;
    //Set new headers
    headers['x-frame-options'] = [{key: 'X-Frame-Options', value: 'SAMEORIGIN'}];
    //Return modified response
    callback(null, response);
};
```

![png](4.png)

### 3. 创建 cloudfront 触发器
这边要注意一个细节，为啥要先编辑代码的原因就是编辑代码只能在 $LATEST 分支才行。 而创建触发器成功会直接升级版本成功。 所以要先编辑代码，然后创建触发器，这时候版本就会变成 `版本1`, 所以当前的版本就是 `版本1`。

如果是先添加触发器，就会变成添加成之后，已经变成 `版本1`，这时候 `版本1` 是不能编辑代码，要编辑代码只能回到 $LATEST 分支， 所以只能到 $LATEST 编辑代码，然后重新添加触发器，这时候就会变成 `版本2`， 相当于多了一个版本。

ps: 每次添加触发器，都会自动升级一个版本。

![png](5.png)

选择 CloudFront， 然后点击 `部署到 Lambda@Edge`

![png](6.png)

这时候选择我们要的那个 cloudfront， 然后选择事件是 源响应 (默认是源请求)

![png](7.png)

这样子点击部署，就会开始部署了。

![png](8.png)

这时候版本就会自动升级到 `版本1`, 当前应用版本也是 `版本1`。 这时候回到函数页面会发现没有刚才的触发器，这时候要切换到 `版本1` 才能看到这个触发器。

![png](9.png)

最后等几分钟之后， cloudfront 重新部署完了。 我们随便登录一台国外的服务，就可以看到已经有加这个头部了

```php
[kbz@VM_16_20_centos ~]$ curl -I http://example.com
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 6261
Connection: keep-alive
Date: Sun, 26 Apr 2020 01:37:36 GMT
Last-Modified: Tue, 21 Apr 2020 03:36:42 GMT
ETag: "3ffde77ec97042c69b8957660c0c2490"
Accept-Ranges: bytes
Server: AmazonS3
X-Frame-Options: SAMEORIGIN
X-Cache: Hit from cloudfront
Via: 1.1 055ec3ff1e0949e45c1919a4f69a8063.cloudfront.net (CloudFront)
X-Amz-Cf-Pop: HIO51-C1
X-Amz-Cf-Id: APDg_Lmk_TjzjWcEH9_cjK234NoFmOhw0JnsfUV--GLc-Z63uRoTNw==
Age: 6
```
这样就可以了。 不过要注意一点的是，我之前在创建触发器的时候，会有角色权限问题，用我自己的账号来设置的话，会报这个错误:

![png](10.png)

后面直接切换成 管理员账号来操作就不会报这个错了。

## 测试
当添加了 `x-frame-options` 之后，当然要测试一下，是否表现正常。

用 Firefox 测试是正常的， iframe 内嵌就加载不出来了：

![png](11.png)

用 chrome 也是正常的。 说明这个头部应用成功。



