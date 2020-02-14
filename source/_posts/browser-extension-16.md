---
title: 浏览器 extension 插件开发系列(16) -- Firefox 遇到的问题
date: 2019-11-25 13:28:16
tags: 
- js
- 浏览器插件
- Firefox 插件
categories: 
- 前端相关
- 浏览器 extension 插件开发系列
---
## 前言
本节主要是讲了一下 Firefox extension 在开发的时候，遇到的一些问题，大的问题会在之前的系列会提到。小的问题会在这边提。
## 如何找到现有的 extension 的代码
开发一个东西，一个很快的捷径就是参考别人的代码，尤其是一些界面细节的时候，所以当我们找到了一个值得我们参考的 extension 的时候，怎么样得到这个插件的源代码，这个也是很重要的。
<!--more-->
以这个应用 `pushbullet` 来说，当安装了 `pushbullet` 的 `Firefox extension`, 接下来当前要找到插件所在的文件，从而看到代码。

注意，`Firefox extension`的代码存在两个地方，一个是本身浏览器自带的`extension`，这个就放在`Firefox`的安装目录里面:

![png](1.png)

而如果是自己去下载的`extension`，那就不是放在这边了。而是放在个人的文件夹中，以 `windows` 来说，就是
```javascript
C:\Users\admin\AppData\Roaming\Mozilla\Firefox\Profiles\0p5aekfy.default\extensions
```

![png](2.png)

这时候，就可以看到你下载的所有`Firefox extension`。但是我们发现有些`extension`是以`xpi`后缀，那怎么看？其实只要把`xpi`重命名为 `zip`，然后解压`zip`包，然后就可以看到了,最后就可以看到`pushbullet`的源码了

![png](3.png)

## popup 页面没办法做文件拖拽的原因
通过 {% post_link browser-extension-15 %}我们知道 Chrome 可以在插件的 popup 上做文件拖拽上传。 但是 `Firefox` 却不行，就是因为在点击上传文件的时候，当文件选择框弹出来的时候，该`panel` 框会关闭:

![png](4.png)

后面查了一下资料，发现如果是用 `add-on` 初始化的`panel`(我们就是这一种)，是不能保持显示的，他一旦焦点失去（第三方登录也是一样），就会关闭掉。只有使用 `xul` 形式的`extension`，才有一个属性是可以保持显示的。 即这一种才行：

![png](5.png)

