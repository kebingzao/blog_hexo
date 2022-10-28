---
title: github建站系列 -- github 部署 demo 演示的地址
date: 2018-05-03 17:45:59
tags: github
categories: github建站系列
---
## 前言
通过 {% post_link github-site-4 %},  其实我们已经将 blog 部署在 github 服务上，并且也有绑定 kebingzao.com 域名。

但是平时除了用写文章的方式来讲解技术之外，其实很多时候都需要有一个 demo 来演示自己写的代码是更直观的。

所以我就打算将我接下来用来演示 demo 的项目，也部署在 github 上
<!--more-->
## 步骤
具体步骤如下

### 1. 首先在github 上创建一个项目，叫 html5Demo

![](1.png)

注意目录结构，docs 目录下面才是逻辑的代码， 可以理解为 docs 相当于 webserver 的根目录(类似于 Apache 的 www 目录)。

![](2.png)

### 2. 接下来点击设置 Setting，然后 GitHub Pages 的配置
source 选择 master branch/docs folder, 然后下面不用选模板， 直接点击右边的 save

![](3.png)

### 3. 接下来等待几秒钟，等github 部署一下，然后就可以点击查看了

![](4.png)

这样就完成了，接下来如果要迭代的话，只需要修改 docs 目录下就行了

![](5.png)



