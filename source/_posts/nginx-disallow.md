---
title: nginx设置站点不能被搜索引擎搜索到
date: 2019-05-31 11:53:30
tags: nginx
categories: nginx相关
---
## 前言
之前运营有反馈说，可以在google 上搜索到我们的测试环境的站点，比如测试官网。 后面我查了一下，发现没有对测试的站点，做爬虫屏蔽。
## 解决
因为爬虫在爬站点的时候，会去请求站点根目录下的 robots.txt, 这个文件会告诉搜索引擎一些允许和禁止规则，那我们测试环境肯定要全部屏蔽的， 所以我们直接在 nginx 设置就行了：
```html
location = /robots.txt {
        return 200 "User-agent: *\nDisallow: /";
    }
```
直接告诉爬虫不允许任何客户端来爬数据。

