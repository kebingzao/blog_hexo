---
title: 浏览器 extension 插件开发系列(07) -- 获取各浏览器端的背景页
date: 2020-02-11 16:11:46
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
浏览器 `extension` 都有两种形式的页面。一种是常驻的背景页。一种是显示的前端页面。而一般常驻的背景页都会保留很多的变量。这些变量经常会在前端的页面里面使用到。接下来就讲解一下，各个浏览器的背景页形式和怎么在前端的页面获取对应的背景页。
<!--more-->
## 1. Chrome
`Chrome`的背景页的js文件，都是在 `manifest.json` 的`background`里面的`scripts`数组里面:
```javascript
"background": {
    "scripts": [
        "js/lib/underscore/underscore.js",
        "js/lib/jquery/jquery.js",
        "js/lib/md5/md5.js",
        // des lib
        "js/lib/des/tripledes.js",
        "js/lib/des/mode.ecb.js",
        // e2ee lib
        "js/lib/e2ee/System.js",
        "js/lib/e2ee/System.IO.js",
        "js/lib/e2ee/System.Text.js",
        "js/lib/e2ee/System.Convert.js",
        "js/lib/e2ee/System.BitConverter.js",
        "js/lib/e2ee/System.BigInt.js",
        "js/lib/e2ee/System.Security.Cryptography.SHA1.js",
        "js/lib/e2ee/System.Security.Cryptography.js",
        "js/lib/e2ee/System.Security.Cryptography.RSA.js",
        "js/lib/e2ee/System.Security.Cryptography.HMACSHA1.js",
        "js/lib/e2ee/System.Security.Cryptography.RijndaelManaged.js",
        // util lib
        "js/util/util.js",
        "js/util/tabs.js",
        "js/util/e2ee.js",
        "js/util/des.js",
        "js/util/localstorage.js",
        "js/util/browser_os.js",
        // model lib
        "js/model/account.js",
        "js/model/file.js",
        "js/model/device.js",
        "js/model/contextmenus.js",
        "js/model/notification.js",
        "js/sys/cache.js",
        "js/sys/events.js",
        "js/sys/baseSocket.js",
        "js/sys/subSocket.js",
        "js/sys/pushManage.js",
        "js/sys/notificationManage.js",
        "js/sys/server.js",
        "js/sys/background.js"
    ]
},
```
也就是说，在启动这个插件的时候，`Chrome` 就会加载这些js文件。这时候如果生成的变量就是背景页的变量。比如 `window.Airdroid` 这个全局变量。这个变量只在背景页的这个`window`环境里面存在。而前端页面，比如 `popup.html` 的`window`环境里面是取不到这个变量的，因为这个是两个不一样的环境。

那么怎么在前端页面的环境取到这个变量呢？ chrome有提供一个函数:
```javascript
chrome.extension.getBackgroundPage()
```
通过这个函数，可以取到背景页的`window`对象。

![png](1.png)

