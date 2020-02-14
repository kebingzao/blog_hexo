---
title: 浏览器 extension 插件开发系列(12) -- 实现右键菜单推送消息
date: 2019-11-25 13:28:12
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
在这个插件有一个功能，挺有趣的，就是登陆完之后，就在网页上，右键菜单有一个`airdroid的推送选项`，通过可以选取文字，图片，网址来推送到手机端。点击之后，就会把这个东西推送到手机端，并且在网页上提示推送成功。 以 `Chrome` 的截图:

1. 直接在网页空白处点击，将当前网页的网址传送过去

![png](1.png)
<!--more-->
2. 在图片上点击会把图片的url传过去

![png](2.png)

3. 选择文字，会把所选的文字传过去

![png](3.png)

4. 在链接上右键点击选择，会把链接发过去

![png](4.png)

5. 推送成功，就会在当前的网页中显示的标记

![png](5.png)

6. 然后推送到手机上，在手机上的截图就是:

![png](6.png)

接下来讲一下这三个浏览器怎么实现右键菜单的情况。其中在 `Chrome` 还实现了一个给当前页面发送信息的功能(就是上图 `Pushed successfully` 这个提示)。

## 各浏览器实现右键菜单
在 `contextmenus.js` 中有定义，完整的代码如下:
```javascript
// 初始化右键推送菜单
(function () {
    Airdroid.ContextMenus = {
        _devicesObj: "",
        // 初始化
        init: function(deviceObjList){
            var self = this;
            this._devicesObj = deviceObjList;
            if(window.chrome){

            }else if(window.safari){

            }else{
                // firefox 桌面右键快捷菜单点击项
                Airdroid.Event.addEventListener(Airdroid.Event.FireFoxEvent.context_menu_item_clicked, function(info) {
                    var msg = "";

                    var message = info.message;
                    if (message.url) {
                        msg = message.url;
                    } else if (message.selection) {
                        msg = message.selection;
                    } else {
                        return;
                    }
                    // 最后推送
                    self.sendPush(msg, info.entry.id);
                });
            }
            // 更新右键菜单
            this.updateContextMenu();
        },
        // 移除右键菜单
        removeContextMenu: function(){
            if (window.safari) {

            } else if (window.chrome) {
                window.chrome.contextMenus.removeAll();
            } else {

            }
        },
        // 更新右键菜单
        updateContextMenu: function(){
            // 根据不同的浏览器来区分
            if (window.safari) {
                this.setUpSafariMenu();
            } else if (window.chrome) {
                this.setUpChromeMenu();
            } else {
                this.setUpFirefoxMenu();
            }
        },
        // safari 桌面右键菜单
        setUpSafariMenu: function(){
            var self = this;
            // 防止多次
            if (Airdroid.safariListenerAdded) {
                return;
            }

            Airdroid.safariListenerAdded = true;

            safari.application.addEventListener('contextmenu', function(e) {
                // 显示菜单项
                _.each(self._devicesObj, function(deviceObj){
                    e.contextMenu.appendContextMenuItem('push:' + deviceObj.getId(), 'Push to ' + deviceObj.getModel());
                });
            }, false);

            safari.application.addEventListener('command', function(e) {
                if (e instanceof SafariExtensionContextMenuItemCommandEvent) {

                    var device_id = e.target.command.split(':')[1];
                    var msg = '';
                    var userInfo = e.userInfo;
                    if (userInfo.selection.length > 0 && userInfo.tagName != 'A') {
                        msg = userInfo.selection;
                    } else if (userInfo.src) {
                        msg = userInfo.src;
                    } else {
                        msg = userInfo.url;
                    }
                    self.sendPush(msg, device_id);
                }
            }, false);
        },
        // firefox
        setUpFirefoxMenu: function(){
            var self = this;
            Airdroid.Event.dispatchEvent(Airdroid.Event.FireFoxEvent.update_context_menu);

            var entries = [];

            _.each(self._devicesObj, function(deviceObj){
                entries.push({
                    'type': 'device',
                    'label': deviceObj.getModel(),
                    'id': deviceObj.getId()
                });
            });
            Airdroid.Event.dispatchEvent(Airdroid.Event.FireFoxEvent.update_context_menu, entries);
        },
        // chrome
        setUpChromeMenu: function(){
            var self = this;
            // 先清掉之前的
            self.removeContextMenu();
            var contextMenuItemClicked = function(deviceObj, info, tab) {
                var msg = '';

                if (info.srcUrl) {
                    msg = info.srcUrl;
                } else if (info.linkUrl) {
                    msg = info.linkUrl;
                } else if (info.selectionText) {
                    msg = info.selectionText;
                } else {
                    msg = info.pageUrl;
                }
                // 最后推送
                self.sendPush(msg, deviceObj.getId());
            };
            // 要抓取的类型
            var contexts = ['page', 'link', 'selection', 'image'];

            _.each(self._devicesObj, function(deviceObj){
                window.chrome.contextMenus.create({
                    'title': deviceObj.getModel(),
                    'contexts': contexts,
                    'onclick': function(info, tab) {
                        contextMenuItemClicked(deviceObj, info, tab);
                    }
                });
            })
        },
        // 推送信息
        sendPush: function(msg, deviceId){
            var self = this;
            Airdroid.Server.sendNotePushMsg(msg, deviceId).done(function(){
                console.log("contextmenu push success");
                self.showToast("Pushed successfully");
            }).fail(function(){
                self.showToast("Pushed failed");
            });
        },
        // 在网页上显示推送结果
        showToast: function (text) {
            if (window.chrome) {
                window.chrome.tabs.query({ 'active': true, 'lastFocusedWindow': true }, function(tabs) {
                    if (tabs && tabs.length > 0) {
                        var tab = tabs[0];
                        console.log("show page toast");
                        window.chrome.tabs.sendMessage(tab.id, {
                            'type': 'show_toast',
                            'text': text
                        });
                    }
                });
            }
        }
    }
})();
```

