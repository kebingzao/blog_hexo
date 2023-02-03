---
title: web 安全之 - 使用 proxy 请求页面来绕过 X-Frame-Options 机制
date: 2023-02-03 16:43:08
tags: security
categories: web安全
---
## 前言
前段时间有个 researcher 回报说，他可以通过使用 [X-Frame-Bypass](https://github.com/niutech/x-frame-bypass) 来绕过我们站点的 `X-Frame-Options` 机制，从而将我们的站点内嵌在他的网页上，从而造成 Clickjacking

对于 Clickjacking 来说，我们一般是使用设置 `X-Frame-Options` header 来防止站点被第三方内嵌，具体可以看我之前的文章:
- {% post_link web-forbidden-iframe-embed %}

通过 `X-Frame-Options` 我们可以很好的防止我们的站点被第三方内嵌。

那么 [X-Frame-Bypass](https://github.com/niutech/x-frame-bypass) 是怎么绕过 `X-Frame-Options` 这个限制的呢?

## 分析
我后面看了一下他的代码，他的方式其实很简单，既然 `X-Frame-Options` 是在 response 的 header 返回的，那么我只要在请求的时候，先去通过第三方服务将我想要的站点的内容取出来，然后只转发这个 body 体，不转发 header，就可以绕过了。

举个例子，比如我内嵌别人站点的代码是这样子的:
```javascript
<iframe src="https://api.codetabs.com/v1/proxy/?quest=https://test-example.com/my_test.html"></iframe>
```
这个逻辑很简单，就是这个 第三方 `api.codetabs.com`, 会去请求 `https://test-example.com/my_test.html`，并且将源代码返回，其实就是转发
<!--more-->
这个请求转发的逻辑很简单, 比如我用 node 自己写了一个页面抓取转发接口:
```javascript
const http = require('http')
const requestLib = require('request');

const server = http.createServer((req, res) => {
  console.log('method: ', req.method) // 打印请求方法，GET

  const url = req.url
  console.log('url: ', url) // 打印被访问的url

  const queryUrl = url.split('?request=')[1];
  if(queryUrl){
    // 接下来请求代理
    requestLib({
      url: queryUrl,
      method: "GET",
    },function (error, response, body){
        console.log('body:', body);
        res.end(body) // 将参数返回给前端
    });
  }else{
    res.end("")
  }
})

server.listen(8008, () => {
  console.log('http://localhost:8008')
})
```
可以看到我抓取之后，只转发了 body 体内容，直接忽略了 response 的 header 了，因此用我本地代理，跟上述等价的就是:
```javascript
<iframe src="http://192.168.40.51:8008/?request=https://test-example.com/my_test.html"></iframe>
```

然后最后的效果就是 `https://test-example.com/my_test.html` 这个页面就被内嵌了

![](1.png)

### X-Frame-Bypass 的做法
X-Frame-Bypass 的核心本质做法，跟我上述的 demo 类似，也是通过 proxy 代理的方式，去转发原页面的代码。 只不过他做的更全。

我上面的例子只针对当前的这个页面做处理，万一这个页面里面还有外链，或者有 form 表单提交呢，这些外链的 `X-Frame-Options` 配置就没办法绕过了。

而 X-Frame-Bypass 就有处理，他会去劫持当前页面的所有的外链形式，然后点击的时候，也走 proxy 代理劫持，相关代码:
```javascript
	// X-Frame-Bypass navigation event handlers
	document.addEventListener('click', e => {
		if (frameElement && document.activeElement && document.activeElement.href) {
			e.preventDefault()
			frameElement.load(document.activeElement.href)
		}
	})
	document.addEventListener('submit', e => {
		if (frameElement && document.activeElement && document.activeElement.form && document.activeElement.form.action) {
			e.preventDefault()
			if (document.activeElement.form.method === 'post')
				frameElement.load(document.activeElement.form.action, {method: 'post', body: new FormData(document.activeElement.form)})
			else
				frameElement.load(document.activeElement.form.action + '?' + new URLSearchParams(new FormData(document.activeElement.form)))
		}
	})
```
这样子就可以让表单提交和 href 外链都可以正常跳转。

## 防范
### 1. 使用 frame-ancestors
researcher 有建议后面用 CSP 的 `frame-ancestors` 的参数来防范， 但是我试了一下，有配:
```javascript
    add_header X-Frame-Options "SAMEORIGIN";
    add_header Content-Security-Policy "frame-ancestors 'self'";
```
但是发现没有效果

![](2.png)

因此我怀疑 `frame-ancestors` 这个配置也是没办法阻止 proxy 代理的，因为他跟 `X-Frame-Options` 一样，都是跟在原页面的 response 的 header，在 proxy 请求转发 body 体的时候，原 response 会被洗掉， 因为 proxy 只会转发 body 体， header 不会转发， 因此只要是 header 的方式，应该都不能生效。

除非这个规则本来是写在 html 里面的，但是很遗憾的是 CSP 的很多参数都可以写在 html 页面的 meta 中，但是 frame-ancestors 恰恰不能用在 meta 标签中生效中，只能在 header 中生效。
- {% post_link csp %}

### 2. 使用 js 代码判断
虽然没办法用 header 来防范，但是可以用 js 来防范，比如:
```javascript
if (window.location != window.parent.location) {
  console.log("oops!!!");
  window.stop ? window.stop() : document.execCommand("Stop");
}
```
这样子一旦该页面被内嵌的话，就会直接白屏:

![](3.png)

## 总结
通过 header 是没办法防的， 通过 js 可以防，不过既然都已经被 proxy 代理，意味着 js 也可以被破就是了。

不过这种危害其实是很低的，他有几个很大的缺陷:
1. 因为他的外链都被劫持代理了，因此导航栏的 url 是不会变的
2. 而且因为被代理了，因此 host 肯定变了，因此所有的需要 cors 校验的接口都会校验失败
3. 一些特殊的验证码机制，比如 google 机器人验证，在被内嵌的页面里面，也是直接报错的

只能说危害很低

## 附录: X-Frame-Bypass 的测试代码:

### index.html
```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale-1.0">
  <meta name="description" content="X-Frame-Bypass: Web Component extending IFrame to bypass X-Frame-Options: deny/sameorigin">
  <title>X-Frame-Bypass Web Component Demo</title>
  <style>
    html, body {
      margin: 0;
      padding: 0;
      height: 100%;
      overflow: hidden;
    }
    iframe {
      display: block;
      width: calc(100% - 40px);
      height: calc(100% - 40px);
      margin: 20px;
    }
    img {
      position: absolute;
      top: 0;
      right: 0;
    }
  </style>
  <script src="https://unpkg.com/@ungap/custom-elements-builtin"></script>
  <script src="x-frame-bypass.js" type="module"></script>
</head>
<body>
<iframe is="x-frame-bypass" src="https://test-example.com/my_test.html"></iframe>
</body>
</html>
```

### 2. x-frame-bypass.js
```javascript
customElements.define('x-frame-bypass', class extends HTMLIFrameElement {
	static get observedAttributes() {
		return ['src']
	}
	constructor () {
		super()
	}
	attributeChangedCallback () {
		this.load(this.src)
	}
	connectedCallback () {
		this.sandbox = '' + this.sandbox || 'allow-forms allow-modals allow-pointer-lock allow-popups allow-popups-to-escape-sandbox allow-presentation allow-same-origin allow-scripts allow-top-navigation-by-user-activation' // all except allow-top-navigation
	}
	load (url, options) {
		if (!url || !url.startsWith('http'))
			throw new Error(`X-Frame-Bypass src ${url} does not start with http(s)://`)
		console.log('X-Frame-Bypass loading:', url)
		this.srcdoc = `<html>
<head>
	<style>
	.loader {
		position: absolute;
		top: calc(50% - 25px);
		left: calc(50% - 25px);
		width: 50px;
		height: 50px;
		background-color: #333;
		border-radius: 50%;  
		animation: loader 1s infinite ease-in-out;
	}
	@keyframes loader {
		0% {
		transform: scale(0);
		}
		100% {
		transform: scale(1);
		opacity: 0;
		}
	}
	</style>
</head>
<body>
	<div class="loader"></div>
</body>
</html>`
		this.fetchProxy(url, options, 0).then(res => res.text()).then(data => {
			if (data)
				this.srcdoc = data.replace(/<head([^>]*)>/i, `<head$1>
	<base href="${url}">
	<script>
	// X-Frame-Bypass navigation event handlers
	document.addEventListener('click', e => {
		if (frameElement && document.activeElement && document.activeElement.href) {
			e.preventDefault()
			frameElement.load(document.activeElement.href)
		}
	})
	document.addEventListener('submit', e => {
		if (frameElement && document.activeElement && document.activeElement.form && document.activeElement.form.action) {
			e.preventDefault()
			if (document.activeElement.form.method === 'post')
				frameElement.load(document.activeElement.form.action, {method: 'post', body: new FormData(document.activeElement.form)})
			else
				frameElement.load(document.activeElement.form.action + '?' + new URLSearchParams(new FormData(document.activeElement.form)))
		}
	})
	</script>`)
		}).catch(e => console.error('Cannot load X-Frame-Bypass:', e))
	}
	fetchProxy (url, options, i) {
		const proxies = (options || {}).proxies || [
			'https://cors-anywhere.herokuapp.com/',
			'https://yacdn.org/proxy/',
			'https://api.codetabs.com/v1/proxy/?quest='
		]
		return fetch(proxies[i] + url, options).then(res => {
			if (!res.ok)
				throw new Error(`${res.status} ${res.statusText}`);
			return res
		}).catch(error => {
			if (i === proxies.length - 1)
				throw error
			return this.fetchProxy(url, options, i + 1)
		})
	}
}, {extends: 'iframe'})
```



