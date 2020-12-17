---
title: github建站系列(13) -- 域名加 ssl
date: 2018-06-06 16:37:24
tags: github
categories: github建站系列
---
## 前言
之前有考虑过为了更安全，给 kebingzao.com 加上 https， 看到很多人的站点都是用的是 `Let's Encrypt` 免费的 ssl 证书， 不过有效期比较短，就 90天左右。

![1](1.png)

不过这个是免费的，因此我也想去搞一个，https://letsencrypt.org/， 但是我后面看了一下，因为我这个项目是托管到 github 的 page 上面的。 这样就意味着，没法使用这种证书了。原因如下：
```text
默认情况下使用GitHub Pages的给定域名则支持http和https两种协议，但是如果使用自定义域名的话，则只能通过http://访问，
也就是说我们在Github上搭建 Hexo 或Jekyll 主题博客后，通过CNAME绑定个人域名后，
我们只能通过http://域名来访问。如果访问https://XXX.github.io/(即原来的GitHub Pages域名)将会被重定向到我们的自定义域名。
但若直接访问https://我们的自定义域名，浏览器会报SSL_DOMAIN_NOT_MATCHED警告。
```
<!--more-->
那么怎么给自己的域名加上 https 呢？这个时候就需要使用第三方网站的证书了。而GitHub Pages并不支持上传SSL证书。后面网上找了一下，大部分都是用 CF 来提供加速, 不过看了一下，发现好像不需要再去用 CloudFlare 走代理。现在的 github 好像已经支持了, 在我的项目的配置项，就有这个选项了，直接打钩就可以了。

## 操作
进入项目配置页面，有一个 `Enforce HTTPS` 的勾选框

![1](2.png)

但是我发现，好像打钩不了，应该是哪里配置错了。后面我查下了一下 github 的 blog: [Custom domains on GitHub Pages gain support for HTTPS](https://github.blog/2018-05-01-github-pages-custom-domains-https/)

原来是需要把DNS调整到以下IP地址：
```text
185.199.108.153
185.199.109.153
185.199.110.153
185.199.111.153
```
如果原本已经有自定义域名的， 就需要在设置里先移除域名再改回来触发一下调整到HTTPS的机制，之后等证书颁发完毕就可以启用HTTPS啦~。 [官方文档操作文档](https://help.github.com/articles/setting-up-an-apex-domain/)

所以我们要先在 DNSPOT 上先把原来的指向改成上面这四个，原先的指向是:

![1](3.png)

后面改为上面那四个，但是我发现了一个很尴尬的问题，就是上面的那四个 A 记录， 我免费的 dnspot，只能设置两条，超过两条，就要购买套餐或者购买A负载均衡记录

![1](4.png)

所以后面我只能先设置为其中两条：

![1](5.png)

后面我重新在 github pages 上设置了一下 custom domain， 发现 `enforce https` 就可以点了

![1](6.png)

过了一阵子，果然生效了

![1](7.png)

现在已经支持 https了， 不过我看了一下， github page 的 custom domain 的 https，也是用的是 let Encrypt 免费证书。不过好处就是 90天到期了，不用我们自己续，而是github 会帮我们续。

![1](8.png)

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





