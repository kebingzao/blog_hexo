---
title: github建站系列(11) -- 对首页的部分长文章增加了阅读全文的按钮
date: 2018-05-09 16:37:24
tags: github
categories: github建站系列
---
## 前言
发现写文章的时候，如果文章很长的时候，会直接显示在首页。所以我们要加个 `阅读全文`的按钮, 这样子在首页就只显示一部分。 参考文章 [Hexo-设置阅读全文](https://www.jianshu.com/p/78c218f9d1e7)。

我们用最简单的方式，就是在想要插入按钮的哪个地方，加上 `<!—more—>`, 这样就行了

## 操作
```text
这篇文章讲的还是很详细的，其方法就是用 image src 请求，抛送 ga 请求
<!--more-->
当然ga请求的参数有很多，我这边大概列了，本次需要的字段的说明，其中 tid 就是GA统计的哪个id值
```
那么展示效果就是:
<!--more-->

![1](1.png)

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