---
title: web 安全之 - A 标签target=”_blank”的安全问题及解决办法
date: 2020-05-14 19:16:41
tags: security
categories: web安全
---
## 前言
前段时间，有位好心的白帽子给我们发了一封邮件，讲了一个关于 A 标签target=”_blank”的安全问题:
```text
Hi team,

I am security researcher and i found this vulnerability in your website  
https://www.example.com/en/

Vulnerability report:  Tab nabbing
`
Issue lies Here :

< a class="item-social-icon item-social-icon-twitter" href= "http://twitter.com/#!/Team" target="_blank" data-type="footerNav-goTwitter" >

Here i can see you are using target=_blank and no more rel tag.
Here , target=_blank means it will open in another new tab but due to tab nabbing it can change parent tab as well .

FIX & MITIGATION :
To mitigate this issue we need to use rel="nofollow noopener noreferrer" as follows: 
```
吓的我赶紧检查了一下，这个问题确实存在，我发现官网的部分外链的 A 标签是通过 `target="_blank"` 来跳转的，所以就会存在这个安全问题。 万幸的是， 我们官网的外链大部分都是我们自己的内部网站。只有少部分是真正的外链, 比如 facebook，Twitter， YouTube 我们的主页，不过这些大的站点一般也不会太大的危害，但是当务之急还是将这些外链的这个安全问题修复一下。
<!--more-->
## 安全问题的原理
我们知道 A 标签的 target 属性规定在何处打开链接文档。如果在一个 A 标签内包含一个 target 属性，浏览器将会载入和显示用这个标签的 href 属性命名的、名称与这个目标吻合的框架或者窗口中的文档。如果这个指定名称或 id 的框架或者窗口不存在，浏览器将打开一个新的窗口，给这个窗口一个指定的标记，然后将新的文档载入那个窗口。从此以后，超链接文档就可以指向这个新的窗口。target="_blank" 的意思是新的浏览器窗口打开此超链接，但是大多数人都没有注意到这个属性其实是有安全缺陷的。

之所以有安全缺陷，是因为我们在调用 window 下的 open 方法创建一个新窗口的同时 (通过点击含有 target="_blank" 的 A 标签)，我们可以获得一个创建窗口的 opener 句柄，但你也许没注意到，通过target="_blank"点开的窗口或者标签页，子窗口也能捕获opener句柄，通过这个句柄，子窗口可以访问到父窗口的一些属性，虽然很有限，但是我们却可以修改父窗口的页面地址，让父窗口显示指定的页面。

如果你打开的是一个同域的页面，那么你将可以在新页面访问到原始页面的所有内容，包括document对象(window.opener.document)。 如果你打开的是一个跨域的页面，你虽然无法访问到document，但是你依然可以访问到location对象。 这意味着，如果你在你的站点或者文章中，嵌入了通过新窗口打开一个新页面的链接，这个新页面可以使用 window.opener，在一定程度上来修改原始页面。(就算是不同域，它也可以直接让你的站点所在的 url 换成它指定的任何页面)

## 举个例子
举个例子，在页面 a.html 中有这样一段代码:
```text
<a href="b.html" target="_blank">跳转</a>;
```
当我们点击页面 a.html 中的跳转链接时，浏览器会在新的窗口或标签页中打开 b.html，假如这个时候 b.html 中有这样一段js代码：
```text
if (window.opener) {
    window.opener.location.href = 'eval.html';
}
```
当页面 b.html 被打开的同时,原来打开 a.html 的标签页会被重定向到 eval.html ,  eval.html 可以是和原来域完全不相关的其它域的资源。

## 危害
既然可以修改父窗口的链接，让父窗口显示指定的页面，那么第一个想到的危害肯定就是钓鱼网站。 
1. 试想我在某一个站点点击了一个外链
2. 然后回过来再看原来的站点的时候，发现它已经被换成了一个跟原来界面一模一样的钓鱼网站，并且这个钓鱼提示你登录信息已过期，让你重新输入用户名和密码。 
3. 这时候很多人就傻傻的认为真的是登录信息过期了，所以就输入了。
4. 然后这个钓鱼网站得到用户名和密码，就重定向到真正的网站，这时候因为真网站的 cookie 还在的原因，所以也直接就登录进去了
5. 用户以为很正常，事实上账号早就不知不觉被盗走了。

当然这个有破绽，比如假网站和真网站的域名不一样，不过谁浏览网站的时候，会去观察网站的域名，而且有些假网站可以让域名变得跟真的域名非常相似，比如 `example.com` 和 `examp1e.com` 不认真看还真以为是同一个呢。

当然也并不需要过分担心，毕竟我们自己站内的资源是受信的，能放到站内的其他网站基本也是站长自己加的，当然不排除被黑的情况。

## 线上真实例子
如果还不了解上面的例子，线上有一个很好显示这个过程的例子:  https://mathiasbynens.github.io/rel-noopener/ ， 进入这个站点，我们点击这两个按钮的其中一个:

![png](1.png)

然后再回到原来的页面看，发现页面变了，变成了这个:

![png](2.png)

变成另外一个 url 了, 然后我们看下新打开窗口的源代码:
```html
<!DOCTYPE html>
<meta charset="utf-8">
<title>Go back!</title>
<h1>Why don’t you go back to the previous tab?</h1>
<script>
    if (window.opener) {
        opener.location = 'https://mathiasbynens.github.io/rel-noopener/#hax';
        // Just `opener.location.hash = '#hax'` only works on the same origin.
    } else {
        document.querySelector('h1').innerHTML = 'The previous tab is safe and intact. <code>window.opener</code> was <code>null</code>; mischief <em>not</em> managed!';
    }