具体可以看这边: [firefox-addon-builder-how-to-keep-a-panel-shown](http://stackoverflow.com/questions/16012885/firefox-addon-builder-how-to-keep-a-panel-shown)

以下是对应的文档:
- [add-on sdk panel](https://developer.mozilla.org/en-US/Add-ons/SDK/High-Level_APIs/panel)
- [xul panel](https://developer.mozilla.org/en-US/docs/Mozilla/Tech/XUL/panel#a-panel.noautohide)

这边就涉及到一个问题，那就是为啥我们要选用 addon 的方式来做扩展?

## 为啥选用 addon 的方式来做扩展
一个很重要的原因是如果选用 `xul` 的方式，这个 ui 的界面的开发会非常不习惯，因为是另外一种语法，跟 `Chrome extension` 的写法差太多了，在界面上根本没办法重用。所以才要做这种自引导性的扩展，而且它还有以下好处:
1. 可以手动添加用户界面， 这样就不用担心做一些适应`Firefox extension`的`xul`页面了，比如： 您需要在相关应用程序中调用 `document.getElementById()`，通过 `UI` 元素的 `ID` 来查找它们，然后操纵它们来注入您的 `UI`。例如，您可以通过 `document.getElementById("main-menubar")` 来访问 `Firefox` 的菜单栏。
2. 可以像`Chrome extension` 开发那样，引入一些js进去。
要做到自引导型扩展，添加下列元素到它的安装清单，即 `install.rdf`
```javascript
<em:bootstrap>true</em:bootstrap>
```
3. 不需要 `Chrome.manifest` 文件
不过没有这个文件，但也多了其他的文件。

## 怎么在浏览器打开这个插件的目录索引
很简单，找到这个插件名字，然后在地址栏输入 `resource://extension-id` 这样就可以了。

![png](6.png)

甚至可以通过这种方式来调整页面细节

![png](7.png)

## 对 cookie 的操作
Firefox 对 cookie 的操作，不像 Chrome 那样，还有一个专门的对象来读取:
```javascript
chrome.cookies
```
而是要在中转页 main.js 那边去设置的，所以抽象出来是这样子的:
```javascript
var COOKIE_URL = "http://*.airdroid.com";
Cookies = {
    get: function(name, cb){
        var uniqId = _.uniqueId("cookie_");
        Airdroid.Event.dispatchEvent(Airdroid.Event.FireFoxEvent.get_cookie,{
            uid: uniqId,
            name: name,
            domain: COOKIE_URL
        });
        Airdroid.Event.addEventListener(uniqId, function(data){
            Airdroid.log("cookie 为" + data.detail);
            _.isFunction(cb) && cb((data.detail ? {
                name: data.detail,
                domain: COOKIE_URL
            } : null));
        },true);
    },
    set: function(opt, cb){
        var uniqId = _.uniqueId("cookie_");
        Airdroid.Event.dispatchEvent(Airdroid.Event.FireFoxEvent.set_cookie,{
            uid: uniqId,
            opt: opt,
            domain: COOKIE_URL
        });
        Airdroid.Event.addEventListener(uniqId, function(data){
            _.isFunction(cb) && cb(data);
        },true);
    },
    remove: function(name, cb){
        var uniqId = _.uniqueId("cookie_");
        Airdroid.Event.dispatchEvent(Airdroid.Event.FireFoxEvent.remove_cookie,{
            uid: uniqId,
            name: name,
            // 这边要 .airdroid.com 不能加星号
            domain: COOKIE_URL.split("//")[1].substr(1)
        });
        Airdroid.Event.addEventListener(uniqId, function(data){
            _.isFunction(cb) && cb(data);
        },true);
    }
}
```
而具体的操作其实在 `main.js`, 这边也是用到了之前将的事件驱动模型: {% post_link browser-extension-10 %}
```javascript
var {Cc, Ci} = require('chrome');
var ioSvc = Cc['@mozilla.org/network/io-service;1'].getService(Ci.nsIIOService);
var cookieSvc = Cc['@mozilla.org/cookieService;1'].getService(Ci.nsICookieService);
var cookieMgr = Cc['@mozilla.org/cookiemanager;1'].getService(Ci.nsICookieManager);

// cookie operate
var getCookie = function(name, domain){
    var cookieString = cookieSvc.getCookieString(ioSvc.newURI(domain, null, null), null);
    if (cookieString != null) {
        var cookies = cookieString.split(';');
        for (var i = 0, cookiesLength = cookies.length; i < cookiesLength; i++) {
            var cookie = cookies[i];
            if (cookie.length > 0) {
                var parts = cookie.split('=');
                if (trim(parts[0]) == name) {
                    return trim(parts[1]);
                }
            }
        }
    }
    return null;
};

page.port.on("get_cookie", function(data){
    data = data.detail;
    var str = getCookie(data.name, data.domain);
    page.port.emit(data.uid, {detail: str});
});

page.port.on("set_cookie", function(data){
    data = data.detail;
    var opt = data.opt;
    cookieSvc.setCookieString(ioSvc.newURI(data.domain, null, null), null, opt.name + '=' + opt.value, null);
    page.port.emit(data.uid, {detail: cookieSvc.getCookieString(ioSvc.newURI(data.domain, null, null), null)});
});

page.port.on("remove_cookie", function(data){
    data = data.detail;
    cookieMgr.remove(data.domain, data.name, '/', false);
    page.port.emit(data.uid, {});
});
```
这样才能得到 cookie

## 进行跨域请求的时候
`Firefox` 在进行跨域请求的时候，也很奇葩，如果服务端没有添加跨域头部的话，会失败，哪怕用 `jsonp get` 请求也不行。 所以要用它自带的网络请求，也是在 `main.js` 里面处理, 当然我们先在背景页先封装方法:
```javascript
// 最基本的get方法
baseGetRequest: function (url, data, type) {
    if(Airdroid.Util.Browser.firefox){
        // Firefox 比较特殊，有限制，如果服务端没有添加跨域头部的话，就会请求失败, 所有要用自带的请求
        var defer = $.Deferred();
        url = url + "?" + $.param(data);
        var uid = _.uniqueId("request_");
        type = type || 'json';
        Airdroid.Event.addEventListener(uid, function(data){
            var status = data.status;
            if (status == 200) {
                try {
                    // 请求成功
                    if(type == 'json'){
                        defer.resolve(data.json || {});
                    }else{
                        defer.resolve(data.text || "");
                    }
                } catch (e) {
                    console.log(e);
                    defer.reject();
                }
            } else if (status === 401) {
                defer.reject();
            } else {
                defer.reject();
            }
        },true);
        Airdroid.Event.dispatchEvent(Airdroid.Event.FireFoxEvent.request_get,{
            url: url,
            uid: uid
        });
        return defer;
    } else {
        return $.ajax({
            type: "GET",
            timeout: 10000,
            url: url,
            data: data,
            dataType: type || 'json'
        });
    }
},
```
具体的实现是在 `main.js` 里面:
```javascript
page.port.on("request_get", function(data){
    data = data.detail;
    Request({
        'url': data.url,
        'onComplete': function(response) {
            page.port.emit(data.uid, {
                json: response.json,
                text: response.text,
                status: response.status,
                statusText: response.statusText
            });
        }
    }).get();
});
```
通过最后回调 uid 来回到背景页的回调。

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

