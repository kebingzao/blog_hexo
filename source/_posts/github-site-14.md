---
title: github建站系列(14) -- NexT 修改内容区域的宽度
date: 2020-06-22 16:37:24
tags: github
categories: github建站系列
---
## 前言
我之前在 pc 上浏览我的 blog 的时候，发现旁边留白太多了，这样在浏览代码块时经常要滚动滚动条才能阅读完整，体验不是很好，同时觉得也不太美观。 所以想调整内容区域的宽度。

我参照这个文章调整: [NexT | 修改内容区域的宽度](https://blog.zuiyu1818.cn/posts/NexT_codewidth.html)

## 操作
NexT 对于内容的宽度的设定如下：
- 700px，当屏幕宽度 < 1600px
- 900px，当屏幕宽度 >= 1600px
- 移动设备下，宽度自适应

如果需要修改内容的宽度，同样需要编辑样式文件。 而样式文件的位置在于
- 主题的布局定义 Hexo/themes/next/source/css/_schemes/Picses/_layout.styl
- 样式的用户配置 Hexo/themes/next/source/css/_custom/custom.styl

<!--more-->
所有的修改原则上尽量不要变动源代码，因此我们编辑用户文件（推荐），新增内容。 

因为我们用的是 pisces 主题，所以直接在 `Hexo/themes/next/source/css/_custom/custom.styl`  补上内容
```text
// Custom styles.
.header{
  width: 70%;
  +tablet() {
    width: 100%;
  }
  +mobile() {
    width: 100%;
  }
}
.container .main-inner {
  width: 70%;
  +tablet() {
    width: 100%;
  }
  +mobile() {
    width: 100%;
  }
}
.content-wrap {
  width: calc(100% - 260px);
  +tablet() {
    width: 100%;
  }
  +mobile() {
    width: 100%;
  }
}
```
这样子就变大了, 从原来的

![1](1.png)

变成

![1](2.png)

果然宽很多。

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