</script>
```
确实就是判断 window.opener 这个句柄是否存在，如果存在，就意味着存在这个安全问题。 而且我发现我如果不直接点击这个连接，而是右击，然后选择 `在新标签页中打开链接`， 

![png](3.png)

这时候新打开的窗体就会变成这个:

![png](4.png)

说明通过这种方式打开的新窗口，并不会带上 `window.opener` 这个句柄 (鼠标点击滚轮中键直接打开也是这个效果，opener 也是为 null)。 所以就不会有这个安全问题了。

## 防范
防范也很简单，我们知道打开新窗口这个操作，可以由 A 标签触发，也可以由 JS 的 window.open 来调用，所以针对这两种情况都要处理:
### 针对 A 标签的防范
为了限制子页面通过 `window.opener` 控制父页面，我们需要在页面上所有使用了 `target="_blank"` 的链接上添加 `rel="noopener"` 属性，但是，火狐浏览器并不支持这个有特殊意义的属性 (事实上通过 [can i use noopener](https://caniuse.com/#search=noopener) 我发现新版的 Firefox 应该也是支持，但是可能是为了兼容旧版的吧)，火狐浏览器里需要写成 `rel="noreferrer"`，我们可以把它们的写法合并，写成`rel="noopener noreferrer"`:
```html
<a href="b.html" target="_blank" rel="noopener noreferrer">跳转</a>;
```
这样，子页面就获取不到父页面的句柄了。
### 针对 window.open 的防范
如果是用 js 的 window.open() 语法的，那么就要改成:
```html
var newWnd = window.open();
newWnd.opener = null;
newWnd.location = url;
```
## 子窗口得到父窗口 opener 句柄的应用场景
小朋友，你有没有疑问，既然 子窗口得到父窗口 opener 句柄，会有安全问题。 为啥浏览器还要保留这个功能， 为啥不删掉。 这个当然是因为这个功能也是有一定的应用场景，我举个应用到这个功能的场景： 第三方登录， 事实上我们站点的第三方登录(facebook，Twitter， google) 就用到了这个句柄，步骤如下:
1. 用户点击 facebook 的第三方登录，前端打开一个新窗口，定位到我们服务器的第三方登录接口，并且前端得到这个新窗口的 opener 对象
2. 服务端进行第三方的授权，当授权完成之后，服务端将授权的cookie写入，并关闭这个新窗口
3. 前端检测到这个窗口被关闭了，就知道第三方授权流程结束了(成功或者失败都有可能)，直接请求登录接口，这时候浏览器会将 cookie 带过去
4. 服务端检测带过来的 cookie，如果是第三方授权后的 cookie，那么就是第三方登录成功了。

所以前端的打开新窗口的代码就是:
```html
popup = window.open(url);
popup.focus();
interval = setInterval(function() {
    if (popup.closed) {
        // do login
        clearInterval(interval);
    }
}, 500);
```

## 总结
我们站点在添加外链的时候，如果不是我们自己的站点，一定要加上 `rel="noopener noreferrer"`, 用 window.open 打开新窗口的时候，也要记得将 window.opener 设置为 null，除非对 opener 句柄有特殊用途，比如上述的第三方登录之类的。

---
参考资料:
- [关于html的a标签的target="__blank "的安全漏洞问题](https://www.cnblogs.com/tugenhua0707/p/9709424.html)
- [链接地址中的target=”_blank”属性，为钓鱼攻击打开了大门](https://www.freebuf.com/vuls/113634.html)
- [使用<a>标签时，你可能会忽略的一个安全问题](https://juejin.im/post/5c36d52651882525e90dc800)
- [Target="_blank" - the most underestimated vulnerability ever](https://www.jitbit.com/alexblog/256-targetblank---the-most-underestimated-vulnerability-ever/)
- [About rel=noopener](https://mathiasbynens.github.io/rel-noopener/)











