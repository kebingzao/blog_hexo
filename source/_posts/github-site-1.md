---
title: github建站系列(1) -- 将你的github仓库部署到github pages
date: 2016-05-17 15:37:24
tags: github
categories: github建站系列
---
## 前言
随着做技术的越来越深入，也想搞个个人站点来存放自己的一些技术积累， 顺便偶尔发发牢骚。 本来想租个服务器的，不过后面看到有人可以将个人站点直接放到 github 上，我觉得这个好省事， 所以也想搞一搞， 关于 github page ，官方也有详细的文档 [GitHub Pages](https://pages.github.com/), 所以就简单实践一下。 

## 实操
### 1. 新建一个github仓库"hello-ghpages"
<!--more-->

![1](1.png)

### 2. 点击setting，到项目设置页面

![1](2.png)

点击 `Launch automatic page generator` 按钮。

![1](3.png)

可以看到默认用 markdown 写的。 点击下面 Continue to layouts

### 3. 选择模板
接下来选择模板

![1](4.png)

选择一个模板，点击 `publish page` 按钮，就直接发布上去了。 如果点击 edit 按钮，直接回退到编辑界面。

接下来就等一下，让github 部署一下。（不用30秒）

![1](5.png)

可以看到已经生效了。

### 4. 修改项目代码
接下来回到 这个项目，然后切换到 `gh-pages` 分支，就可以看到对应的html 文件了。

![1](6.png)

如果要改模板的话，直接 clone 下来， 然后切换到 gh-pages 分支，然后删掉其他的，只剩一个简单的index文件，然后文件里面只剩下 hello pages。

首先 clone 下来

![1](7.png)

切换到具体的文件夹，并且切换到 gh-pages 分支（默认master分支）

![1](8.png)

接下来删掉所有文件，并添加一个 index 文件，最后 commit 并且 push 到 gh-pages 分支里面去。

![1](9.png)

最后查看 http://kebingzao.github.io/hello-ghpages/

![1](10.png)

发现已经改过来了。 gh-pages 分支只剩下 index.html 这一个文件

![1](11.png)

## 总结
这样子就将我们的某一个仓库部署到 github pages。 接下来就开始创建自己的个人博客。


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