## Chrome 实现右键菜单功能
主要是这个函数 `window.chrome.contextMenus`, 对应的代码是:
```javascript
// chrome
setUpChromeMenu: function(){
    var self = this;
    // 先清掉之前的
    self.removeContextMenu();
    var contextMenuItemClicked = function(deviceObj, info, tab) {
        var msg = '';

        if (info.srcUrl) {
            msg = info.srcUrl;
        } else if (info.linkUrl) {
            msg = info.linkUrl;
        } else if (info.selectionText) {
            msg = info.selectionText;
        } else {
            msg = info.pageUrl;
        }
        // 最后推送
        self.sendPush(msg, deviceObj.getId());
    };
    // 要抓取的类型
    var contexts = ['page', 'link', 'selection', 'image'];

    _.each(self._devicesObj, function(deviceObj){
        window.chrome.contextMenus.create({
            'title': deviceObj.getModel(),
            'contexts': contexts,
            'onclick': function(info, tab) {
                contextMenuItemClicked(deviceObj, info, tab);
            }
        });
    })
},
```
这边可以看到，它有四种类型，对应上面的四张图 (网址，图片，文字， 链接)， 这边还有一个细节。如果`create` 超过一个的话，就会变成二级菜单的方式:

![png](7.png)

如果只有一个的话，就会只有一个设备名:

![png](8.png)

同时 `manifest.json` 要有 `contextMenus` 的权限

![png](9.png)

## Safari 实现右键菜单操作
如果是在 `Safari` 中，实现右键菜单的功能的效果图如下:
1. 直接在网页空白处点击，将当前网页的网址传送过去

![png](10.png)

2. 在图片上点击会把图片的url传过去

![png](11.png)

3. 选择文字，会把所选的文字传过去

![png](12.png)

4. 在链接上右键点击选择，会把链接发过去

![png](13.png)

代码片段如下:
```javascript
// safari 桌面右键菜单
setUpSafariMenu: function(){
    var self = this;
    // 防止多次
    if (Airdroid.safariListenerAdded) {
        return;
    }

    Airdroid.safariListenerAdded = true;

    safari.application.addEventListener('contextmenu', function(e) {
        // 显示菜单项
        _.each(self._devicesObj, function(deviceObj){
            e.contextMenu.appendContextMenuItem('push:' + deviceObj.getId(), 'Push to ' + deviceObj.getModel());
        });
    }, false);

    safari.application.addEventListener('command', function(e) {
        if (e instanceof SafariExtensionContextMenuItemCommandEvent) {

            var device_id = e.target.command.split(':')[1];
            var msg = '';
            var userInfo = e.userInfo;
            if (userInfo.selection.length > 0 && userInfo.tagName != 'A') {
                msg = userInfo.selection;
            } else if (userInfo.src) {
                msg = userInfo.src;
            } else {
                msg = userInfo.url;
            }
            self.sendPush(msg, device_id);
        }
    }, false);
},
```
首先是监听 `contextmenu` 事件，也就是说，当用户在网页上点击右键的时候，就会将设备列表添加进去。接下来就监听一个事件`command`，也就是当用户点击右键菜单列表的时候会触发，那么是怎样触发的呢？

