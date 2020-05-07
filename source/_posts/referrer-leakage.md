---
title: 页面禁用 referer - 第三方站点的 referer 头部泄露重置密码链接
date: 2020-05-07 19:33:41
tags: security
categories: web安全
---
## 前言
之前有个好心的白帽子有发了一个邮件过来:
```text
i am a security researcher and i founded this vulnerability in your website.

I have found that you are leaking via the referrer the reset password link. I am attaching the photo as proof of concept that the site is indeed leaking the reset password link via the referrer.

Proof of concept:       
图片1
图片2
图片3
    Thats when someone loads the reset password link and decided to click on external links.

Impact

Password Reset Token will be leaked by the Server to that third party site and that token can be used by third parties to reset the password and take over the account.

Thanks  
```
他说我们的忘记密码功能的重置密码链接，会加载第三方统计，这时候带过去的 referer 头部就会将这个页面的 url 带过去，从而暴露了重置密码的 token。

## 分析
我分析了一下，发现重置密码这个页面，确实在加载第三方统计的时候，referer 头部会将整个 url 当做值带过去, 也就是这些第三方程序是可以在用户进入这个重置密码页面的时候，通过 referer 头部来得到这个重置密码的链接，比如下图：
<!--more-->

![png](1.png)

### 危害性
危害性一定程度上是有的，不过当初在做重置密码的时候，本身 token 就存在有效性了，比如半小时。 并且在重置密码的时候，还要输入正确的邮箱，才能重置成功。 所以还是比较安全。

但是总归是一个隐患，所以还是要解决掉。

## 解决方式
### 直接不加载第三方统计
这种是最简单的，不加载第三方统计，就不会有暴露问题。 但是有时候我们又需要第三方统计的一些数据来帮助我们了解用户的一些活跃数据和 talking data。所以这个不是最优解。

### 禁止 referer 头部
最好的方式就是这个页面在加载外部资源的时候，禁止 referer 头部。 后面我们也是用这种方式来处理的。

## 论页面禁止 referer 的 6 种方式
### 1. head 标签中添加 meta属性
可以在 head 标签中添加 meta 属性，设置 
```text
name='referrer' content='never'
```
referer 的 metedata 参数可以设置为以下几种类型的值：
- never
- always
- origin
- default

如果在文档中插入 meta 标签，并且 name 属性的值为 referrer，浏览器客户端将按照如下步骤处理这个标签：
1. 如果 meta 标签中没有 content 属性，则终止下面所有操作
2. 将 content 的值复制给 referrer-policy ，并转换为小写
3. 检查 content 的值是否为上面 list 中的一个，如果不是，则将值置为 default

接下来浏览器后续发起 http 请求的时候，会按照 content 的值，做出如下反应 (下面 referer-policy 的值即 meta 标签中 content 的值):
1. 如果 referer-policy 的值为 never：删除 http head 中的 referer
2. 如果 referer-policy 的值为 default：如果当前页面使用的是 https 协议，而正要加载的资源使用的是普通的 http 协议，则将 http header 中的 referer 置为空
3. 如果 referer-policy 的值为 origin：只发送 origin 部分
4. 如果 referer-policy 的值为 always：不改变http header 中的 referer 的值，注意：这种情况下，如果当前页面使用了 https 协议，而要加载的资源使用的是 http 协议，加载资源的请求头中也会携带 referer

#### 例子
举个例子，如果你的页面是:
```text
<meta name="referrer" content="never">
```
那么加载的第三方资源将不会带上 referer 头部:

![png](2.png)

而且可以看到请求的 `Referrer Policy` 变成 `no-referrer`。

然后如果是换成:
```text
<meta name="referrer" content="origin">
```
就换变成只带 host 域名， 跟 origin 头部几乎一样 (多了最外面一个斜杠)

![png](3.png)

然后对应请求的 `Referrer Policy` 变成 `origin`:

![png](4.png)

