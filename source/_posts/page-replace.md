---
title: 实现站点同一个页面但是地址栏不同的情况
date: 2019-07-24 17:09:56
tags: html
categories: 前端相关
---
## 前言
之前项目有做了一个企业版的功能，但是要在官网上推广，所以我们就在官网增加了一个页面，叫做**bizHome.html**的页面，专门用来介绍这个企业版的功能。
但是有一天商务过来说，能不能把这个页面 www.example.com/bizHome.html 变成这个页面 www.example.com/business ，这样会比较好推广，而且也比较正式，但是线上放出去的这些 bizHome.html 的外链也要可以访问。
<!--more-->
## 解决
所以原来的bizHome.html是不能动的。 而且还要在项目根目录增加一个 business 目录，里面的 index.html 文件要跟 bizHome.html 的内容一样。当然我们不可能直接复制代码，那样太 low 了，而且不好维护。 有考虑过直接在 business/index.html 直接将 location.href 指向 bizHome.html 这个页面。这样也是可以的，但是这样子地址栏就会变了，多了一层跳转，变成 bizHome.html, 感觉体验也不太好。
后面的解决方法，就是通过 meta 标签的 refresh 操作，将 content 指向 bizHome.html，这样子既可以实现显示 bizHome.html 的内容，而且地址栏还不会变。代码如下：
```html
<!DOCTYPE html>
<html>
<head lang="en">
    <meta charset="UTF-8">
    <meta http-equiv="refresh" content="0;url=../bizHome.html" />
    <title></title>
</head>
<body>

</body>
</html>
```