其实这个事件是从网页上传过来的，`Safari`可以在`plist`文件里面设置可以在网页上注入的js脚本的文件（这一点跟chrome还有点像）,如果在 `Info.plist` 文件中，有这一行:
```javascript
<string>js/sys/safari-end-script.js</string>
```
其实就是注入了这个脚本，`safari-end-script.js`, 内容如下:
```javascript
document.addEventListener('contextmenu', function(e) {
    var userInfo = {
        'selection': window.getSelection().toString(),
        'tagName': e.target.tagName
    };

    if (e.target.tagName == 'IMG') {
        userInfo.src = e.target.src;
    } else if (e.target.tagName == 'A') {
        userInfo.url = e.target.href;
    } else {
        userInfo.title = window.document.title;
        userInfo.url = window.location.href;
    }

    safari.self.tab.setContextMenuEventUserInfo(e, userInfo);
}, false);
```

也就是如果用户在页面上点击右键菜单的话，就会触发`contextmenu`，然后传递到背景页，然后在背景页中的监听事件判断这个是由`extension`生成的菜单点击过来的。

从调试窗口也可以看出来。

![png](14.png)

## Firefox 实现右键菜单操作
效果如下:
1. 直接在网页空白处点击，将当前网页的网址传送过去

![png](15.png)

2. 选择文字，会把所选的文字传过去

![png](16.png)

3. 在链接上右键点击选择，会把链接发过去

![png](17.png)

如果只有一个设备的话，那就是：

![png](18.png)

多个设备就是：

![png](19.png)

具体代码片段如下:
```javascript
// firefox
setUpFirefoxMenu: function(){
    var self = this;
    Airdroid.Event.dispatchEvent(Airdroid.Event.FireFoxEvent.update_context_menu);

    var entries = [];

    _.each(self._devicesObj, function(deviceObj){
        entries.push({
            'type': 'device',
            'label': deviceObj.getModel(),
            'id': deviceObj.getId()
        });
    });
    Airdroid.Event.dispatchEvent(Airdroid.Event.FireFoxEvent.update_context_menu, entries);
},
```
同时在初始化的时候，监听从中转页面过来的右键菜单点击事件：
```javascript
// firefox 桌面右键快捷菜单点击项
Airdroid.Event.addEventListener(Airdroid.Event.FireFoxEvent.context_menu_item_clicked, function(info) {
    var msg = "";

    var message = info.message;
    if (message.url) {
        msg = message.url;
    } else if (message.selection) {
        msg = message.selection;
    } else {
        return;
    }
    // 最后推送
    self.sendPush(msg, info.entry.id);
});
```
而这个`update_context_menu` 事件其实就是在 中转页面（`main.js`）操作的:
```javascript
// 显示桌面右键菜单
var menus = [];
page.port.on('update_context_menu', function(data) {
    var entries = data.detail;
    menus.map(function(menu) {
        menu.destroy();
    });

    menu = [];

    if (!entries || entries.length == 0) {
        return;
    }

    var pageMenu = contextMenu.Menu({
        'label': 'AirDroid',
        'image': self.data.url('images/icon_32.png'),
        'context': contextMenu.PageContext()
    });

    var linkMenu = contextMenu.Menu({
        'label': 'AirDroid',
        'image': self.data.url('images/icon_32.png'),
        'context': contextMenu.SelectorContext('a[href]')
    });

    var selectionMenu = contextMenu.Menu({
        'label': 'AirDroid',
        'image': self.data.url('images/icon_32.png'),
        'context': contextMenu.SelectionContext()
    });

    menus.push(pageMenu);
    menus.push(linkMenu);
    menus.push(selectionMenu);

    entries.map(function(entry) {
        pageMenu.addItem(contextMenu.Item({
            'label': entry.label,
            'image': entry.imageUrl,
            'contentScript': 'self.on("click", function(node, data) {' +
                    '    self.postMessage({ "title": document.title, "url": document.URL });' +
                    '});',
            'onMessage': function(message) {
                page.port.emit('context_menu_item_clicked', { 'entry': entry, 'message': message });
            }
        }));

        linkMenu.addItem(contextMenu.Item({
            'label': entry.label,
            'image': entry.imageUrl,
            'contentScript': 'self.on("click", function(node, data) {' +
                    '    self.postMessage({ "title": node.textContent, "url": node.href });' +
                    '});',
            'onMessage': function(message) {
                page.port.emit('context_menu_item_clicked', { 'entry': entry, 'message': message });
            }
        }));

        selectionMenu.addItem(contextMenu.Item({
            'label': entry.label,
            'image': entry.imageUrl,
            'contentScript': 'self.on("context", function (node) {' +
                    '    return node.nodeName != "A";' +
                    '});' +
                    'self.on("click", function(node, data) {' +
                    '    self.postMessage({ "selection": window.getSelection().toString() });' +
                    '});',
            'onMessage': function(message) {
                page.port.emit('context_menu_item_clicked', { 'entry': entry, 'message': message });
            }
        }));
    });
});
```
这样子在点击右键菜单的时候，就触发`context_menu_item_clicked`这个事件，然后把对应的内容传过去。