#### 兼容性
这个标准还是比较老的，不过通过 [can i use meta:referrer](https://caniuse.com/#search=meta%20referrer) 还是可以看到大部分的主流浏览器 (Edge, Firefox, Chrome) 都有支持 (我亲测过了)

![png](5.png)

### 2. 添加ReferrerPolicy属性
添加 meta 标签相当于对文档中的所有链接都取消了 referer ，而 ReferrerPolicy 则更精确的指定了某一个资源的 referer 策略。关于这个策略的定义可以参照 MDN。比如我想只对某一个图片取消referrer，如下编写即可:
```text
<img src="xxxx.jpg"  referrerPolicy="no-referrer" />
```
A 标签也支持这个属性:
```text
<a href="xxxx.html"  referrerPolicy="no-referrer" />
```

#### 兼容性
通过 [can i use referrerPolicy](https://caniuse.com/#search=ReferrerPolicy)， 可以看到除了 IE 和 少部分手机浏览器， 大部分的主流浏览器还是支持的:

![png](6.png)

### 3. 通过 rel=’noreferrer‘
还可以通过标签的 rel 属性来禁止 referer 头部:
```text
<a href="xxxx.html"  rel="noreferrer" />
```
当然目前兼容的有限, 只支持  `<a>`, `<area>`, `<form>` 这三个元素:
{% blockquote mozilla https://developer.mozilla.org/en-US/docs/Web/HTML/Link_types/noreferrer %}
The noreferrer keyword for the rel attribute of the `<a>`, `<area>`, and `<form>` elements instructs the browser, when navigating to the target resource, to omit the Referer header and otherwise leak no referrer information — and additionally to behave as if the noopener keyword were also specified.
{% endblockquote %}

通过 [can i use noreferrer](https://caniuse.com/#search=noreferrer) 可以看到支持的浏览器版本:

![png](7.png)

### 4. 代理模式
这个就比较好理解了，把自己的服务器当做代理服务器, 请求先经过自己服务器, 修改referer头, 再反向代理到真正的服务器地址。

### 5. 外链 通过 iframe 来打开
如果是通过外链的话，那么可以通过 iframe 的方式来打开:
```text
function open_without_referrer(link){
  document.body.appendChild(document.createElement('iframe')).src = 'javascript:"<script>top.location.replace(\''+link+'\')<\/script>"';
}
```
这个其实就是通过 `top.location.replace` 方法替换当前的页面，从而丢失掉 referer 来源，这时候如果点击浏览器的回退按钮，就会发现已经回退不过去了。

### 6. 外链通过新窗口打开
如果是通过 window.open 打开的方式，也可以这样做:
```text
function open_new_window(full_link){ 
    window.open('javascript:window.name;', '<script>location.replace("'+full_link+'")<\/script>');
 }
```
这个跟上面的 iframe 差不多，也是通过 `location.replace` 方法来更新新打开窗口的文档。从而丢失掉 referer 来源。

### 推荐一个第三方库 noreferrer.js
提供跨浏览器支持的更好的办法是使用一个第三方的库 [noreferrer.js](https://github.com/knu/noreferrer/blob/master/noreferrer.js)，它可以自动识别浏览器并选择最优方案。

不过这边需要注意一点的是, 对于某些几乎不支持的特性浏览器，比如 Opera，`noreferrer.js` 的解决方案是利用 google 的url中转。在国内的网络环境下，你懂的。。。 所以可以自己搭建一个跳转的页面，或者用其他站点的url跳转接口。

## 最后的解决方案
当然回到本次的应用场景，后面我还是采用 meta 标签的方式，然后将 content 设置为 origin， 让其只传 origin，这样也是为了方便某些第三方统计的数据收集和跟踪。

不过后面还发现一个问题，我发现有些第三方统计，还会有自己的自定义头部，然后还是带上整个页面的 url:

![png](8.png)

针对这种情况，那就整个的第三方统计都不要加载了。

---
参考文章:
- [html禁用referer](https://blog.csdn.net/weixin_43627766/article/details/90311889)
- [如何让浏览器在访问链接时不要带上referer](https://segmentfault.com/q/1010000000123441)
