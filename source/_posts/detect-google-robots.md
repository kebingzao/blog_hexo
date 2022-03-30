---
title: 怎么让你的 web 站点在被 google 收录的情况，移除 google 搜索结果
date: 2022-03-30 20:04:18
tags: 
- google
- seo
categories: seo 相关
---
## 前言
前段时间，有运营的同学反馈有一个不应该被 google 搜索收录的站点，被收录了， 并且在 google 搜索结果中，可以看到搜索结果。

## 检查 robots.txt
正常情况下，我们如果站点不想被 搜索引擎 爬虫进行收录的话， 是会在根目录下的 robots.txt 设置 disallow。

所以我检查了这个站点的 robots.txt 文件，看是否有配置这个文件，或者这个路由(有些是没有实体 txt 文件的，而是在 nginx 那边根据 `robots.txt` 路径来进行返回)
```javascript
[kbz@centos156 ~]$ curl https://example.com/robots.txt
User-agent:*
Disallow: /
```
检查了一下，确实有配置所有的爬虫都是 disallow。 那么为啥还会被收录呢， 会不会是 robots.txt 的语法有问题?

为了检查 robots.txt 的语法是否有问题，我就采用了一些第三方工具来进行验证

### 1. 使用百度的 robots 工具进行验证
刚开始用 [百度的 robots 工具](https://ziyuan.baidu.com/robots/index) 来检查 robots 文件是否存在以及合法性
<!--more-->
![](1.png)

检测结果是 robots 文件没问题

### 2. Google Search Console 平台提供的 robots.txt 提供的测试工具
后面改用更专业的 Google Search Console 平台提供的 [robots.txt 提供的测试工具](https://www.google.com/webmasters/tools/robots-testing-tool)来检测

> 这个要先在你的 google 账号那边先验证这个站点的所有权，才能用 (有多重验证方式，有 DNS TXT 的验证 和 html 根目录的验证方式)

> 每个人的验证都是分开，比如两个人都有这个站点的所有权，并且都采用 html 根目录的验证方式，那么就有可能在根目录下有两个 `googlexxx.html` 的文件，这个是不冲突的，因为他们属于不同的所有人

![](2.png)

发现我们的 robots.txt 文件配置都是没有问题的， 都是禁止 爬虫访问

那为啥还会被收录呢?

## google 收录的情况
从 google 的搜索结果页中， 我们可以看到虽然 google 有收录了这个站点，但是是没有具体的页面信息， 说明他并没有去访问到这个页面， 从而也说明我们的 robots 政策其实是有在生效的

![](3.png)

点击一下[了解原因](https://support.google.com/webmasters/answer/7489871?hl=zh-Hans) 这个按钮，我可以看到他的解释

![](4.png)

也就是说，google 虽然因为 robots.txt 的原因，不会去抓这个页面的内容， 但是一旦这个页面在其他页面有外链应用，他是可以顺着这个外链抓到这个页面的 url 的，这个 url 就会被收录进去。

所以最后的结果就是，站点页面的具体内容 google 爬虫看不到，但是这个页面的 url 因为有其他页面的外链应用，所以 google 还是收录了， 因此就会出现 google 搜索内容只有 url 而没有具体的页面信息

## 解决方式
其实针对这种方式是有解决方案的，有以下几种解决方案:

### 1. 使用 移除 工具来临时移除
临时移除，也就是使用 [“移除”工具](https://support.google.com/webmasters/answer/9689846?hl=zh-Hans), 来暂时阻止网站上的内容出现在搜索结果中,

具体操作很简单，在你在 Google Search Console 平台验证了这个站点的所有权之后，就可以在这个站点下，使用移除工具了，具体操作 [屏蔽网址](https://support.google.com/webmasters/answer/9689846?hl=zh-Hans)

![](5.png)

注意要选择以这个站点根目录为前缀的方式，这样子就相当于整个站点都屏蔽掉内容了。 这样子就可以暂时将这个站点的搜索结果都屏蔽掉了。

不过需要注意的是，这个不是永久删除哦， 有效期只有 6 个月，有几个重要提示
- **即便请求获得了批准，有效期也只会持续大约 6 个月**。在此期限结束后，您的信息就会重新显示在 Google 搜索结果中
- **屏蔽网址不会阻止 Google 抓取您的网页，只会阻止 Google 在搜索结果中显示该网址**。当您提出暂时屏蔽某个网址的请求后，如果该网址依然存在且未以其他方式（如 noindex 标记）遭到屏蔽，Google 就可以继续抓取该网址。因此，您的网页很有可能在您将其移除或对其使用密码保护之前被再次抓取或缓存，并在暂时隐藏期结束后重新出现在搜索结果中
- 在您使用此工具时，如果 Google 无法访问您的网址（错误 404、502/3），系统便会认为该网页已不复存在，您的屏蔽请求也将到期。之后在该网址下找到的任何网页都会被视作可在 Google 搜索结果中显示的新网页。

### 2. 通过在网页中添加 noindex 标记来永久移除内容
如果要永久移除内容，那么有几种方式
1. **移除或更新您网页上的内容**。这是一种最为安全的方式，可防止您的信息显示在可能不遵循 noindex 标记的其他搜索引擎中，还可确保其他人无法访问您的网页。
2. **通过密码保护您的网页**。通过限制对网页的访问，可让合适的用户查看您的网页，同时阻止 Googlebot 和其他网页抓取工具访问该网页。
3. **向网页中添加 noindex 标记**。noindex 标记仅会阻止您的网页显示在 Google 搜索结果中。用户和其他不支持 noindex 的搜索引擎仍可访问您的网页。

上面三种方式，只有最后一种 **向网页中添加 noindex 标记** 是符合我们的需求的

而且如果要在网页中添加 noindex 标记，那么有个前提，就是这些页面不能使用 robots.txt 屏蔽，要保证 google 机器人可以抓到这些页面，从而使得 noindex 生效。

这边[官方的解释](https://developers.google.com/search/docs/advanced/crawling/block-indexing?hl=zh-cn)是:
```javascript
重要提示：为让 noindex 指令生效，网页或资源不得被 robots.txt 文件屏蔽，并且必须能被抓取工具访问。
如果该网页被 robots.txt 文件屏蔽或抓取工具无法访问该网页，那么抓取工具将永远无法看到 noindex 指令，
因此该网页可能仍会显示在搜索结果中，例如，如果有其他网页链接到该网页的情况。
```

所以我们要永久移除搜索结果的话，就要分两步走:
1. 允许 google 机器人抓取 我们站点的页面 (事实上应该所有的机器人都不要索引)
2. 站点页面都加上 noindex 的 meta 标签

```javascript
<meta name="robots" content="noindex">
```

所以我们的 robots.txt 文件就要从原先的
```javascript
User-agent:*
Disallow: /
```
改成要允许 google 的机器人可以通过, 具体语法不难，我们可以在 Google Search Console 的 robots.txt 进行测试，然后没问题之后，再同步到站点的 robots.txt 中 (因为 google 对不同类型的机器人都好几个，干脆都开放了)
```javascript
User-agent: Googlebot
User-agent: Googlebot-News
User-agent: Googlebot-Image
User-agent: Googlebot-Mobile
User-agent: Adsbot-Google
Allow: /

User-agent: *
Disallow: /
```

![](6.png)

可以看到改成这样子，就会变成 google 相关的爬虫都可以过了，但是其他的爬虫还是被禁止。 将这一份更新到站点根目录的 robots.txt 中。

然后同时将这个站点的 html 页面的 meta 标签都补上
```javascript
<meta name="robots" content="noindex">
```

这样子就可以永久移除搜索内容了

### 3. 暂时移除 + 移除外链
之所以要用 noindex 的方式是因为外链已经被收录了， 所以如果可以找到引用这个站点的外链页面的话，可以通过 暂时移除+清除外链的方式 来达到永久删除的情况 (不需要走 noindex 的方式)。

因为暂时移除的 6 个月的有效期之后，google 会重新抓取，这时候因为已经没有对应的外链了，所以就不会再收录了。

至于清除外链的方式，有两种:
1. 如果是我们自己站点引用的外链，在该站点的 robots.txt 将这个页面设置为 disallow， 禁止爬虫收录
2. 如果是别人站点引用我们的外链，就可以在 google 后台提交移除外链， 正常我们可以在 Google Search Console 那边的 `链接数量` 查看应用该站点的外部链接数量，然后点击 `导出外部连接` 来导出这些外链

![](7.png)

类似于这样子 (只保留主域名，后面的 path 不需要，防止后面 path 修改的时候，还要再添加):
```javascript
https://appadvice.com/
https://apps_apple_com.iframe.weship2you.com/
https://nilesilo.com/
https://www.bethanne.net/
https://www.linkedbd.com/
https://www.tecplac.com/
https://www.tecupdate.com/
```

然后接下来我们通过 Google Search Console 的这个工具 [拒绝指向您网站的链接](https://www.google.com/webmasters/tools/disavow-links-main)

然后上传我们导出的外站外链的 url，

![](8.png)

点击上传

![](9.png)

这样子我们就可以将这些外站的外链都否认掉了，根据 Google Search Console官方说明，Google需要花几周的时间去处理上传的文件，当Google重新抓取网页时会将您上传的清单整合进Google索引当中。

## 总结
其实最好的方式就是用 robots.txt 全部 disallow 掉，同时通过暂时移除 + 移除暴露的外链 的方式来处理。

但是有时候一些垃圾站如果经常用外链收录你的站点的话，经常移除外链的话也很麻烦，就可以采用 noindex 的方式，直接让 google 爬虫不收录

---

## 参考资料
- [使用 robots.txt 测试工具测试 robots.txt](https://support.google.com/webmasters/answer/6062598?hl=zh-Hans)
- [“移除”工具-暂时阻止网站上的内容出现在搜索结果中](https://support.google.com/webmasters/answer/9689846?hl=zh-Hans)
- [从 Google 搜索结果中移除您网站上托管的网页](https://developers.google.com/search/docs/advanced/crawling/remove-information?hl=zh-cn#i-control-the-web-page)
- [漫游器元标记、data-nosnippet 和 X-Robots-Tag 规范](https://developers.google.com/search/docs/advanced/robots/robots_meta_tag?hl=zh-cn#robotsmeta)
- [使用 noindex 阻止搜索引擎编入索引](https://developers.google.com/search/docs/advanced/crawling/block-indexing?hl=zh-cn)
- [如何清理垃圾外链](https://www.seobroc.com/garbage-chain/)
- [百度 robots 检测工具](https://ziyuan.baidu.com/robots/index)
- [robots.txt 简介](https://developers.google.com/search/docs/advanced/robots/intro?hl=zh-cn)
- [创建 robots.txt 文件](https://developers.google.com/search/docs/advanced/robots/create-robots-txt?hl=zh-cn)