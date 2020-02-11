---
title: 浏览器 extension 插件开发系列(01) -- 前言和确认需求
date: 2020-02-11 10:52:56
tags: 
- js
- 浏览器插件
- Chrome 插件
- Firefox 插件
- Safari 插件
categories: 
- 前端相关
- 浏览器 extension 插件开发系列
---
## 前言
最近因为 Google 公布了，到 2020 年 6 月不在支持非 Chrome OS 平台的 chrome app 的支援。具体看: [Google 確定今年 3 月不再受理上架 Chrome Apps，6 月結束支援](https://www.eprice.com.tw/tech/talk/1184/5441568/1/), 刚好我们有一个 chrome app 有应用到业务中。那么就要赶在 6月份之前，将这个 chrome app 改造成 chrome extension 的方式。这样子就可以继续用了。 考虑到之前我也有做过一个 浏览器 extension 插件，而且是同时兼容 chrome， Firefox， Safari 这三个浏览器的， 但是那时候是 2016 年做的插件，已经隔了三年多了，几乎全忘光了，赶紧翻了一下之前做的笔记，还好那时候有做开发笔记。所以就将这一份开发笔记挪到了 blog 上。
<!--more-->
<font color=red><b>ps: 这边需要注意的是这一个系列的笔记是我在 2016 年开发浏览器插件的时候，做的笔记，到现在已经快 4 年了，我不太确定相关的语法和特性是否有改了很多。所以如果想借鉴的同学还是要去读文档更有效，我这边只能给你们提供一个思路而已。</b></font>
<br><font color=red><b>ps: 接下来这个系列的所有的语境都是在 2016 年那时候做笔记的时候，直接照抄的。值得一提的虽然那时候只是做个 demo(后面因为其他原因，这个插件并没有上架到 chrome store 商店)，但是实际上实现的功能并不少。</b></font>

<br><font color=red><b>ps: 这个是第一版的情况，我记得后面有用 react 重构改了一版，而且去掉了 Firefox 和 Safari 插件的兼容，只保留了 Chrome extension 插件。同时界面上也进行了改版。但是那一版做的比较急吧，我并没有找到那一版的笔记。 所以只能用最初的这一版了。(如果后面有时间的话，再根据现有的代码还原一下当时的情况)</b></font>

## 想法和确定需求
我们产品那时候为了拉新，打算扩大平台可用性，因此盯上了浏览器插件这个新的平台，打算做一个浏览器插件，最好兼容 chrome， Firefox， Safari。这个插件就是extension 形式的。所以我就按照现有的一些功能，做了第一版的 demo。 需要完成一下功能:

1. 登录(账号登录，第三方登录)
2. 文件上传(多文件上传，拖拽上传)
3. 发送文本(e2ee加密)
4. 消息接收(手机通知，文件，文本)
5. 右键菜单直接推送消息

接下来的教程会一一介绍。

## demo 效果图
第一版的效果界面如下:

![png](1.png)

![png](2.png)

## 目录结构
因为这个插件要兼容三种浏览器的插件格式。即Firefox，chrome，Safari， 因此目录结构是有要求的。 因此整个目录看起来会比较乱，接下来将根据不同的插件扩展来分一下所属的目录结构。

![png](3.png)

### 1. chrome extension
chrome 插件的目录结构比较集中
1. resources/airdroid/data/ -- 需要打包的文件都在这个目录
2. resources/airdroid/data/manifest.json -- 是入口配置文件 (Firefox不需要)

### 2.Firefox extension
1. default 目录 -- 一些配置项
2. resources -- 主要的资源
  1. data -- 跟其他浏览器插件公用的资源
  2. lib -- 启动之后的入口文件
3. bootstrap.js -- 插件的启动文件
4. harness-options.json -- 插件的配置文件
5. icon.png -- 插件的显示图标
6. icon64.png -- 插件的显示的大图标
7. install.rdf -- 插件的安装配置文件
8. kbz@airdroid.net -- 这个文件放在这里是没有用的，主要是放在其他目录里面用于插件调试的
9. locales.json -- 插件的本地化文件
10. options.xul -- 插件的配置页面

### 3.safari extension 文件
1. resources/airdroid/data/ -- 需要打包的文件都在这个目录, 同时这个目录，在安装的时候，要把data目录名字改成 airdroid.safariextension
2. Info.plist -- 配置文件
3. global.html -- 全局文件
4. icon-64.png -- 图标
5. safari-end-script.js -- 注入脚本，主要是用来设置当前tab页面右键菜单内容的

### 4.gulp 打包文件
1. .variables -- 用于取消系统之间的打包差异
2. gulpfile.js -- gulp的打包配置文件
3. package.json -- gulp的打包配置文件


















