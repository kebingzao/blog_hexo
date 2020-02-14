---
title: 浏览器 extension 插件开发系列(17) -- Safari 遇到的问题
date: 2019-11-25 13:28:17
tags: 
- js
- 浏览器插件
- Safari 插件
categories: 
- 前端相关
- 浏览器 extension 插件开发系列
---
## 前言
本节主要是讲了一下 Safari extension 在开发的时候，遇到的一些问题，大的问题会在之前的系列会提到。小的问题会在这边提。
## 如何找到现有的 extension 的代码
开发一个东西，一个很快的捷径就是参考别人的代码，尤其是一些界面细节的时候，所以当我们找到了一个值得我们参考的 extension 的时候，怎么样得到这个插件的源代码，这个也是很重要的。
<!--more-->
以这个应用 `pushbullet` 来说，当安装了 `pushbullet` 的 `Safari extension`, 接下来当前要找到插件所在的文件，从而看到代码。假设我已经安装了，接下来只要到这个文件夹就可以看到(mac 机子):
```javascript
~/Library/Safari/Extensions
```

![png](1.png)

就可以看到这个文件了

![png](2.png)

接下来进行解压:
```javascript
xar -xf 'TheExtensionName.safariextz'
```
就可以得到解压之后的文件夹了

![png](3.png)

里面就是对应的代码了

## 将扩展打包成安装包
当我们扩展写完之后，如果要给比人用的话，那么是要变成安装包的方式的，具体步骤是在扩展创建器里面，点击创建软件包，然后选择一个目录来生成安装包。这样就可以了

![png](4.png)

![png](5.png)

## 怎么调试弹出框 popup 页面
在开发 `Safari extension` 的时候，在调试弹出框 `popup` 的时候，会出现一个很蛋疼的问题。就是在弹出页面右键选择审查元素，这时候调试窗口出来了，但是弹出页面就消失了(原因就是因为失去焦点了，所以消失了，跟`Firefox`一样)如果要进行界面上的大改动，肯定不能这样调的。

有一种方法是单独在浏览器窗口中打开，然后作为普通`html`页面来调试, 直接复制完整的 url， 然后在浏览器打开；

![png](6.png)

![png](7.png)

## 获取 cookie 的事情
在用 `Safari extension` 实现第三方登录的时候，发现服务端写入前端的`cookie`，在前端请求的时候，带过去的`cookie`中只显示`ex_account_sid`，而没有了 `ex_account_info`, 后面在用抓包工具抓取的时候，发现`ex_account_sid` 叫做 `parameter cookie`，  而 `ex_account_info` 叫做 `session_cookie`, 而 `parameter cookie`在`extension` 请求的时候，是可以带过去的，而 `session cookie`是带不过去的，后面查了一下服务端的代码， 发现如果`keep`参数为0(就是不保持登录)， 那么 `ex_account_info` 就没有设置过期时间，这时候该`cookie`就会变成一个 `session cookie`，这时候浏览器插件就不会在请求的时候，把这个`cookie`带过去，而`ex_account_sid` 一直都有设置过期时间，才能带过来的。

所以总结一下， 不是不能带 cookie，而是只有 `parameter cookie` 类型的 cookie 才能带，就是要有明确的过期时间的 cookie，才能带过去。 而像`session cookie` 这种会话 cookie 是带不过去的。

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




