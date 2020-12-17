---
title: github建站系列(4) -- 绑定 kebingzao.com
date: 2016-05-19 16:37:24
tags: github
categories: github建站系列
---
## 前言
通过 {% post_link github-site-3 %} 已经创建了一个自己的blog，而且界面也不丑， 但是域名还是 github 的域名，不是我自己的域名， 接下来将 kebingzao.com 指定到 这个blog页面。

## 实操
是在 blog_hexo 项目下，新建一个 CNAME 文件，放到 source 文件夹中
<!--more-->

![1](1.png)

内容就是要cname 的地址 

![1](5.png)

然后用 `hexo d` 部署到 kebingzao.github.io 项目上

![1](3.png)

接下来登录 DNSPod， 然后修改 A指向的ip。

![1](4.png)

原来指向的是 tumblr 的 ip，现在改为 github 的ip

![1](6.png)

这样就可以了， 访问下。

![1](7.png)

发现已经生效了。

## 总结
好了，个人域名也绑定好了，接下来是不是应该写文章了。

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
