---
title: github建站系列(17) -- 为你的 blog 添加google adsence 广告
date: 2020-12-07 17:37:24
tags: github
categories: github建站系列
---
## 前言
不要问为啥要添加广告，在写博客的同时还有可能有钱赚，何乐为不为？

至于为啥是 google adsence，而不是百度广告联盟之类的，是因为我这个域名并没有在国内备案， 国内广告联盟接不了。

## 操作
### 1. 首先要先申请一个 [google adsense](https://www.google.cn/adsense/start/) 账号
### 2. 粘贴广告代码
注册账号成功之后，进入页面可以看到有广告代码
<!--more-->

![1](1.png)

```text
<script data-ad-client="ca-pub-9657591565519636" async src="https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js"></script>
```

针对 Next Theme，可以复制代码到 `themes\next\layout\_partials\head.swig` 中任意一个script块下。 我直接放到 head 下面

![1](2.png)

接下来就重新构建部署上去。 一旦将代码部署到线上之后。就可以点击 Done，让 google 进行审核

![1](3.png)

![1](4.png)

这个等待会非常的久，虽然两天就会回复你，但是以我的经历来看，都是被拒了十几次，终于在我锲而不舍的重复申请下，一个月后，终于申请通过了(这种速度不知道是不是跟疫情有关系)。

### 3.设置广告
申请成功之后，到首页，可以看到页面已经变了

![1](5.png)

为了省事，我选择设置自动广告
```text
自动广告是 google AdSense 近来提供的一种广告形式，它能够通过分析你的博客布局结构，自定义的在你的网站中插入合适的广告，无论是内容，还是广告尺寸，
都是完全契合网站内容本身的，算是一种比较高质量的广告。
打开 AdSense 首页，然后转到广告。您可以在概览中为各个网站设置自动广告。
如果您的网站已经启用自动广告，在审核通过的几个小时内，您便会在网站上看到相关的广告，并开始累积收入
(之前插入的检验代码，接入了 google AdSense 的自动广告)。
```

这种广告投放的几率比较小，在PC端效率比较低，如果你的网站支持移动端查看的话，会自动投放移动端自适应的广告。

![1](6.png)

设置自动广告，我们就不需要再调整代码了。 直接查看效果，然后点击开启即可。

![1](7.png)

自动广告会给你显示几个显示的广告位，有一些位置会很恶心，比如这个移动端的顶部广告

![1](8.png)

太影响体验了，直接 remove 掉，不要， 最后点击 `apply to size` 应用一下

![1](9.png)

这时候自动广告就生效了。

![1](10.png)

这样子自动广告就启动了。 等个几个小时，应该就可以生效了。 不过有点坑的是，这些位置都有点奇怪。 其实 侧边栏 固定会比较好，不过先体验一下，自动广告的效果吧。

回到首页，就可以看到报告了

![1](11.png)

每周大概有 57 次的浏览， 流量好少 ？？？ 接下来慢慢观察吧

而且看了一下提现规则， 要 800 港币才能提现， 我太难了。

![1](12.png)

等了几个小时之后，终于发现有广告了

![1](13.png)

这个广告位置放的还可以，不会影响阅读体验。

## 总结
google adsence 虽然接入了，但是很多优化细节其实都没有处理， 再加上流量还不够， 要提现得等到猴年马月了。 纯粹当了玩票的吧。 后面有时间再慢慢优化这东西。

---
参考文档
- [Hexo Next 接入 google AdSense 广告](https://www.cnblogs.com/DHUtoBUAA/p/12283738.html)
- [hexo博客next主题添加google adsense(亲测可用)](https://juejin.cn/post/6844903805264330765)

---
github 建个人站点系列文章:
{% post_link github-site-1 %}
{% post_link github-site-2 %}
{% post_link github-site-3 %}
{% post_link github-site-4 %}
{% post_link github-site-5 %}
{% post_link github-site-6 %}
{% post_link github-site-7 %}
{% post_link github-site-8 %}
{% post_link github-site-9 %}
{% post_link github-site-10 %}
{% post_link github-site-11 %}
{% post_link github-site-12 %}
{% post_link github-site-13 %}
{% post_link github-site-14 %}
{% post_link github-site-15 %}
{% post_link github-site-16 %}
{% post_link github-site-17 %}