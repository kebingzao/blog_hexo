---
title: 浏览器 extension 插件开发系列(03) -- Firefox 插件的启动以及调试
date: 2019-11-25 13:28:03
tags: 
- js
- 浏览器插件
- Firefox 插件
categories: 
- 前端相关
- 浏览器 extension 插件开发系列
---
## 前言
通过 {% post_link browser-extension-2 %} 可以知道`chrome extension`的一些启动和调试, 接下来讲讲`Firefox`的。

`Firefox` 需要的文件比较多:
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

<!--more-->
## 怎么安装
首先把这些文件所在的目录打成一个`zip`包，然后重命名为后缀为 `xpi` 的 `airdroid_extension.xpi`. 然后打开`Firefox`的一个空白页，直接拖上去就可以了。

![png](1.png)

然后点击安装

![png](2.png)

就可以在导航条看到:

![png](3.png)

同时可以在插件栏里面看到:

![png](4.png)

![png](5.png)

## 开发时的安装
上面的安装已经成功了，但是其实在开发中是比较麻烦的。需要经常重复这个过程:

1. 先把这个目录打包成zip
2. 然后重命名为 xpi后缀
3. 最后再拖到Firefox 安装
4. 安装完之后，还要重启，最后才能安装成功看效果。

那如果开发的时候，每次都这样，那蛋都碎了。因此我后面找了一个更好的方式：不用每次都打包成xpi，而是用路由的方式来指定，修改 `install.rdf` 的 `id`
具体方法为：
1. 先把之前那个拖入的xpi的移除掉。不然会有命名冲突
2. 找到Firefox所在用户属性目录的， 比如
```text
C:\Users\admin\AppData\Roaming\Mozilla\Firefox\Profiles\0p5aekfy.default\extensions
```
3. 新建一个文本文件，名字命名为`install.rdf` 里面的 `em:id`, 比如 airdroid@example.net , 然后打开文件， 输入这个项目所在的路径 `F:\airdroid_code\extension\sample\` 最后点击保存。

![png](7.png)

这个文件里面的内容为：

![png](8.png)

即开发目录所对应的目录：

![png](9.png)

4. 重启`Firefox`， 这时候`Firefox`会去检查这个目录里面的扩展， 如果有新的，会出现该提示。点击允许安装，这样，相当于插件就安装了，以后修改的话，就不用每次都安装了，而是直接在对应的项目目录里面修改。

第一次会出现让你安装的提示：

![png](10.png)

勾选按钮。点击继续，这时候就看到已经安装了。打开调试框，看下log:

![png](11.png)

接下来在对应的文件，改下log:

![png](12.png)

然后重启一下， 这时候就不用再安装了，直接`debugger` 就行了, 可以看到log已经变成最新的了:

![png](13.png)

用这种方式会很方便。

## 调试插件的代码
那么怎么在插件开发的时候，像 chrome 那样可以直接查看插件的代码呢。 `Firefox`的调试，不像`chrome`那样，直接在`界面上右键-检查`就可以调试了。你在界面上点击右键，什么都出不来。

首先从 `harness-options.json` 获取 `jetpackID`

![png](14.png)

然后在`tab`上打开所在的目录 `resource://kbz-at-airdroid-dot-net/`

![png](15.png)

然后找到对应的页面。比如 `popup` 页面, 但是发现通过这种方式调试，会有一些问题。 而且整个过程会没有连贯性，这种方式只能用来调试界面会比较多。 

![png](16.png)

因此要用另外一种调试方式。即 `addons -> setting -> 调试附加组件`

![png](17.png)

选择所要调的组件，这时候就会跳出这个组件的调试窗口

![png](18.png)

就可以看到完整的调试信息：

![png](19.png)

这样子就可以解决调试的问题了。

## 总结
这样子就完成了 `Firefox` 插件的安装和调试问题了。但是很明显，都没有 `Chrome` 来的方便。

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




