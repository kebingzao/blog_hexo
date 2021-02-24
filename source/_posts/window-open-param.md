---
title: window.open() 打开的子页面往跨域的主页面传参问题
date: 2021-02-24 15:25:20
tags:  js
categories: 前端相关
---
## 前言
前段时间有遇到比较奇葩的需求，就是正常情况下，我们站点在进行第三方登录的时候，会通过 `window.open` 的方法，打开一个服务端的接口页面，然后这个服务端的接口去判断当前是否有授权行为，如果有的话，就写入 cookie，然后关闭当前窗体， 如果没有的话，就跳转到第三方的授权页面，然后在授权结束，回调的时候，写入 cookie，然后关闭当前窗体，前端会监听当前的窗体，当发现已经关闭的时候，就会去请求自动登录的接口，这时候，因为 cookie 信息已经让服务端写入了，所以这时候就可以正常登录，这时候前端还会读取服务端的某一个 token cookie， 然后生成当前一次性令牌 token， 以供服务端进行校验。

正常情况下是没有问题的，但是由于 chrome 浏览器在新版本的时候，cookie的默认设置会有 `SameSite=Lax`，这时候就会导致前端读取不了服务端的 cookie ({% post_link cookie-samesite %})， 从而无法生成一次性访问 token 令牌， 这时候再去请求自动登录的接口，就会出现校验失败。

所以这边的想法是，服务端在生成 cookie 的时候，能不能在窗口关闭之前，往前端传 当前的一次性令牌的 token。这样子前端就可以带上这个 token 去进行自动登录的校验。

所以这边就会涉及到 `window.open()` 打开的子页面， 往主页面传参问题。

<!--more-->

## 前端模拟情况
接下来我们用前端模拟一下这种情况，看能不能实现，我们知道 window.open 打开的新窗口，是可以获取到主页面的窗口句柄的，也就是 `window.opener`, 所以原则上是可以操作这个主页面的 window 对象。 接下来我们模拟两个页面:

一个是主页面 `parent.html`:
```text
<!DOCTYPE>
<html>
<head>
<meta charset="utf-8" />
</script>
</head>
<body>
<button id="btn">点我打开新窗口</button>
<script>
document.getElementById("btn").onclick = function(){
	window.open('child.html',"新增","width=500,height=480,screenX=400,screenY=100");
}
// 子页面调用的函数
window.triggerFn = function(data){
 alert("trigger by child page, data is:" + data);
}
</script>
</body>
</html>
```
另一个是相同项目下的 `child.html` 页面：
```text
<!DOCTYPE">
<html>
<head>
<meta charset="utf-8" />
</script>
</head>
<body>
<button id="btn">点我传递消息给主页面</button>
<script>
document.getElementById("btn").onclick = function(){
	if (window.opener != null && !window.opener.closed) {
		window.opener.triggerFn("hello");
	}
	window.close(true);　
}
</script>
</body>
</html>
```

然后实践一下:

![1](1.png) 

然后点击子界面的按钮，这时候就可以发现消息传到主界面，同时子界面关闭了

![1](2.png) 

这时候发现子界面可以直接调用主界面的 window 对象(当然前提是主界面并没有关闭或者重新刷新)。

看起来好像挺完美的， 接下来我将子界面换成另一个域名的路径，之前是同个站点的页面, 修改主界面的这个代码为
```text
window.open('http://127.0.0.1/test/child.html',"新增","width=500,height=480,screenX=400,screenY=100");
// window.open('child.html',"新增","width=500,height=480,screenX=400,screenY=100");
```

这是再试下， 这是就发现点击就没有任何响应了

![1](3.png) 

看了一下控制台的报错，浏览器认为这个是跨域行为，因此被禁止了， 还是同源策略的原因。 第一次之所以可以，这个是因为两个页面是同源的。

```text
child.html:12 Uncaught DOMException: Blocked a frame with origin "http://127.0.0.1" from accessing a cross-origin frame.
    at HTMLButtonElement.document.getElementById.onclick (http://127.0.0.1/test/child.html:12:17)
```

这时候通过断点可以看到，虽然子界面看似可以访问主界面的 window 对象，但是所有的操作都是报这个错：

![1](4.png) 

所以如果把子页面改成服务端的页面也是一样的，因为都是跨域，而且就算服务端的页面在 response 的时候，加上 CORS 的跨域头，也是没有用。

## 服务端模拟情况
那么是不是真的无解呢， 跨域的情况下，看似子页面的 window.opener 对象虽然有，但是啥也干不了， 但是有一种情况还真可以干，那就是可以重写 `window.opener.location.href` 的值，这边涉及到一个 web 的安全问题: {% post_link a-target-blank %}, 这种看似安全缺陷的用法， 恰恰可以解决这个问题。

接下来我们直接通过服务端的接口来模拟，`client.html` 是客户端页面，也就是主页面的代码:
```text
<!DOCTYPE>
<html>
<head>
<meta charset="utf-8" />
</script>
</head>
<body>
<button id="btn">点我打开新窗口</button>
<script>
document.getElementById("btn").onclick = function(){
	var win = window.open('http://127.0.0.1:3000/child/',"新增","width=500,height=480,screenX=400,screenY=100");
	var t_q = setInterval(function () {
		if (win.closed) {
		    // child win is closed
			clearInterval(t_q);
		}
	},200)
}
// 监听 hash 变化
window.onhashchange = function(ev) {
	// 获取新的 hash， 我这边是简单处理
	var arr = document.location.hash.substring(1).split("=");
	if(arr.length > 1 && arr[0] === "result"){
		alert("服务端传的数据是:" + decodeURIComponent(arr[1]));
	}
}
</script>

</body>
</html>
```
然后服务端的代码如下:
```text
router.get('/child/', async (ctx, next) => {
    // 这边做一些很复杂的操作，从而得到 result 的值, 这边 sleep 2 s
    Atomics.wait(new Int32Array(new SharedArrayBuffer(4)), 0, 0, 2 * 1000);
    let resultJson = '{"result":"success","code":1}';
    ctx.body=`<script>
window.opener.location.href = 'http://localhost/test/client.html#result=${resultJson}' ;
window.close(); 
window.opener=null
    </script>`;
})
```
服务端我用的是 nodejs 模拟的，用的是 koa 框架，而且为了表示正在处理逻辑，还做了一个 2s 的 sleep 操作。

这边要注意一个细节，虽然可以重写主页面的 href ， 但是为了不影响当前主页面的操作， 肯定是不能引起页面刷新的，所以这边重写的时候，要将传递的参数放在主页面的 hash 参数后面，这样子主页面就不会引起刷新， 同时主页面通过监听 `window.onhashchange` 事件来获取服务端传过来的 hash 值。

具体操作截图如下，刚开始点击是这样子的：

![1](5.png) 

然后过了 2s 之后，子页面重写主页面的 url，并且关闭窗口，然后主页面通过监听 hash 变化，取的 hash 值:

![1](6.png) 

## 总结
通过这种方式，就可以在用 `window.open` 打开跨域的页面的时候，往主页面传递参数， 当然这个参数不能太大。而且也适合服务端的接口的情况。



