---
title: 浏览器 extension 插件开发系列(08) -- 背景页启动和登录持久化
date: 2019-11-25 13:28:08
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
接下来开始讲这个项目的开发过程。刚开始肯定是背景页启动。不管是哪一种浏览器，背景页都是加载一堆的js。
<!--more-->
```javascript
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
```
前面都是一些js的第三方库，只有最后一些`model`模块和`sys`模块才是业务逻辑。而入口文件就是`background.js`, 也就是当这些文件加载完之后，就生成了一堆的`AirDroid` 命名空间内的变量， 这些变量都是非常重要的。

![png](1.png)

## background.js 内容
接下来看看入口文件的内容, `background.js`:
```javascript
// 开始启动
(function () {
  Airdroid.Util.init();
  // 初始化私钥
  Airdroid.Util.E2ee.init();
  Airdroid.Event.init();
  // 监听登录事件
  Airdroid.Event.addEventListener(Airdroid.Event.TYPE.signed_in, function () {
    console.info("响应登陆事件");
    Airdroid.Cache.setCacheType();
    // 登录之后，初始化右键菜单
    Airdroid.Account.getDevicesObjList().done(function(data){
      Airdroid.ContextMenus.init(data);
    })
  });
  // 监听登出事件
  Airdroid.Event.addEventListener(Airdroid.Event.TYPE.signed_out, function () {
    console.info("响应注销事件");
    // 移除右键菜单
    Airdroid.ContextMenus.removeContextMenu();
  });
  // 检查自动登录
  Airdroid.Account.checkAutoSignIn();
})();
```
逻辑很简单：
1. 初始化工具类 util
2. 初始化私钥
3. 初始化事件驱动模型
4. 监听登录事件和登出事件
5. 检查自动登录

其中自动登录，就是把本地存储的`cookie值`带过去服务端进行登录验证。 如果登录成功触发登录事件，并保存登录后的`cookie值`。登录失败就触发登出事件，并删除`cookie值`。

这边要注意一点的是， 因为`Safari`不支持`global页面的cookie传递`，所以这边将`cookie值`传过来，然后保存在本地。同时自动登录的时候也要把`cookie值`当做参数传过去。

ps： `Safari` 这边原则上是不支持 `session cookie` 的传递，对于 `parameter cookie` 类型的 `cookie` 是可以传递的。 具体看 {% post_link browser-extension-17 %}

## 实现登录持久化
其实以上就实现了登录持久化了，就是在背景页启动的时候，去检查自动登录的情况。

1. 先判断本地的`localstorage` 是否有对象的 `cookie值`。(之所以一定要存本地存储，是因为在`Safari`下，`global`不支持`cookie`传递，也就是说在`global`页面进行请求的时候，`cookie`不会自动带到服务端，所以服务端也读不到，所以我们要自己存储 `cookie`, 并当做参数带过去)
2. 如果有的话。就调用登录接口，把mail和保存在内存中的cookie传过来，进行登录。
3. 如果登录成功，就说明自动登录成功。

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








