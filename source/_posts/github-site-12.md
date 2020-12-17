---
title: github建站系列(12) -- 文章底下增加 copyright
date: 2018-05-10 16:37:24
tags: github
categories: github建站系列
---
## 前言
写 blog 文章的时候，我们一般都会想在文章底下增加 copyright 的版权声明。要求转载者要注明出处， 毕竟是自己的心血，也不能被白蹭。

## 操作
这个其实很简单，因为我们用的是 hexo 的 next 主题。 它是有带这个功能的，只不过默认不开启而已。

所以我们只要在 `theme/next/_config.yml` 的 `post_copyright` 改为 true 就行了，其他都不用改

```text
# Declare license on posts
post_copyright:
  enable: true
  license: CC BY-NC-SA 3.0
  license_url: https://creativecommons.org/licenses/by-nc-sa/3.0/
```
这样就可以了，刷新一下：
<!--more-->

![1](1.png)

这个就是我们想要的结果。

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