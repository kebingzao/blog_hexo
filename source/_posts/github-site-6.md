---
title: github建站系列(6) -- 开始写文章了
date: 2018-05-05 16:37:24
tags: github
categories: github建站系列
---
## 前言
终于，终于，要写文章了。 hexo 是用 markdown 格式来写的，具体可以看 [hexo writing](https://hexo.io/zh-cn/docs/writing.html)

## 实操
可以通过
```text
hexo new xxx
```
来生成一个文章
```text
admin@admin-PC MINGW64 /f/airdroid_code/github/blog_hexo (master)
$ hexo new "test post"

INFO  Created: F:\airdroid_code\github\blog_hexo\source\_posts\test-post.md
```
这时候就会生成一个 `test-post.md` 这时候只要在这个md里面写文章就行了
<!--more-->

![1](1.png)

当然在写的过程中，要一直开着这个服务才行：

![1](2.png)

写完之后， 因为涉及到两个项目，一个是源代码项目 `blog_hexo`, 一个是构建的 blog 的静态资源文件项目 `kebingzao.github.io`, 所以当文章写完之后，流程是这样子的:
1. 因为是在 `blog_hexo` 这个项目里面写的，所以写完之后，先把修改提交上远端的 master
2. 接下来就是执行 `hexo g` 进行静态文件编译， 编译完之后，再通过 `hexo deploy` 上传到 `kebingzao.github.io`的远程资源库

这样子这两个项目就都提交上去了。 过一会儿就可以到 kebingzao.com 查看新写的文章了。

## 注意
不过有时候会发现 `hexo deploy` 上去之后，还是旧的，那么就要先用 `hexo clean` 刷新一下，然后再 `hexo deploy` 上去。

不过这样一来， `kebingzao.github.io` 这个项目里面的 `readme` 和 `cname` 就又被覆盖了， 所以还得重新再补上去。

## 总结
终于开始写文章了，那么肯定有阅读量，评论之类的，所以接下来开始安装一些第三方插件

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