## 2. Firefox
跟`Chrome`不一样。`Firefox`的背景页是在`main.js`中设置的, `lib/main.js` 部分代码如下:
```javascript
// 初始化插件的常驻背景页
var page = require('sdk/page-worker').Page({
  'contentURL': self.data.url('page.html?version=1'),
  'contentScriptFile': [
    self.data.url('js/lib/underscore/underscore.js'),
    self.data.url('js/lib/jquery/jquery.js'),
    self.data.url('js/lib/md5/md5.js'),
    // des lib
    self.data.url('js/lib/des/tripledes.js'),
    self.data.url('js/lib/des/mode.ecb.js'),
    // e2ee lib
    self.data.url('js/lib/e2ee/System.js'),
    self.data.url('js/lib/e2ee/System.IO.js'),
    self.data.url('js/lib/e2ee/System.Text.js'),
    self.data.url('js/lib/e2ee/System.Convert.js'),
    self.data.url('js/lib/e2ee/System.BitConverter.js'),
    self.data.url('js/lib/e2ee/System.BigInt.js'),
    self.data.url('js/lib/e2ee/System.Security.Cryptography.SHA1.js'),
    self.data.url('js/lib/e2ee/System.Security.Cryptography.js'),
    self.data.url('js/lib/e2ee/System.Security.Cryptography.RSA.js'),
    self.data.url('js/lib/e2ee/System.Security.Cryptography.HMACSHA1.js'),
    self.data.url('js/lib/e2ee/System.Security.Cryptography.RijndaelManaged.js'),
    // util lib
    self.data.url('js/util/util.js'),
    self.data.url('js/util/tabs.js'),
    self.data.url('js/util/e2ee.js'),
    self.data.url('js/util/des.js'),
    self.data.url('js/util/localstorage.js'),
    self.data.url('js/util/browser_os.js'),
    // model lib
    self.data.url('js/model/account.js'),
    self.data.url('js/model/file.js'),
    self.data.url('js/model/device.js'),
    self.data.url('js/model/contextmenus.js'),
    self.data.url('js/model/notification.js'),
    self.data.url('js/sys/cache.js'),
    self.data.url('js/sys/events.js'),
    self.data.url('js/sys/baseSocket.js'),
    self.data.url('js/sys/subSocket.js'),
    self.data.url('js/sys/pushManage.js'),
    self.data.url('js/sys/notificationManage.js'),
    self.data.url('js/sys/server.js'),
    self.data.url('js/sys/background.js')
  ]
});
```
这边需要注意的一点是。这边有引入一个页面 `page.html`。 其实这个页面就是一个空白页, 只是用来充当背景页的html容器。 `page.html`:
```javascript
<!DOCTYPE html>
<html>
    <head>
        <meta charset="utf-8">
    </head>
    <body>
    </body>
</html>
```
那么如何在前端的页面获取背景页的对象呢? 其实在`Firefox`中是没有像`Chrome`那样，直接有一个
```javascript
chrome.extension.getBackgroundPage() 
```
方法来获取`background`对象的。在`Firefox`中，前端页面和背景页的通信， 全部是通过触发事件来进行传输的。也就是说，无论是前端给背景页通信，还是背景页给前端通信。其实都是通过观察者模式来触发的。也就是说，假设前端要取得背景页的一个对象，那么要先`emit`一个事件给后端，让他知道前端要数据。然后在`on`监听一个事件。让背景页`emit`这个事件，把想要的这个数据传过来。

后续有完整的一个消息通信流程来讲述这个， 具体看 {% post_link browser-extension-10 %}

## 3. Safari
`Safari` 的背景页其实就是`global页面`。

![png](2.png)

那么前端页面是如果取得`global页面`的数据，并通信的呢。

而且Safari的前端页面分为两个类型:
1. Info.plist 所指定的popup页面。
针对这个页面。Safari有一个方法可以直接访问背景页的window变量
```javascript
safari.extension.globalPage.contentWindow
```

![png](3.png)

2. 其他的html页面
如果是这种页面的话。是没有`safari.extension.globalPage` 这个对象的。 所以要用消息传递的方式来传输。即监听的方式, 部分代码如下:
```javascript
// 如果是 popup这种 popover页面，那么可以直接访问后台的全局变量
if (safari.extension.globalPage) {
  var gp = safari.extension.globalPage.contentWindow;
  self.setUpBackgroundPage(gp);
  self.inited(args);
} else {
  // 其他的，像 reply 这种新建的窗体，就只能用数据传输的方式
  safari.self.addEventListener('message', function(e) {
    var eventName = e.name;
    console.log("消息过来了，名字为：" + eventName);
    if (eventName == 'page_data') {
      var proxy = safari.self.tab;
      e.message = _.isString(e.message) ? JSON.parse(e.message) : e.message;
      self.setUpBackgroundPage(e.message, proxy.dispatchMessage);
      self.inited(args);
    }else{
      if(self.funCbObj[eventName]){
      self.funCbObj[eventName](e.message);
      delete self.funCbObj[eventName];
    }
  }
}, false);
safari.self.tab.dispatchMessage('page_init');
```
对于这种方式，具体看这个： {% post_link browser-extension-10 %}







