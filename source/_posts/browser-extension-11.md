---
title: 浏览器 extension 插件开发系列(11) -- 登录模块(包括第三方登录和弹框)
date: 2020-02-12 16:00:29
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
本节我们讲一下登录模块的实现。一共有两种登录方式(当然无论是哪一种登录，都离不开服务端的配合)。
1. 账号登录
2. 第三方登录  

![png](1.png)
<!--more-->
## 实现
账号登录没什么好说的，就调用背景页的`Airdroid.Account.signIn` :
```javascript
this.account.signIn(mail, pwd)
  .done(function () {
    self.afterSignIn();
  }).fail(function (resp) {
    //alert(resp.msg);
    var msg = resp && resp.msg || "login error!";
    $.toast(msg, 'warning');
  }).always(function(){
    self.signInEnableDom();
    setTimeout(function(){
      self._isSigning = false;
    },100);
  });
```
## 第三方登录实现
主要是第三方登录。主要提供Facebook， Twitter， Google+ 三种方式。 操作方式就是点击之后，弹出一个授权框。 `popup.js` 的代码如下:
```javascript
// 第三方登录
thirdPartSigninHandle: function(e){
    var self = this;
    var service = $(e.currentTarget).attr("data-type");
    self.util.toUrlParam({
        service: service,
        lang: 'en',
        extension: "1"
    }).done(function(dataStr){
        self.util.Tab.popupWindow({
            url: self.server.urls.id + 'user/tplogin.html?' + dataStr,
            width: 700,
            height: 600,
            location: 'yes'
        }).done(function () {
            // 登录, 这时候服务端会写上 ex_account_sid ,这时候只要重新调下自动登录就行了
            $.toast('Logining', 'info');
            self.signInDisableDom();
            self.account.checkAutoSignIn(true).fail(function(){
                self.signInEnableDom();
            });
        })
    });
},
```
逻辑其实没啥好说的，就是配合服务端的接口，来实现第三方登录的授权。这边主要讲一个弹出`popupWindow`的各浏览器的实现。 封装在 `tabs.js` 里面，里面有各个浏览器的弹出窗体实现，并且当窗体关闭的时候，会执行`done`回调, `util/tabs.js`:
```javascript
/**
 *
 * 显示弹出窗口
 * @param opts 配置项 opts 包括
 *      url   : 弹窗内容的 url地址
 *      width : 窗口文档显示区的高度 以像素计    默认800
 *      height: 窗口的文档显示区的宽度 以像素计  默认600
 */
popupWindow: function (opts) {
    var defer = $.Deferred();
    var isFirefox = Airdroid.Util.Browser.firefox;
    var popup, interval, url, onClose, optsArr;
    opts.id = _.uniqueId("new_window_id");
    opts = $.extend({
        url: './',
        width: 800,
        height: 600
    }, opts);

    if(Airdroid.Util.Browser.chrome){
        // 如果没有指定位置，就计算位置，让弹出的窗口居中。
        opts.top = opts.top || (window.screen.availHeight - 30 - opts.height) / 2;
        opts.left = opts.left || (window.screen.availWidth - 10 - opts.width) / 2;

        url = opts.url;
        // 这两个参数不是窗口配置项，所以删掉他们。
        delete opts.url;

        optsArr = [];
        $.each(opts, function (key, val) {
            optsArr.push(key + "=" + val);
        });
        popup = window.open(url, '_blank', optsArr.join(','));
        popup.focus();
        interval = setInterval(function () {
            if (popup.closed) {
                defer.resolve();
                clearInterval(interval);
            }
        }, 500);
    }else if(Airdroid.Util.Browser.safari){
        var w = safari.application.openBrowserWindow();
        w.activeTab.url = opts.url;
        w.addEventListener("close", function(){
            console.log("窗口关闭");
            // todo 这边有个很严重的问题，因为safari下不能读取cookie的因素，导致第三方自动登陆，哪怕登陆成功，前端这边也不知道
            defer.resolve();
        },false);
    }else{
        // firefox
        Airdroid.Event.dispatchEvent(Airdroid.Event.FireFoxEvent.new_window_open,opts);
        // Firefox要做特殊处理
        Airdroid.Event.addEventListener(Airdroid.Event.FireFoxEvent.new_window_close, function(tid){
            if(tid == opts.id){
                //如果关闭就执行defer操作
                defer.resolve();
                // todo 因此Firefox的特殊性，导致第三方窗口一打开，popup窗口就会关闭，所以这边要手动重新检查是否登录
                Airdroid.Account.checkAutoSignIn();
            }
        },true);
    }
    return defer;
},
```
### 1. Chrome
逻辑跟正常的`html`一样，差不多:
```javascript
popup = window.open(url, '_blank', optsArr.join(','));
popup.focus();
interval = setInterval(function () {
    if (popup.closed) {
        defer.resolve();
        clearInterval(interval);
    }
}, 500);
```
### 2. Safari
`Safari`要调用一个方法 `openBrowserWindow`:
```javascript
var w = safari.application.openBrowserWindow();
w.activeTab.url = opts.url;
w.addEventListener("close", function(){
    console.log("窗口关闭");
    // todo 这边有个很严重的问题，因为safari下不能读取cookie的因素，导致第三方自动登陆，哪怕登陆成功，前端这边也不知道
    defer.resolve();
},false);
```
并且可以监听这个窗口的`close`事件。

ps: 但是这个有个问题，因为`Safari`的`popup`页面在请求接口的时候，不能把`cookie`带过去，所以当第三方登录成功之后，前端也不知道，所以这个问题，后面要想办法解决??? 所以就会这么一个问题，就是明明第三方登录授权成功，服务端也写入 `cookie`, 但是前端就是没办法取到，导致哪怕第三方登录成功，但请求还是失败

ps： `Safari` 这边原则上是不支持 `session cookie` 的传递，对于 `parameter cookie` 类型的 `cookie` 是可以传递的。 具体看 {% post_link browser-extension-17 %}, 本例之所以不支持是因为第三方登录服务端写入的 cookie 是属于 `session cookie`， 所以才会取不到。

### 3. Firefox
`Firefox`的做法，就是到中间页面触发一个`new_window_open`方法，并监听一个`new_window_close`方法。 在 `main.js` 中:
```javascript
// 新建一个新的窗口
var openWindow = function(info){
    console.log('OPEN WINDOW');

    var url = info.url;
    if (url.indexOf('http') == -1) {
        url = self.data.url(url);
    }

    var size = '';
    if (info.width && info.height) {
        size = ',width=' + info.width + ',height=' + info.height;
    }

    var popup = require('sdk/window/utils').openDialog({
        'features': Object.keys({
            'chrome': true,
            'centerscreen': true,
            'resizable': false,
            'scrollbars': true
        }).join() + size,
        name: info.id
    });

    popup.addEventListener('load', function() {
        tabs.activeTab.url = url;
    });
    popup.addEventListener('close', function() {
        page.port.emit('new_window_close', info.id);
    });
};
// http://stackoverflow.com/questions/22002010/addon-sdk-way-to-make-a-dialog
page.port.on('new_window_open', function(data) {
    openWindow(data.detail);
});
```
## 总结
所以第三方登录的弹窗，虽然每个浏览器都可以弹，但是对于`Safari`来说，因为 `cookie` 的原因，导致没办法校验是否真的登录成功。其他的两个浏览器没问题





