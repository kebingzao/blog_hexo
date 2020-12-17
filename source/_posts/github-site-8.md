---
title: github建站系列(8) -- 增加阅读次数
date: 2018-05-05 16:47:24
tags: github
categories: github建站系列
---
## 前言
接下来我们为 blog 添加增加阅读次数的功能。 我们采用 leanCloud 这个第三方服务， 具体配置文档 [为NexT主题添加文章阅读量统计功能](https://notes.doublemine.me/2015-10-21-%E4%B8%BANexT%E4%B8%BB%E9%A2%98%E6%B7%BB%E5%8A%A0%E6%96%87%E7%AB%A0%E9%98%85%E8%AF%BB%E9%87%8F%E7%BB%9F%E8%AE%A1%E5%8A%9F%E8%83%BD.html#%E9%85%8D%E7%BD%AELeanCloud)

## 操作
首先我们要先到 [leancloud](https://www.leancloud.cn/) 注册一个账号，免费的，然后新建一个叫 blog 的应用
<!--more-->

![1](1.png)

接下来点击进去， 创建一个 class

![1](2.png)

点击设置，这时候就可以看到 appId 和 appKey了

![1](3.png)

复制 AppID 以及 AppKey 并在 NexT 主题的 _config.yml 文件中我们相应的位置填入即可，正确配置之后文件内容像这个样子:
```text
leancloud_visitors:
  enable: true
  app_id: HIV4XRdKJXB0eaccccjiVNaK-ccczoHsz
  app_key: zN3v8grxW0Q8eXwNlEnnnnWrF
```

这个时候重新生成部署 Hexo 博客，应该就可以正常使用文章阅读量统计的功能了。

需要特别说明的是：记录文章访问量的唯一标识符是文章的`发布日期`以及`文章的标题`，因此请确保这两个数值组合的唯一性，如果你更改了这两个数值，会造成文章阅读数值的清零重计。

因为这个 id 和 key是直接暴露出来的，接下来就要设置域名安全权限：

![1](4.png)

这样就可以了， 重启 hexo， 放到线上去就有了。

![1](5.png)

## 总结
这样子文章的阅读量就有统计了。

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