## Chrome 实现网页端脚本注入
在 `Chrome` 下如果在网页端用右键菜单推送消息的话，如果消息成功的话，就有回执到网页端的。 效果就是:

![png](5.png)

这个是因为`extension`有在网页上嵌入一段脚本，当推送成功的时候，就会把消息传到当前的页面。然后将`ui`渲染到当前的`tab`页面。

![png](20.png)

具体的代码片段如下:
```javascript
// 推送信息
sendPush: function(msg, deviceId){
    var self = this;
    Airdroid.Server.sendNotePushMsg(msg, deviceId).done(function(){
        console.log("contextmenu push success");
        self.showToast("Pushed successfully");
    }).fail(function(){
        self.showToast("Pushed failed");
    });
},
// 在网页上显示推送结果
showToast: function (text) {
    if (window.chrome) {
        window.chrome.tabs.query({ 'active': true, 'lastFocusedWindow': true }, function(tabs) {
            if (tabs && tabs.length > 0) {
                var tab = tabs[0];
                console.log("show page toast");
                window.chrome.tabs.sendMessage(tab.id, {
                    'type': 'show_toast',
                    'text': text
                });
            }
        });
    }
}
```
然后这个注入的脚本 `inject.js` 内容为:
```javascript
var Airdroid = window.Airdroid || {};

chrome.runtime.onMessage.addListener(function(message, sender, sendResponse) {
    if (message.type == 'show_toast') {
        Airdroid.showToast(message.text);
    }
});

Airdroid.showToast = function(text) {
    var toast = document.createElement('div');
    toast.setAttribute('style', '-webkit-transition:opacity .2s ease-in;opacity:0;position:absolute;bottom:50px;left:50%;width:300px;margin-left:-150px;z-index:16777270;color:white;text-align:center;background-color:rgba(0,0,0,.5);border-radius:30px;padding:10px;font-size:16px;');
    toast.textContent = text;

    document.body.insertBefore(toast, document.body.firstChild);

    setTimeout(function() {
        toast.style.opacity = 1;
    }, 10);

    setTimeout(function() {
        toast.style.opacity = 0;
    }, 2000);

    setTimeout(function() {
        document.body.removeChild(toast);
    }, 2200);
};

Airdroid.log = function(message) {
    chrome.runtime.sendMessage({ 'type': 'log', 'message': message });
};
```
而且这个还要在`manifest.json` 中配置:
```javascript
"content_security_policy": "script-src 'self' 'unsafe-eval'; object-src 'self'",
"content_scripts": [{
"matches": [ "http://*/*", "https://*/*" ],
"js": ["js/sys/inject.js"]
}],
```
这样就可以了。

## 总结
我们实现在浏览器中操作右键菜单，并发送 push 的功能。

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





