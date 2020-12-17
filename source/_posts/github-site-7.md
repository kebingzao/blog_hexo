---
title: github建站系列(7) -- 安装评论插件 DISQUS
date: 2018-05-05 16:38:24
tags: github
categories: github建站系列
---
## 前言
接下来我们安装一下 next 这个主题的一些第三方服务集成，包括评论，阅读量，搜索等等。 这边有很详细的文档 [第三方服务集成](https://theme-next.iissnan.com/third-party-services.html)

而我们这一次先安装 评论插件 DISQUS

## 安装
按照文档来，编辑 主题配置文件 _config.yml， 将 disqus 下的 enable 设定为 true，同时提供您的 shortname。count 用于指定是否显示评论数量。
```text
# Disqus
disqus:
  enable: true
  shortname: zach-2
  count: true
```
而这个 shortname 就是要先在 disqus 绑定一个站点，这时候就会生成一个 shortname 了，本例就是 `zach-2`
<!--more-->

![1](1.png)

然后就可以得到这个 shortname， 然后填进去就行了, 这时候在文章的底部就会显示 disqus 的评论模块

![1](2.png)

这样子评论模块就接好了。

## 总结
评论模块接好了，接下来我们增加阅读次数的插件。

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


