---
title: github建站系列(16) -- 为你的 blog 添加看板娘
date: 2020-09-11 17:37:24
tags: github
categories: github建站系列
---
## 前言
前段时间又看到别人家的 blog 有看板娘了， 觉得挺好看的。 所以也打算搞起来。

参照这个第三方库来操作也很简单。 [Live2D Widget](https://github.com/stevenjoezhang/live2d-widget)

## 操作
### 1. 下载这个库到对应的目录
```text
$   git clone "https://github.com/stevenjoezhang/live2d-widget" themes/next/source/live2d-widget
Cloning into 'themes/next/source/live2d-widget'...
remote: Enumerating objects: 423, done.
remote: Total 423 (delta 0), reused 0 (delta 0), pack-reused 423
Receiving objects: 100% (423/423), 960.57 KiB | 30.00 KiB/s, done.
Resolving deltas: 100% (263/263), done.
```
<!--more-->
### 2. 修改 autoload.js 文件
修改 `themes/next/source/live2d-widget` 下的 `autoload.js` 文件, 改成本地资源路径

![1](1.png)

### 3. 最后添加资源文件
最后添加资源文件， 并且放到 head 下面， 我是把 `jquery` 和 `font-awesome.min.css` 直接下载到 本地来， 在  `next/layout/_layout.swig`

![1](2.png)

### 4. 最后执行 hexo s 重新编译即可

![1](3.png)

这样子就完成了。

如果想进行更详细的配置的话，可以根据 `Live2D Widget` 这个库的文档进行更详细的，可定制化的配置。 

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
