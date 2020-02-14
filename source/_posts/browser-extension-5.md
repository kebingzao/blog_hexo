---
title: 浏览器 extension 插件开发系列(05) -- Safari 插件申请开发者证书
date: 2019-11-25 13:28:05
tags: 
- js
- 浏览器插件
- Safari 插件
categories: 
- 前端相关
- 浏览器 extension 插件开发系列
---
## 前言
本节聊一下 `Safari extension` 怎么申请开发者证书
## 1. 首先将Safari的开发者模式打开

![png](1.png)
<!--more-->
## 2. 接下来显示扩展创建器

![png](2.png)

## 3. 发现创建完之后，会提示没有开发者证书

![png](3.png)

## 4. 因此要去官网申请一个开发者证书
官网地址： https://developer.apple.com/ ， 登录之后，选择开发类型为 Safari 扩展

![png](4.png)

然后加入

![png](5.png)

资料填写完，选择开始：  id 为 `J294WP5P2Y`

![png](6.png)

![png](7.png)

进入证书

![png](8.png)

选择Safari 扩展证书下载

![png](9.png)

创建一个证书

![png](10.png)

最后会让你选择一个本地证书来签名

![png](11.png)

## 5. 导出本地证书
而我们还没有导出本地证书，接下来就要导出这台mac机子的本地证书
1. 在“钥匙串访问” 菜单中选择 “证书助理” -> “从证书颂发机构求证书” 
2. 填写 “电子邮件” “常用名称” “请求是” 选择 “存储到磁盘”
3. 然后点“继续”按钮，会弹出存储为对话框，选择保存“请求文件”的位置，比如桌面，然后点存储。

![png](12.png)

![png](13.png)

![png](14.png)

![png](15.png)

这时候，本地证书已经导出来了。
## 6. 导入本地证书生成开发者证书
然后重新上传该证书

![png](16.png)

![png](17.png)

## 7. 下载开发者证书并导入
最后下载开发者证书，并导入进去

![png](18.png)

![png](19.png)

![png](20.png)

然后在“钥匙串访问”里，切换到“登录”页面，点击左下角的“+”按钮，导入刚刚下载的证书。OK。

再次打开`safari`的“开发” 菜单的 “显示扩展创建器”，就会发现，“安装” 和 “创建软件包”按钮可用了~并且已经拥有了开发者证书，还有你填写的邮箱:

![png](21.png)

![png](22.png)

这样就大功告成了

---
系列文章:
{% post_link browser-extension-1 %}
{% post_link browser-extension-2 %}
{% post_link browser-extension-3 %}
{% post_link browser-extension-4 %}
{% post_link browser-extension-5 %}
{% post_link browser-extension-6 %}
{% post_link browser-extension-7 %}
{% post_link browser-extension-8 %}
{% post_link browser-extension-9 %}
{% post_link browser-extension-10 %}
{% post_link browser-extension-11 %}
{% post_link browser-extension-12 %}
{% post_link browser-extension-13 %}
{% post_link browser-extension-14 %}
{% post_link browser-extension-15 %}
{% post_link browser-extension-16 %}
{% post_link browser-extension-17 %}

