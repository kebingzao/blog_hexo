---
title: 浅谈之 - HTML 的 meta 标签有多少种
date: 2020-09-04 13:39:40
tags: js
categories: 
- 前端相关
- 前端浅谈系列
---
## 前言
meta 标签是 html 标记 head 区的一个关键标签，它位于HTML文档的 <head> 和 <title> 之间（有些也不是在<head>和<title>之间）。它提供的信息虽然用户不可见，但却是文档的最基本的元信息。meta标签用来描述一个HTML网页文档的属性，例如作者、日期和时间、网页描述、关键词、页面刷新等。

## meta 标签结构
从官方文档来看 [<meta>: The Document-level Metadata element](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/meta), 这个标签有四个属性:
### charset
charset 属性规定 HTML 文档的字符编码。

charset 属性是 HTML5 中的新属性，且替换了
```text
<meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
```
仍然允许使用 http-equiv 属性来规定字符集，但是使用新方法可以减少代码量。

常用的值有:
- `UTF-8` - Unicode 字符编码
- `ISO-8859-1` - 拉丁字母表的字符编码

在理论上，可以使用任何字符编码，但并不是所有浏览器都能够理解它们。某种字符编码使用的范围越广，浏览器就越有可能理解它。 如需查看所有可用的字符编码，请访问 [IANA 字符集](http://www.iana.org/assignments/character-sets/character-sets.xhtml)。

### content
content 属性的内容是 htp-equiv 或 name 属性的值，具体取决于你用哪一个。(无法单独使用)

### http-equiv
该属性可以包含HTTP头的名称，属性的英文全称为 http-equivalent 。它定义了可以改变 server 和 user-agent 行为的指令。该指令的值在 content 属性内定义。

### name 
name 属性的定义是属于 document-level metadata，不能和以下属性同时设置： http-equiv 或 charset。 该元数据名称与 content 属性包含的值相关联。

接下来我们详细介绍一下 http-equiv 和 name 属性的值。都有很多值，不同的值表示不同的意思。


## http-equiv 的属性值
因为整个浏览器的历史跨度很大，有很多旧的 http-equiv 的值到了现代浏览器，其实已经废弃了， 我们先讲一下剩余的还有效的 http-equiv 的值。 具体文档可以看: [mozilla http-equiv](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/meta), 我们先来介绍文档中明确标有的属性:

### 1. content-security-policy
允许页面作者定义当前页面的内容策略。内容策略主要指定允许的服务器地址和脚本端点，这有助于防止 XSS(cross-site scripting) 攻击。

CSP 的实质就是白名单制度，开发者明确告诉客户端，哪些外部资源可以加载和执行，等同于提供白名单。它的实现和执行全部由浏览器完成，开发者只需提供配置。

CSP 大大增强了网页的安全性。攻击者即使发现了漏洞，也没法注入脚本，除非还控制了一台列入了白名单的可信主机。

两种方法可以启用 CSP。一种是通过 HTTP 头信息的Content-Security-Policy的字段。 即:
```text
Content-Security-Policy: script-src 'self'; object-src 'none';
style-src cdn.example.org third-party.org; child-src https:
```
![网络用图](1.jpg)

另一种是通过网页的 `<meta>` 标签。
```html
<meta http-equiv="Content-Security-Policy" content="script-src 'self'; object-src 'none'; style-src cdn.example.org third-party.org; child-src https:">
```
上面代码中，CSP 做了如下配置：
- 脚本：只信任当前域名
- <object>标签：不信任任何URL，即不加载任何资源
- 样式表：只信任cdn.example.org和third-party.org
- 框架（frame）：必须使用HTTPS协议加载
- 其他资源：没有限制

启用后，不符合 CSP 的外部资源就会被阻止加载, 浏览器的控制台就会报错。 

![网络用图](2.jpg)

具体更多关于 CSP 安全策略的，可以看 [Content Security Policy 入门教程](http://www.ruanyifeng.com/blog/2016/09/csp.html)

### 2.content-type
这个其实已经过时了，因为如果要设置字符编码的话，推荐使用元素上的charset属性。 

并且由于无法在 XHTML 或 HTML5 的 XHTML 序列化中更改文档类型 (文档类型只能是这个 `text/html`)， 因此不要使用将MIME类型设置为XHTML MIME类型。

所以这个属性其实没有用处了

### 3.default-style
指定要使用的首选样式表， 例如:
```html
<meta http-equiv="default-style" content="the document's preferred stylesheet">
```
上面 content 属性的值必须匹配同一文档中的一个 link 元素上的 title 属性的值，或者必须匹配同一文档中的一个 style 元素上的 title 属性的值。 这个属性其实很少用到。

### 4.x-ua-compatible
用于告知浏览器以何种版本来渲染页面。`X-UA-Compatible` 是针对 IE8 版本的一个特殊文件头标记，用于为 IE8 指定不同的页面渲染模式。 不过现在基本上 IE 系列的浏览器基本上都凉了，所以现在基本上都是这样子配的:
```text
<meta http-equiv="X-UA-Compatible" content="IE=Edge,chrome=1" >
```
这个意思就是如果 IE 有安装 Google Chrome Frame，那么就走安装的组件，如果没有就使用最高版本的 IE。

ps1: 针对IE 6，7，8等版本的浏览器插件 Google Chrome Frame，可以让用户的浏览器外观依然是IE的菜单和界面，但用户在浏览网页时，实际上使用的是Google Chrome浏览器内核。

ps2: 而且这东西主要是用于设置 IE 兼容性，现在只支持 "IE=edge" 这一个值，并且只有低于 2013 年发行的 IE 11 版本才需要设置它。

### 5.refresh
该指令指定：
1. 如果content属性只包含一个正整数，则表示该页面重新加载的秒数。
```text
<meta http-equiv="refresh" content="300">
```

2. 如果content属性包含一个正整数，后跟字符串'; url ='，那么表示当前页面XX秒后重定向到另一个有效的URL。
```text
<meta http-equiv='refresh' content='2;URL=http://www.github.com/'> //意思是2秒后跳转到github
```

以上这 5 个属性，就是 W3C 目前针对 meta 的 http-equiv 属性定义的规范值，但是除了这些属性之外，还有一些早期浏览器有兼容的属性，不过现在都已经过时了， 基本上现代浏览器都不再支持了，也稍微了解一下。

### 6. 已过时 http-equiv 的值
#### 1.Pragma (已过时)
禁止浏览器从本地计算机的缓存中访问页面内容。如：
```text
<meta http-equiv='Pragma' content='no-cache'>
```
#### 2.Expires (已过时)
可以用于设定网页的到期时间。一旦网页过期，必须到服务器上重新传输。
```text
<meta http-equiv="Expires" content="0" />
```
#### 3.Cache-Control (已过时)
指定请求和响应遵循的缓存机制。  一般常用的几个设置值:
- max-age（单位为s）指定设置缓存最大的有效时间，定义的是时间长短
- public 指定响应会被缓存，并且在多用户间共享
- private 响应只作为私有的缓存，不能在用户间共享
- no-cache 指定不缓存响应，表明资源不进行缓存
- no-store 绝对禁止缓存

关于 cache-control 的字段设置值，可以看 [浅谈之-浏览器缓存](https://kebingzao.com/2018/07/05/browser-cache/#Cache-Control)

```text
<meta http-equiv="Cache-Control" content="no-cache, no-store, must-revalidate" />
```

这三个属性在 HTML5 的时候就被设置为过期了，这是因为 W3C 推荐页面的缓存策略应该是要通过 HTTP 的 response header 来设置的，是属于服务端策略。 因此不建议在前端的页面中设置。

而且这个是 IE 时代的私有属性，在 IE9 以前支持的，而现在主流的 Chrome / Firefox / Safari，包括 IE9 ~ IE11 都不支持。 这个东西是 HTTP/1.0 时代的产物，因为 HTTP/1.0 里关于缓存的可设定太少了，而 HTTP/1.1 刚出来的时候还不是所有浏览器都支持，所以把这个玩意儿放到了 HTML 页面里了。如果你的页面需要兼容 IE 低版本，那么可以加上。

现在上哪找还只支持 HTTP/1.0 的 Web 服务器去啊，全都是 HTTP/1.1 了。而且随着 HTTPS 的普及，估计很快就该全面 HTTP/2 了。 所以现代浏览器几乎已经不再支持这种方式，如果是 IE9 更早的浏览器的页面要设置为不缓存的话，那么倒是可以这样子设置:
```text
<meta http-equiv="Cache-Control" content="no-cache, no-store, must-revalidate" />
<meta http-equiv="Pragma" content="no-cache" />
<meta http-equiv="Expires" content="0" />
```

#### 4.set-cookie (已过时)
定义页面的cookie，这个也是已经过时的属性了，现代浏览器也不会支持了，现在都是使用 HTTP 头的 Set-Cookie 替代

#### 5.Window-target (已过时)
强制页面在当前窗口以独立页面显示:
```text
<meta http-equiv="Window-target"content="_top">
```
这个也是过时了，现代浏览器根本不支持了。具体看 [Does Window-target meta tag work for busting frames?](https://stackoverflow.com/questions/1521195/does-window-target-meta-tag-work-for-busting-frames)

## name 的属性值
接下来我们介绍一下 name 的属性值，还是一样，先从 W3C 的官方文档先看: [Standard metadata names](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/meta/name), 根据 W3C 的规范来看， name 属性其实分为三大类:
- HTML 规范
- 其他的规范
- 其他值

这三块都在文档里面，某种程度上也都是标准， 现代浏览器都要支持。 但是除了这三块， 各种互联网大厂以及不同的浏览器厂家也都有自己的特殊的值。

### 1. HTML 规范的值
这一部分的内容主要是为了 SEO 相关的，尤其是 description 和 keywords，对 SEO 来说尤为重要。

#### 1.1 application-name
定义在网页中运行的应用程序的名称。 其实主要用于比较复杂的网页，比如 SPA 之类的

#### 1.2 author
用于标注网页作者

#### 1.3 description
包括一个关于页面内容的缩略而精准的描述。一些浏览器，如Firefox和Opera，会使用这个当做网页书签的默认描述。

#### 1.4 generator
用于标明网页是什么软件做的。

#### 1.5 keywords
用于告诉搜索引擎，你网页的关键字

#### 1.6 referrer
referrer 控制 document 发起的 Request 请求中附加的 [Referer HTTP header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referer)，相应的值在content中:

| value | description |
|---|---|
| 空字符串 | 默认值 `no-referrer-when-downgrade` |
|`no-referrer`| 不发送 HTTP Referer 头|
|`origin` | 发送 document 的 origin|
|`no-referrer-when-downgrade` | 这个是默认值，当请求安全级别下降时不发送 referrer。目前，只有一种情况会发生安全级别下降，即从 HTTPS 到 HTTP。HTTPS 到 HTTP 的资源引用和链接跳转都不会发送 referrer。|
|`same-origin`| 对于同源的链接和引用，会发送referrer，其他的不会。 |
|`origin-when-cross-origin` | 这个相当于 origin 和 same-origin 的 OR 合体。同源的链接和引用，会发送完全的 referrer 信息；但非同源链接和引用时，只发送源信息。|
|`strict-origin`| 这个相当于 origin 和 no-referrer-when-downgrade 的 AND 合体。即在安全级别下降时不发送 referrer, 安全级别未下降时发送源信息。(注意：这个是新加的标准，有些浏览器可能还不支持。)
|`strict-origin-when-cross-origin`| 同源的链接和引用，会发送 referrer。安全级别下降时不发送 referrer。其它情况下发送源信息。(注意：这个是新加的标准，有些浏览器可能还不支持。)
|`unsafe-URL`| 无论是否发生协议降级，无论是本站链接还是站外链接，统统都发送 Referrer 信息。正如其名，这是最宽松而最不安全的策略。|

其实这个属性还跟 web 安全有关系，甚至很多跟盗链相关的策略也跟他有关系， 这也是为啥它的 value 有那么多细致的安全的区别了，关于 referrer 跟安全的关系， 具体可以看 [web 安全之 - 页面禁用 referer(第三方站点的 referer 头部泄露重置密码链接)](https://kebingzao.com/2020/05/07/referrer-leakage/)

#### 1.7 theme-color
可以设置浏览器的地址栏或者是手机端的选项卡的颜色:
```text
<meta name="theme-color" content="#91D4DA">
```

### 2. 其他规范的值
#### 2.1 color-scheme
比如 css 色彩规范就有这个值，用于指定文档的配色方案， 最经常见就是网页的暗黑模式:
```text
<meta name="color-scheme" content="light dark">
```
不过这个是需要浏览器支持的，是属于还在完善中的标准。

#### 2.2 viewport
css 设备适配规范就有这个值，这个也是做手机端页面必须要用到的一个 meta 值，关于 viewport 的理解，可以看这一篇文章 [移动前端开发之viewport的深入理解](https://www.cnblogs.com/2050/p/3877280.html),  其中有关于 3 个 viewport 的理论:
- `layout viewport` ->  浏览器默认的viewport, 这个layout viewport 的宽度可以通过 `document.documentElement.clientWidth`来获取。
- `visual viewport` ->  浏览器可视区域的大小, visual viewport的宽度可以通过 `window.innerWidth` 来获取
- `ideal viewport`  ->  移动设备的理想viewport

| value | description |
|---|---|
| `width`|  设置 layout viewport 宽度，也可以用 `device-width` |
| `height`| 设置 layout viewport 的高度，也可以用 `device-height` |
| `initial-scale`| 设置页面的初始缩放值，为一个数字，可以带小数, 范围在 0.0 - 10.0 |
| `maximum-scale`| 允许用户的最大缩放值，为一个数字，可以带小数, 范围在 0.0 - 10.0 |
| `minimum-scale`| 允许用户的最小缩放值，为一个数字，可以带小数, 范围在 0.0 - 10.0 |
| `user-scalable` | 是否允许用户进行缩放，值为"no"或"yes", no 代表不允许，yes代表允许|
| `viewport-fit` | 控制文档是如何填充满屏幕的, 有三个值， `auto` 表示 此值不影响初始布局视图端口，并且整个web页面都是可查看的。 `contain` 表示视图端口按比例缩放，以适合显示内嵌的最大矩形。 `cover` 表示视图端口被缩放以填充设备显示。这个属性目前大部分的浏览器还没有兼容，具体看 [viewport-fit](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@viewport/viewport-fit)|

最常见的情况就是设置为:
```text
<meta name="viewport" content="width=device-width, initial-scale=1">
```
这样子的意思就是 要把当前的 viewport 宽度设为 ideal viewport 的宽度。

### 3. 其他合理的值
接下来介绍其他的官方属性值。
#### 3.1 creator
文档的创建者，可以是组织或者是机构，如果有多个，就写多个 meta 标签

#### 3.2 publisher
文档的发布者

#### 3.3 robots
搜索引擎的机器人策略，robots 用来告诉爬虫哪些页面需要索引，哪些页面不需要索引，具有以下值:

| value | description | Used By |
|---|---|---|
|`index`|  允许robot索引本页面（默认）| All|
|`noindex`| 不允许robot索引本页面 | All|
|`follow`| 允许搜索引擎继续通过此网页的链接索引搜索其它的网页（默认） | All|
|`nofollow`| 搜索引擎不继续通过此网页的链接索引搜索其它的网页 | All|
|`all`| 等同于 `index` +  `follow`|  Google|
|`none`| 等同于 `noindex` +  `nofollow` | Google|
|`noarchive`| 要求搜索引擎不缓存页面内容 | Google, Yahoo, Bing|
|`nosnippet`|	禁止在搜索引擎结果中显示该页面的任何描述。  | Google, Bing|
|`noimageindex`|  要求此页面不作为引用页面的索引图像的显示。| Google|
|`nocache`| 	和 `noarchive` 同义 | Bing|

如果规则比较复杂的话，直接在站点根目录使用 robots.txt 来描述规则会更好。


#### 3.4 googlebot
跟 `robots` 一样，google 的爬虫会参照这个

### 4.一些非官方的属性
上面的 1,2,3 都是 W3C 官方文档中的属性， 但是还有一些是互联网大厂或者是其他浏览器兼容的，但是这些属性就不能保证这些现代的浏览器都兼容了。
#### 4.1 google 规范的值
具体文档可以看这个 [Google 可以识别的特殊标记](https://support.google.com/webmasters/answer/79812?hl=zh-Hans), 除了上面列到的官方属性之外，这些的 name 大部分都是 google， 还有以下:
1. nositelinkssearchbox
```text
<meta name="google" content="nositelinkssearchbox" />
```
当用户搜索您的网站时，Google 搜索结果有时会显示一个供您网站专用的搜索框，以及其他直接指向您网站的链接。此标记用于告知 Google 不要显示站点链接搜索框。

2. notranslate
```text
<meta name="google" content="notranslate" />
```
如果 Google 发现网页内容所用的语言很可能不是用户想阅读的语言，则往往会在搜索结果中提供翻译链接。这样通常会让您有机会将独特而富有吸引力的内容提供给更多用户。不过，在某些情况中，您可能并不希望我们这样做。此元标记用于告知 Google 您不希望我们提供该网页的翻译。

3. nopagereadaloud
```text
<meta name="google" content="nopagereadaloud" />
```
禁止网络浏览器使用 Google 助理语音指令“阅读此页”和“阅读”来朗读已标记的网页。

4. google-site-verification
```text
<meta name="google-site-verification" content="..." />
```
您可以在网站的顶级网页上使用此标记，以向 Search Console 验证您对该网站的所有权。请注意，虽然“name”和“content”属性的值必须与提供给您的值完全匹配（包括大小写），但是否将标记从 XHTML 更改为 HTML，或者标记的格式是否与网页的格式相符，这些都无关紧要。

5. rating
```text
<meta name="rating" content="adult" />
```
将网页标记为包含成人内容，以表明网页应被安全搜索结果滤除。

#### 4.2 apple 规范的值
具体文档可以看: [Supported Meta Tags](https://developer.apple.com/library/archive/documentation/AppleApplications/Reference/SafariHTMLRef/Articles/MetaTags.html), 具有以下值, 都是只有 IOS 下才有用:

1. apple-mobile-web-app-capable
```text
<meta name="apple-mobile-web-app-capable" content="yes">
```
是否启用 WebApp 全屏模式

2. apple-mobile-web-app-status-bar-style
```text
<meta name="apple-mobile-web-app-status-bar-style" content="black">
```
设置状态栏的背景颜色,只有在 “apple-mobile-web-app-capable” content=”yes” 时生效

3. format-detection
```text
<meta name="format-detection" content="telephone=no">
```
禁止自动探测并格式化手机号码

#### 4.3 其他浏览器规范

## og 标签
og 是一种新的HTTP头部标记，（即Open Graph Protocol：开放内容协议）, 先看一下背景:
### og 背景
2010年F8会议上Facebook公布了Open Graph，把这种种不同的Graph连结起来，将形成Open Graph。

Open Graph通讯协定(Protocol)本身是一种制定一套 Metatags 的规格，用来标注你的页面，告诉我们你的网页代表哪一类型的现实世界物件。另一伙伴网站，即Amazon旗下的Internet Movie Database(IMDb)，将用这个Open Graph Protocol为每一部电影标注页面。按下IMDb上的“赞”按钮，就会自动把那部电影加入Facebook使用者profile中的“最爱的电影”。

Facebook已和Yahoo、Twitter合作采用OAuth 2.0认证标准。Graph API翻新了Facebook的平台程序代码，让Facebook里的每个物件都拥有独特的ID。通过Open Graph把其他社交网站建构的网络给连接起来，将创造一个更聪明、更与社交连接、更个人化也更具语意意识的网络。

Open Graph最让人津津乐道的是“喜欢”(Like)按钮，此按钮安装在伙伴网站，可立即用来表示认同。“活动”(Activity streams)外挂 ，让Facebook使用者友人所从事的各种活动都列在那个第三方网站上。“推荐”(Recommendations)外挂则向使用者提供备受建议的内容，“不只是十大最多人用电子邮件转寄的文章，这是真正超强的推荐”。“社交”(Social bar)可提供整合为一的社交体验，把“喜欢”按钮、Facebook聊天 、和友人名单资讯都整合起来，功能与Google Friend Connect或Meebo chat工具条相似。

Facebook新版本Graph API意味着Facebook上任何一个页面都会有独立的ID，用户可以成为某一页面的粉丝。该项功能将会使Facebook的每一页面连接成为一个整体。

### og 属性
og 具有以下属性: 
- og:title -> 标题
- og:type  -> 类型, 常用值:article book movie
- og:image -> 略缩图地址
- og:url -> 页面地址
- og:description -> 页面的简单描述
- og:site_name -> 页面所在网站名
- og:videosrc -> 视频或者Flash地址
- og:audiosrc -> 音频地址

### og 语法
og 标签虽然是 meta 标签，但是他不用 name， 而是用另一个非 W3C 规范的一个属性 `property` 来实现语法:
```text
<meta property=”og:title” content=”{dede:field.title/} ” />
```
比如某一个站点是这样子设置的:
```text
<meta property="og:url" content="https://web.xxx.com"/>
<meta property="og:type" content="article"/>
<meta property="og:description" content="xxxxxxxxx"/>
<meta property="og:image" content="https://xxx.xxx.com/xxx.png"/>
```

### og 标签的好处
og 标签最大的好处就是对 SEO 的优化，都知道搜索引擎机器人爬取的是我们的页面，即html代码，meta 信息尤为关注，所以我们增加的 og meta 标签是可以别搜索引擎发现并评估权重的，也就是说你将原有 meta 信息优化手段同时使用到 og:title 这种属性值当中，加强 meta 信息优化内容；对于权重提升和排名还是很有利的。

### og 标签的兼容性
有些 seo 检测工具会检测到 Meta Property=og 报错，这个不管他。 



---

## 参考资料
- [<meta>: The Document-level Metadata element](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/meta)
- [HTML <meta> charset 属性](https://www.runoob.com/tags/att-meta-charset.html)
- [Content Security Policy 入门教程](http://www.ruanyifeng.com/blog/2016/09/csp.html)
- [Disable browser caching with meta HTML tags](http://cristian.sulea.net/blog/disable-browser-caching-with-meta-html-tags/)
- [移动前端开发之viewport的深入理解](https://www.cnblogs.com/2050/p/3877280.html)
- [【SEO技术】网站meta标签og属性优化教程](http://www.link356.com/seojishu/20.html)
