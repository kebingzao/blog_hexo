---
title: 浏览器 extension 插件开发系列(02) -- Chrome 插件的启动以及调试
date: 2019-11-25 13:28:02
tags: 
- js
- 浏览器插件
- Chrome 插件
categories: 
- 前端相关
- 浏览器 extension 插件开发系列
---
## 前言
接下来讲各个浏览器的启动以及调试。本节讲的是`Chrome`。 `chrome` 的整个目录是 `resources/airdroid/data/`, 而`chrome` 的配置文件是 `manifest.json`.  路径是 `resources/airdroid/data/manifest.json` (`Firefox`和`Safari`不需要这个文件)。
<!--more-->
## manifest.json
```javascript
{
    "manifest_version": 2,
    "name": "Airdroid",
    "description": "Access Android phone/tablet from computer remotely and securely. Manage SMS, files, photos and videos, WhatsApp, Line, WeChat and more on computer.",
    "version": "1.0",
    "permissions": [
        "tabs",
        "cookies",
        "contextMenus",
        "notifications",
        "clipboardWrite",
        "clipboardRead",
        "http://*.airdroid.com/",
        "https://*.airdroid.com/",
        "http://s3.amazonaws.com/",
        "https://airdroid-test.s3.amazonaws.com/"
    ],
    "content_security_policy": "script-src 'self' 'unsafe-eval'; object-src 'self'",
    "content_scripts": [{
        "matches": [ "http://*/*", "https://*/*" ],
        "js": ["js/sys/inject.js"]
    }],
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

    "icons": {
        "16": "images/icon.png",
        "48": "images/icon.png",
        "128": "images/icon.png"
    },

    "browser_action": {
        "default_icon": "images/icon.png",
        "default_popup": "popup.html"
    }
}
```
接下来解释一下各字段:

|字段|描述|
|---|---|
|manifest_version|内部版本号（每次提交store都要升这个号）|
|name|插件名|
|description|插件描述|
|version|插件版本号|
|permissions|插件所需要的权限|
|content_security_policy|注入脚本所需要的安全权限|
|content_scripts|注入脚本的脚本文件名以及所匹配的域名|
|background|背景页所需要加载的js|
|icons|插件的icon|
|browser_action|插件在导航条的icon以及点击这个icon所对应要弹出的页面|

## 放到 chrome 浏览器
接下来就是把文件的开发目录放到 `chrome` 里面:

`chrome://extensions/` --> `加载已解压的扩展程序（要先勾选开发者模式）` ->  `在弹出的文件选择框选择目录`

最后的效果就是这样子:

![png](1.png)

这样就已经启用了。

## 调试
接下来怎么调试呢？？

`chrome extension` 分为两种页面:

### 1. 背景页
一种是一直长存的背景页，即这个插件一启用，如果浏览器没有关闭的话，那么该页面就一直在后台。点击背景页，可以查看背景页的代码:

![png](2.png)

![png](3.png)

可以看到背景页这个页面是 `chrome` 自己生成的。里面就是引入一堆的js文件。

而这些背景页的js文件。就是上述`manifest` 的 `background -> scripts` 里面的东西。接下来就可以正常调试了。

**注意： 如果开发过程中，`background` 的js文件有修改过的话，要点击“重新加载”， 好让代码更新过来。**

![png](4.png)

### 2.普通页面
另一种是当前显示的普通页面，插件还可以有很多的页面。这些页面如果开启的话，那么就可以调试。如果关闭的话，那么就关了，就不能调试了。

![png](5.png)

这个 `popup` 就是一个这种页面。 要调试也很简单，就是`右键点击-》 检查`

![png](6.png)

如果这个页面被关闭的话，这个调试页面也会关闭。

## 总结
关于 `chrome extension` 插件的启动和调试大概就这样了。

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























