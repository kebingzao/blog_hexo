---
title: github建站系列(2) -- 创建个人blog主页
date: 2016-05-18 15:37:24
tags: github
categories: github建站系列
---
## 前言
通过 {% post_link github-site-1 %} 我们知道怎么把 github 仓库部署到 github pages上。 但是访问的时候，就要访问 http://kebingzao.github.io/hello-ghpages/  才能访问得到。

而我的原意是要在 github 上建一个自己的个人blog主页。 那么肯定是要可以访问 http://kebingzao.github.io/ 才能得到主页。

但是现在访问 http://kebingzao.github.io/ 发现是 404 页面
<!--more-->

![1](1.png)

说明我们没有这个仓库。根据 github 规则， 每个帐号只能有一个仓库来存放个人主页，而且仓库的名字必须是 `username/username.github.io`，这是特殊的命名约定。你可以通过 http://username.github.io 来访问你的个人主页。

通过向导很容易创建一个仓库，并测试成功。不过，同样的，没有博客的结构。需要注意的个人主页的网站内容是在master分支下的。

## 实操
所以第一步，就是要建立一个仓库， 名字为 kebingzao.github.io

![1](2.png)

这时候已经建完了，现在还差一个index文件。 因此接下来要拉到本地来，并新建一个index 文件。

![1](3.png)

新建index文件，并commit and push

![1](4.png)

最后刷新 http://kebingzao.github.io/ 就可以看到已经有了

![1](5.png)

## 总结
既然博客的项目和站点都好了，接下来就继续搭建。

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