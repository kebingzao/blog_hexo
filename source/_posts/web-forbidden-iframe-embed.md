---
title: web 页面防iframe嵌入（防止点击劫持Clickjacking）
date: 2018-05-05 23:14:34
tags: security
categories: web安全
---
之前我们项目的官网有反馈一个问题，就是用户将我们的官网的页面嵌入到他自己的站点。

通过这种方式，可以用来做点击劫持 Clickjacking

因此我们要防止 页面内嵌


#### 方法1： 通过js来防止：

{% codeblock lang:js %}
if (window.location != window.parent.location) {
 window.parent.location = window.location;
}
{% endcodeblock %}

但是这个容易被破解，只要

// 顶层窗口中放入代码 var location = document.location;
// 或者 var location = "";

<!--more-->
#### 方法2：使用 X-Frame-Options  header

使用X-Frame-Options防止网页被Frame

因为我们用的是nginx，所以只要添加这一栏header 即可

{% codeblock lang:js %}
add_header X-Frame-Options SAMEORIGIN;
{% endcodeblock %}

这个方法才能真正杜绝点击劫持

---

先拿测试的来测。

![step one](web-forbidden-iframe-embed/1.png)

发现加了之后，果然不能嵌入了

![step one](web-forbidden-iframe-embed/2.png)

设置是 SAMEORIGIN ，也就是说同源的就可以了

---

#### 使用 X-Frame-Options 有三个可选的值：
* DENY：浏览器拒绝当前页面加载任何Frame页面
* SAMEORIGIN：frame页面的地址只能为同源域名下的页面
* ALLOW-FROM：允许frame加载的页面地址


注意这边有个细节: `ALLOW-FROM` 这个参数其实新版的浏览器已经不再支持了。所以就不要用了:
- [X-Frame-Options: ALLOW-FROM in firefox and chrome](https://stackoverflow.com/questions/10658435/x-frame-options-allow-from-in-firefox-and-chrome)
- [X-Frame-Options](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options)

{% blockquote mozilla https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options %}
ALLOW-FROM uri (obsolete)
This is an obsolete directive that no longer works in modern browsers. Don't use it. In supporting legacy browsers, a page can only be displayed in a frame on the specified origin uri. Note that in the legacy Firefox implementation this still suffered from the same problem as SAMEORIGIN did — it doesn't check the frame ancestors to see if they are in the same origin. The Content-Security-Policy HTTP header has a frame-ancestors directive which you can use instead.
{% endblockquote %}

也就是说，如果需要用到 `allow-from` 来指定固定的域名的话，可以用 [Content-Security-Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) 这个头部的 `frame-ancestors` 来代替。 

而且如果已经存在 `frame-ancestors` 这个值了，那么`X-Frame-Options` 这个头部将会被忽略:

- https://www.w3.org/TR/CSP2/#frame-ancestors-and-frame-options

{% blockquote mozilla https://www.w3.org/TR/CSP2/#frame-ancestors-and-frame-options %}
7.7.1. Relation to X-Frame-Options
This directive is similar to the X-Frame-Options header that several user agents have implemented. The 'none' source expression is roughly equivalent to that header’s DENY, 'self' to SAMEORIGIN, and so on. The major difference is that many user agents implement SAMEORIGIN such that it only matches against the top-level document’s location. This directive checks each ancestor. If any ancestor doesn’t match, the load is cancelled. [RFC7034]

The frame-ancestors directive obsoletes the X-Frame-Options header. If a resource has both policies, the frame-ancestors policy SHOULD be enforced and the X-Frame-Options policy SHOULD be ignored.
{% endblockquote %}

---

#### 站点测试
可以通过 https://clickjacker.io/  来检测你的站点是否有 Clickjacking 的安全缺陷。

#### 除了X-Frame-Options之外，Firefox的"Content Security Policy"以及Firefox的NoScript扩展也能够有效防御ClickJacking，这些方案为我们提供了更多的选择