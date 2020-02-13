---
title: 浏览器 extension 插件开发系列(14) -- 点击reply出现回复小窗口
date: 2020-02-13 10:47:12
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
通过 {% post_link browser-extension-13 %} 我们知道在收到文本消息，或者文件消息的时候，这时候会有一个回复操作。点击，就会弹出一个回复小窗口

![png](1.png)
<!--more-->
![png](2.png)

然后输入要回复的消息，按`enter`直接回复。 所以我们要在打开这个`reply`窗体的时候，还要把这一条消息的内容传过去。 接下来我们讲讲怎么实现这个过程。

## 打开 reply 页面
打开回复窗口的代码在 `notificationsManage.js` 中:
```javascript
// 打开回复窗口
openQuickReply: function(obj){
    // 打开回复窗口
    var spec = {
        'url': 'reply.html?webuid=' + obj.webuid,
        'width': 320,
        'height': 420
    };

    var lastScreenX = Airdroid.Util.LocalStorage.getLocalStorageItem('quickReplyScreenX');
    var lastScreenY = Airdroid.Util.LocalStorage.getLocalStorageItem('quickReplyScreenY');
    if (lastScreenX && lastScreenY) {
        spec.top = parseInt(lastScreenY);
        spec.left = parseInt(lastScreenX);
    } else {
        spec.top = Math.floor((window.screen.availHeight / 2) - (spec.height / 2)) - 100;
        spec.left = Math.floor((window.screen.availWidth / 2) - (spec.width / 2)) + 100;
    }

    if (window.chrome) {
        spec.type = 'popup';
        spec.focused = true;

        var listener = function(message, sender, sendResponse) {
            chrome.runtime.onMessage.removeListener(listener);

            if (message.webuid == obj.webuid) {
                sendResponse(obj);
            }
        };

        chrome.runtime.onMessage.addListener(listener);

        chrome.windows.create(spec, function(window) {
            chrome.windows.update(window.id, { 'focused': true }, function() {
            });
        });
    } else if (window.safari) {
        var listener = function(e) {
            if (e.name == 'request_conversation_push') {
                // 这边要去掉里面的数组
                delete obj.buttons;
                console.log("接受其他页面的数据请求==>" + JSON.stringify(obj));
                e.target.page.dispatchMessage('conversation_push', JSON.stringify(obj));
                safari.application.removeEventListener('message', listener, false);
            }
        };

        safari.application.addEventListener('message', listener, false);

        var w = safari.application.openBrowserWindow();
        w.activeTab.url = safari.extension.baseURI + spec.url + '&width=' + spec.width + '&height=' + spec.height;
    } else {
        spec.notification = obj;
        // firefox
        Airdroid.Event.dispatchEvent(Airdroid.Event.FireFoxEvent.open_quickreply, spec);
    }
},
```
可以看到不同浏览器的打开处理方式又是不一样的。这个稍后分析，反正实现的效果肯定是一样。 当打开 `reply.html` 这个页面之后，接下来就进行一些逻辑操作，主要是在 `reply.js` 上:
```javascript
(function () {
    console.log("start reply");

    function getUrlParam (key, url) {
        key = key
                .replace(/[\[]/, "\\\[")
                .replace(/[\]]/, "\\\]");

        var r = new RegExp("[\\?&]" + key + "=([^&#]*)");
        var values = r.exec(url || window.location.href);
        return values == null ? "" : decodeURIComponent(values[1]);
    }

    if(getUrlParam("height")){
        window.resizeTo(parseInt(getUrlParam("width")), parseInt(getUrlParam("height")));
    }
    // 主程序
    var Reply = BasePage.extend({
        // 绑定事件
        events: {
            "keydown #reply": "replyHandle"
        },
        // 初始化程序
        afterInit: function () {
            var self = this;
            $("body").prepend(new Tpl('reply/reply').render());
            // 获取消息的id
            var webuid = getUrlParam("webuid");

            if (window.chrome) {
                chrome.runtime.sendMessage({ 'webuid': webuid }, function(response) {
                    self.render(response);
                });
            } else if (window.safari) {
                var listener = function(e) {
                    if (e.name == 'conversation_push') {
                        self.render(JSON.parse(e.message));
                        safari.self.removeEventListener('message', listener, false);
                    }
                };
                safari.self.addEventListener('message', listener, false);
                safari.self.tab.dispatchMessage('request_conversation_push');
            } else {
                // 触发请求数据事件
                self.eventObj.emit('request_conversation_push');
                self.eventObj.on('conversation_push', function(push) {
                    self.render(push);
                });
            }
        },
        // 渲染界面
        render: function(obj){
            this.options = obj;
            document.getElementById('container').style.display = 'block';

            document.getElementById('title').textContent = obj.title;
            document.getElementById('desc').textContent = 'Via AirDroid';

            var message = document.getElementById('message');
            message.textContent = obj.message;
            message.scrollTop = message.scrollHeight;

            var heightNeeded = document.getElementById('container').offsetHeight - window.innerHeight;
            window.resizeBy(0, heightNeeded);
        },
        // 回复操作
        replyHandle: function(e){
            var self = this;
            if (e.keyCode == 13 && !e.shiftKey) {
                if (document.getElementById('reply').value.length > 0) {
                    var msg = document.getElementById('reply').value;
                    var cb = function(){
                        setTimeout(function() {
                            if(window.chrome || window.safari){
                                window.close();
                            }else{
                                // 关闭窗口
                                self.eventObj.emit("close");
                            }
                        }, 120);
                    };
                    if(this.options.pushType == self.Airdroid.PUSH_TYPE.NOTIFICATION){
                        // 发送信息
                        self.server.sendIMMsg(this.options, msg).done(function(){
                            cb();
                        });
                    }else{
                        // 发送信息
                        self.server.sendNotePushMsg(msg, this.options.deviceId).done(function(){
                            cb();
                        });
                    }
                }
                return false;
            }
        }
    });
    Reply.init();
    // 关掉的时候，释放资源
    window.onunload = function(){

    };
})();
```
其实逻辑非常简单，除了渲染页面之外，就只有一个回复操作的监听。
## Chrome 打开小窗口并请求数据
对于`Chrome` 来说，打开一个新窗口直接用 `chrome.windows.create`。 然后在背景页开启一个监听，如果过来的`webuid`一样，就把消息内容传过去, 这样就可以在 `reply` 页面初始化的时候，请求数据的时候，背景页就可以将一些内容传过去了:
```javascript
var listener = function(message, sender, sendResponse) {
    chrome.runtime.onMessage.removeListener(listener);

    if (message.webuid == obj.webuid) {
        sendResponse(obj);
    }
};

chrome.runtime.onMessage.addListener(listener);
```
然后在 `reply.js` 初始化完成之后，就开始请求数据:
```javascript
chrome.runtime.sendMessage({ 'webuid': webuid }, function(response) {
    self.render(response);
});
```
## Safari 打开小窗口并请求数据
对于`Safari`来说，调用`safari.application.openBrowserWindow()` 来开一个新窗口，然后触发`request_conversation_push`并监听`conversation_push` 从而实现数据交换，所以在背景页是这样子的:
```javascript
var listener = function(e) {
    if (e.name == 'request_conversation_push') {
        // 这边要去掉里面的数组
        delete obj.buttons;
        console.log("接受其他页面的数据请求==>" + JSON.stringify(obj));
        e.target.page.dispatchMessage('conversation_push', JSON.stringify(obj));
        safari.application.removeEventListener('message', listener, false);
    }
};

safari.application.addEventListener('message', listener, false);
```
然后前端页面在初始化之后，直接触发`request_conversation_push`来像背景页请求数据,并且通过监听`conversation_push`来得到背景页返回的数据:
```javascript
var listener = function(e) {
    if (e.name == 'conversation_push') {
        self.render(JSON.parse(e.message));
        safari.self.removeEventListener('message', listener, false);
    }
};
safari.self.addEventListener('message', listener, false);
safari.self.tab.dispatchMessage('request_conversation_push');
```
## Firefox 打开小窗口并请求数据
对于`Firefox`来说，打开窗口也是在中转页面（`main.js`）进行的 
```javascript
Airdroid.Event.dispatchEvent(Airdroid.Event.FireFoxEvent.open_quickreply, spec);
```
具体代码如下:
```javascript
// 打开快速回复窗口
// http://stackoverflow.com/questions/22002010/addon-sdk-way-to-make-a-dialog
page.port.on('open_quickreply', function(data) {
    var info = data.detail;
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
        name: info.notification.webuid
    });

    popup.addEventListener('load', function() {
        tabs.activeTab.on('ready', function(tab) {
            var worker = tab.attach({
                'contentScriptFile': [
                    self.data.url('js/lib/jquery/jquery.js'),
                    self.data.url('js/lib/jquery/jquery.toast.js'),
                    self.data.url('js/lib/underscore/underscore.js'),
                    self.data.url('js/util/tplHelper.js'),
                    self.data.url('js/web/base.js'),
                    self.data.url('js/web/reply.js')
                ]
            });

            attachListeners(worker);
            // 请求对应的数据
            worker.port.on('request_conversation_push', function() {
                worker.port.emit('conversation_push', info.notification);
            });

            worker.port.on('close', function(reply) {
                popup.close();
            });
        });

        tabs.activeTab.url = url;
    });
});
```
这边还要重新加载跟`reply.html`一样的js文件。 通信方式开始在这个方法里面设置的：
```javascript
// 请求对应的数据
worker.port.on('request_conversation_push', function() {
    worker.port.emit('conversation_push', info.notification);
});

worker.port.on('close', function(reply) {
    popup.close();
});
```
前端页面只要触发就行了:
```javascript
self.eventObj.emit('request_conversation_push');
self.eventObj.on('conversation_push', function(push) {
    self.render(push);
});
```
而 `eventObj` 就是 `Firefox` 的 `self.port`。

## 总结
本节实现了怎么在各个浏览器打开一个新的窗体页面，并且实现数据交换。

