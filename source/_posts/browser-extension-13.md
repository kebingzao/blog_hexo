---
title: 浏览器 extension 插件开发系列(13) -- 实现消息过来出现桌面通知
date: 2020-02-12 17:26:22
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
在做插件的时候，如果消息或者通知过来的时候，就会出现桌面通知，如果点击会有一些操作，比如直接回复，或者打开聊天窗口。但是不同的浏览器的实现和展示不一样。

但是总的要实现的过程如下:(这边以chrome插件的实现过程来截图,其他浏览器的截图会在各自的小节里面):

主要是有三种：
### 1. 通知消息
<!--more-->
![png](1.png)

桌面通知如下：

![png](2.png)

### 2. 文本消息

![png](3.png)

点击`reply`, 出现小小的回复框

![png](4.png)

输入之后，直接按回车，就发送

### 3. 发送文件消息

![png](5.png)

点击reply

![png](6.png)

也可以在 `receive` 里面看到

![png](7.png)

点击上面的“点击下载“，就会调用一个显示页面去

![png](8.png)

以上这三种都是在 websocket 通道传过来的。而 websocket 通道是在用户登录之后连接上的。

![png](9.png)

这个具体代码太具有业务了。这边不细讲了。只要知道这些消息的推送都是由服务端通过 websocket 通道推送给这个插件端，然后插件端再去处理，不管是显示还是回复。上面的操作涉及到的要点非常多。一节讲不完，本节主要讲当插件接收到消息推送的时候，怎么显示桌面通知，比如这个效果:

![png](2.png)

包括下面的选项对应的点击操作等等。还是一样，每个浏览器分析一遍。
## 1. Chrome 实现消息过来出现桌面通知
从业务我们可以知道，当 websocket 通道收到通知或者文本，文件消息的时候，就要弹出桌面通知。 所以这边是有三种类型的，对应下面的三种桌面通知的显示效果：
1. 通知

![png](2.png)

2. 文本信息

![png](10.png)

3. 文件消息

![png](5.png)

所以我们这边针对 websocket 通道过来的信息，是有进行分类的:
```javascript
var notificationObj = new Airdroid.Notification(dataMsgBody);
Airdroid.NotificationManage.create(notificationObj);
```
而这个 `create` 方法就是:
```javascript
// 创建一个桌面通知
create: function(notificationObj){
    if(!this._listenersSetUp){
        this.setUpNotificationListeners();
        this._listenersSetUp = true;
    }
    // 加入到对象中
    if(window.chrome){
        this.createInChrome(notificationObj);
    }else if(window.safari){
        this.createInSafari(notificationObj);
    }else{
        this.createInFirefox(notificationObj);
    }
},
```

这边有用一个 model 类来管理 `model/notification.js`
```javascript
// 管理推送过来的消息
(function () {
    var AirdroidNotification = function (data) {
        var self = this;
        var obj = $.extend(true,{}, this._defaultOptions);
        obj.webuid = data.webuid;
        obj.pushType = data.pushType;
        switch (data.pushType){
            case Airdroid.PUSH_TYPE.NOTE:
                // 文本消息
                obj = $.extend(obj,{
                    iconUrl: 'images/msg.png',
                    title: 'Push Note via AirDroid',
                    message: data.msg,
                    deviceId: data.channel_id,
                    time: data.pid
                });
                // 加上回复选项
                obj.buttons.push({
                    'title': 'reply',
                    'iconUrl': 'images/ic_action_sms.png',
                    'type': '3',
                    'onclick': function() {
                        console.log("点击回复");
                        // 打开回复窗口
                        self.quickReply();
                    }
                });
                // 如果是Firefox，点击就会出现回复
                if(Airdroid.Util.Browser.firefox || Airdroid.Util.Browser.safari){
                    obj.onclick = function(){
                        // 跳转到相应地址
                        self.quickReply();
                    };
                }
                break;
            case Airdroid.PUSH_TYPE.FILE:
                // 文件消息
                obj = $.extend(obj,{
                    iconUrl: data.thumbnail,
                    title: 'Push File via AirDroid',
                    message: data.filename,
                    deviceId: data.channel_id,
                    time: data.webuid,
                    //特有的 下载地址
                    url: data.url
                });
                // 加上回复选项
                obj.buttons.push({
                    'title': 'reply',
                    'iconUrl': 'images/ic_action_sms.png',
                    'type': '3',
                    'onclick': function() {
                        console.log("点击回复");
                        // 打开回复窗口
                        self.quickReply();
                    }
                });
                // 如果是Firefox，点击就会出现回复
                if(Airdroid.Util.Browser.firefox || Airdroid.Util.Browser.safari){
                    console.log("点击回复");
                    obj.onclick = function(){
                        // 跳转到相应地址
                        self.quickReply();
                    };
                }
                break;
            case Airdroid.PUSH_TYPE.NOTIFICATION:
                // 手机通知
                var appinfo = JSON.parse(data.appinfo);
                    //data.actionTitles = ["test1", "test2"];
                obj = $.extend(obj,{
                    iconUrl: data.icon,
                    title: data.title,
                    message: data.content,
                    deviceId: data.deviceId,
                    time: data.time,
                    // 特有的
                    id: data.id,
                    nid: data.nid,
                    appName: appinfo.name,
                    apkId: data.apkid
                });
                // 判断是否可回复
                if(data.quick_reply){
                    obj.buttons.push({
                        'title': 'reply',
                        'iconUrl': 'images/ic_action_sms.png',
                        'type': '3',
                        'onclick': function() {
                            console.log("点击回复");
                            // 打开回复窗口
                            self.quickReply();
                        }
                    });
                }
                // 加上勿扰选项
                obj.buttons.push({
                    'title': 'Mute ' + obj.appName,
                    'iconUrl': 'images/ic_action_halt.png',
                    'type': '2',
                    'onclick': function() {
                        console.log("点击勿扰");
                        self.mute();
                    }
                });
                // 接下来加上 quickaction
                if(data.actionTitles && _.isArray(data.actionTitles)){
                    _.each(data.actionTitles,function(title){
                        obj.buttons.push({
                            'title': "Android: " + title,
                            'iconUrl': 'images/ic_action_android.png',
                            'type': '4',
                            'short_title': title,
                            'onclick': function() {
                                console.log("点击quickaction选项");
                                self.quickAction(title);
                            }
                        });
                    })
                }
                // 加上点击事件
                obj.onclick = function(){
                    // 跳转到相应地址
                    if(Airdroid.AndroidMapping[data.apkid]){
                        Airdroid.Util.Tab.openTab(Airdroid.AndroidMapping[data.apkid]);
                    }
                };
                // 加上一个擦除的选项
                obj.buttons.push({
                    'title': 'dismiss',
                    'iconUrl': 'images/ic_action_cancel.png',
                    'type': '1',
                    'onclick': function() {
                        self.dismiss();
                    }
                });
                break;
        }
        // 加上描述
        if(obj.deviceId){
            var deviceObj = Airdroid.Account.getDeviceObjById(obj.deviceId);
            obj.contextMessage = "{0} via AirDroid at {1}".format(deviceObj.getModel(),
                    new Date().toLocaleTimeString().replace(/:\d+ /, ' '));
        }
        this.options = obj;
    };

    AirdroidNotification.prototype = {
        // 默认的属性
        _defaultOptions : {
            webuid: '',
            buttons:[],
            iconUrl: '',
            message: '',
            priority: 2,
            title: '',
            contextMessage: "",
            type: "basic"
        },
        // 触发变化
        triggerChange: function(){
            var self = this;
            setTimeout(function(){
                Airdroid.Event.dispatchEvent(self.options.pushType == Airdroid.PUSH_TYPE.NOTIFICATION ? Airdroid.Event.TYPE.notification_change : Airdroid.Event.TYPE.push_change);
            },200);
        },
        // 设置勿扰
        mute: function(){
            Airdroid.NotificationManage.setAddBlockList(this.getId());
            this.triggerChange();
        },
        // 清除手机操作
        dismiss: function(){
            Airdroid.NotificationManage.setDismiss(this.getId());
            this.triggerChange();
        },
        // 通知项点击回复
        quickReply: function(){
            var self = this;
            Airdroid.NotificationManage.showQuickReply(this.getId());
            this.triggerChange();
        },
        // quickaction操作
        quickAction: function(title){
            Airdroid.NotificationManage.setQuickAction(this.getId(), title);
            this.triggerChange();
        },
        getTimeOnScreen: function() {
            return this.options.priority > 0 ? 25 * 1000 : 8 * 1000;
        },
        // 获取唯一id
        getId: function(){
            return this.options.webuid;
        },
        // 获取属性值
        getOptions: function(){
            return this.options;
        },
        // 销毁时触发的事件
        destroy: function(){
            console.log("消息关闭，id为：" + this.getId())
        },
        // 显示的时候
        display: function(){

        }
    };

    Airdroid.Notification = AirdroidNotification;
})();
```
可以看到，这个 `model` 对不同消息的处理不一样，甚至针对不同浏览器还有差别的， 因为本节讲的是 `Chrome`，所以看看 `Chrome` 是怎么创建通知的? 所有通知的创建和管理都在 `notificationManage.js` 这个对象中, 在 `Chrome` 上创建一个通知的代码如下：
```javascript
// 在chrome上创建一个通知
createInChrome: function(notificationObj){
    var self = this;
    try{
        var n_options = notificationObj.getOptions();
        var options = this.getNotificationOptions(n_options);
        // 这时候要进行去重判断, 条件是如果包名一样，并且是普通通知
        // 如果是普通通知，那么弹出框的id就是包名
        var key = n_options.pushType == Airdroid.PUSH_TYPE.NOTIFICATION ? n_options.apkId : notificationObj.getId();
        // 获取当前存在的通知
        chrome.notifications.getAll(function(active) {
            var exists = active && active[key];
            // 如果存在
            if (exists) {
                self._notificationsObj[key] = notificationObj;
                try {
                    // 更新
                    chrome.notifications.update(key, options, function() {
                        notificationObj.display();
                    });
                } catch (e) {
                    if (e.message.indexOf('contextMessage') >= 0) {
                        delete options.contextMessage;
                        chrome.notifications.update(key, options, function() {
                            notificationObj.display();
                        });
                    } else {
                        throw e;
                    }
                }
            } else {
                // 创建成功之后的回调
                var notificationCreated = function() {
                    self._notificationsObj[key] = notificationObj;
                    notificationObj.display();
                    console.log("通知过来了=====");
                };
                // 开始创建
                var createNotification = function() {
                    try {
                        chrome.notifications.create(key, options, function() {
                            if (chrome.runtime.lastError) {
                                console.log(chrome.runtime.lastError);
                            }
                            notificationCreated();
                        });
                    } catch (e) {
                        if (e.message.indexOf('contextMessage') >= 0) {
                            delete options.contextMessage;
                            chrome.notifications.create(key, options, function() {
                                notificationCreated();
                            });
                        } else {
                            throw e;
                        }
                    }
                };
                createNotification();
            }
        });
    }catch(e){
        console.log("出错了");
    }
    // 7秒后关闭
//            setTimeout(function(){
//                notification.close();
//            },7000);
},

// 设置监听项
setUpNotificationListeners: function(){
    var self = this;
    if(window.chrome){
        // 监听整体的点击事件
        chrome.notifications.onClicked.addListener(function(key) {
            var notificationObj = self._notificationsObj[key];
            var options = notificationObj.getOptions();
            chrome.notifications.clear(key, function(wasCleared) {
                notificationObj.destroy();
            });
            if (options) {
                if (options.onclick) {
                    options.onclick();
                }
            }
        });
        // 监听关闭事件
        chrome.notifications.onClosed.addListener(function(key, byUser) {
            var notificationObj = self._notificationsObj[key];
            var options = notificationObj.getOptions();
            if (options) {
                if (options.onclose) {
                    options.onclose();
                }
            }
            notificationObj.destroy();
            // 删掉
            delete self._notificationsObj[key];
        });
        // 监听按钮的点击事件
        chrome.notifications.onButtonClicked.addListener(function(key, index) {
            var notificationObj = self._notificationsObj[key];
            var options = notificationObj.getOptions();
            chrome.notifications.clear(key, function(wasCleared) {

            });

            if (options) {
                options.buttons[index].onclick();
            }
        });
    }else if(window.safari){

    }else{
        // firefox 桌面通知点击事件
    }
},
```
注意，这边有个变量`_notificationsObj` 用来存储所有的桌面通知，然后在通知关闭的时候，再把这条通知从数组里面去掉。而且`Chrome`还支持对通知进行内容更新，如果该通知存在的话。 同时要绑定通知的点击事件，按钮的点击事件和关闭事件，在`options`参数里面已经绑定好了，只要直接触发`onclick`事件就行了。

从上面的代码可以看到，之所以3种消息的类型不一样，是因为传入的`options`不一样，这个是在`notification.js` 设置的:
### 1.1. 应用通知
当消息为应用通知的时候，代码是这样的
```javascript
var appinfo = JSON.parse(data.appinfo);
//data.actionTitles = ["test1", "test2"];
obj = $.extend(obj,{
    iconUrl: data.icon,
    title: data.title,
    message: data.content,
    deviceId: data.deviceId,
    time: data.time,
    // 特有的
    id: data.id,
    nid: data.nid,
    appName: appinfo.name,
    apkId: data.apkid
});
// 判断是否可回复
if(data.quick_reply){
    obj.buttons.push({
        'title': 'reply',
        'iconUrl': 'images/ic_action_sms.png',
        'type': '3',
        'onclick': function() {
            console.log("点击回复");
            // 打开回复窗口
            self.quickReply();
        }
    });
}
// 加上勿扰选项
obj.buttons.push({
    'title': 'Mute ' + obj.appName,
    'iconUrl': 'images/ic_action_halt.png',
    'type': '2',
    'onclick': function() {
        console.log("点击勿扰");
        self.mute();
    }
});
// 接下来加上 quickaction
if(data.actionTitles && _.isArray(data.actionTitles)){
    _.each(data.actionTitles,function(title){
        obj.buttons.push({
            'title': "Android: " + title,
            'iconUrl': 'images/ic_action_android.png',
            'type': '4',
            'short_title': title,
            'onclick': function() {
                console.log("点击quickaction选项");
                self.quickAction(title);
            }
        });
    })
}
// 加上点击事件
obj.onclick = function(){
    // 跳转到相应地址
    if(Airdroid.AndroidMapping[data.apkid]){
        Airdroid.Util.Tab.openTab(Airdroid.AndroidMapping[data.apkid]);
    }
};
// 加上一个擦除的选项
obj.buttons.push({
    'title': 'dismiss',
    'iconUrl': 'images/ic_action_cancel.png',
    'type': '1',
    'onclick': function() {
        self.dismiss();
    }
});
```
对于应用通知，有两个最基本的，就是 `勿扰`和`擦除` 这两个操作按钮

![png](11.png)

还有两个可用选项，就看推送过来的消息有没有这个配置，一个是`快速回复`，比如`WhatsApp`之类的应用，可用直接回复。

![png](12.png)

另外一个是是否允许`quickaction`操作(也是一种快捷操作)。

![png](13.png)

然后每一个`button`都会绑定对应的`onclick`事件

### 1.2. 文本消息
文件消息就比较简单，只有一个回复选项
```javascript
obj = $.extend(obj,{
    iconUrl: 'images/msg.png',
    title: 'Push Note via AirDroid',
    message: data.msg,
    deviceId: data.channel_id,
    time: data.pid
});
// 加上回复选项
obj.buttons.push({
    'title': 'reply',
    'iconUrl': 'images/ic_action_sms.png',
    'type': '3',
    'onclick': function() {
        console.log("点击回复");
        // 打开回复窗口
        self.quickReply();
    }
});
```
点击直接打开回复窗口。
### 1.3. 发送文件消息
跟文本消息很像
```javascript
obj = $.extend(obj,{
    iconUrl: data.thumbnail,
    title: 'Push File via AirDroid',
    message: data.filename,
    deviceId: data.channel_id,
    time: data.webuid,
    //特有的 下载地址
    url: data.url
});
// 加上回复选项
obj.buttons.push({
    'title': 'reply',
    'iconUrl': 'images/ic_action_sms.png',
    'type': '3',
    'onclick': function() {
        console.log("点击回复");
        // 打开回复窗口
        self.quickReply();
    }
});
```
不过左边的那个图片，要换成对应文件的缩略图。

最后`manifest.json` 要有 `notifications` 权限:

![png](14.png)

## 2. Firefox实现消息过来出现桌面通知
`Firefox` 浏览器的实现流程如下，也是有3种类型:
1. 通知消息

![png](15.png)

2. 文本消息

![png](16.png)

不过这边有个细节，这边没有`reply` 操作按钮(这个跟 `Chrome` 不一样)， 只有 popup 页面上的那个才有， 但是点击的桌面通知的时候，还是相当于点击`reply`

![png](17.png)

就会出现小小的回复框

![png](18.png)

输入之后，直接按回车，就发送

3. 文件消息

![png](19.png)

也没有`reply`操作按钮，但是点击整个桌面通知，就相当于`reply`操作，即弹出一个回复窗口。 也可以在 `popup` 页面的 `receive tab` 里面看到

![png](20.png)

点击上面的“点击下载“，就会调用一个显示页面去。

![png](21.png)

不过可以看出一个差别，就是 `Firefox` 的桌面通知是没有像 `Chrome` 那样下面可以设置一些操作按钮，他只有一个针对这个通知的点击事件。 这些都会在 `notification.js` 这个 `model` 类继续处理，比如这样子:
```javascript
// 如果是Firefox，点击就会出现回复
if(Airdroid.Util.Browser.firefox || Airdroid.Util.Browser.safari){
    console.log("点击回复");
    obj.onclick = function(){
        // 跳转到相应地址
        self.quickReply();
    };
}
```
如果是 `Firefox` 或者是 `Safari`， 就只绑定通知的绑定事件了，没有那么多花里胡哨的按钮了。这边的封装跟 `Chrome` 是一样的:
```javascript
var notificationObj = new Airdroid.Notification(dataMsgBody);
Airdroid.NotificationManage.create(notificationObj);
```
接下来看看`Firefox`是怎么创建通知的:
```javascript
// firefox 出桌面通知
createInFirefox: function(notificationObj){
    var self = this;
    var n_options = notificationObj.getOptions();
    // 这时候要进行去重判断, 条件是如果包名一样，并且是普通通知
    // 如果是普通通知，那么弹出框的id就是包名
    var key = n_options.pushType == Airdroid.PUSH_TYPE.NOTIFICATION ? n_options.apkId : notificationObj.getId();

    var spec = {
        'key': key,
        'title': n_options.title,
        'message': n_options.message,
        'contextMessage': n_options.contextMessage,
        'iconUrl': n_options.iconUrl
    };

    if (spec.message.length > 500) {
        spec.message = spec.message.substring(0, 500);
    }
    // 添加
    self._notificationsObj[key] = notificationObj;

    Airdroid.Event.dispatchEvent(Airdroid.Event.FireFoxEvent.show_notification, spec);

    Airdroid.Event.dispatchEvent(Airdroid.Event.FireFoxEvent.notifications_changed, _.keys(self._notificationsObj));
},

// 设置监听项
setUpNotificationListeners: function(){
    var self = this;
    if(window.chrome){
        // chrome ..
    }else if(window.safari){

    }else{
        // firefox 桌面通知点击事件
        Airdroid.Event.addEventListener(Airdroid.Event.FireFoxEvent.notification_clicked, function(key) {
            var notificationObj = self._notificationsObj[key];
            var options = notificationObj.getOptions();
            _.isFunction(options.onclick) && options.onclick();
            delete self._notificationsObj[key];
            // 清除角标
            Airdroid.Event.dispatchEvent(Airdroid.Event.FireFoxEvent.notifications_changed, _.keys(self._notificationsObj));
        });
    }
},
```
从上面代码可以看到，`Firefox`触发`show_notification`这个事件来显示通知，同时监听`notification_clicked`这个事件当桌面通知被点击的时候。那么这两个事件是怎么运行的呢，其实都在中转页面（`main.js`）中来操作的，也就是说，其实`Firefox`创建桌面通知是在中转页面来做的:
```javascript
// 通知过来，显示桌面图标
page.port.on('show_notification', function(data) {
    var notification = data.detail;
    if (notification.iconUrl.lastIndexOf('', 0) == 0) {
        notification.iconUrl = self.data.url(notification.iconUrl);
    }

    if (notification.contextMessage) {
        notification.message = notification.contextMessage + '\n' + notification.message;
    }

    if (prefs.dont_show_mirrors) {
        console.log('Not showing notification, disabled in preferences');
        return;
    }

    var notifications = require('sdk/notifications');
    notifications.notify({
        'title': notification.title,
        'text': notification.message,
        'iconURL': notification.iconUrl,
        'onClick': function() {
            // 通知点击事件
            page.port.emit('notification_clicked', notification.key);
        }
    });
});

// 通知条数改变，就在图标上面显示小红点
page.port.on('notifications_changed', function(data) {
    var notifications = data.detail;
    if (firefoxVersion >= 36 && widget) {
        var count = notifications.length;
        if (count > 0) {
            widget.badge = count;
        } else {
            widget.badge = '';
        }
    }
});
```
同时还有一个很有意思的地方，就是可以清掉或者设置上面的小红点 `notifications_changed`

![png](22.png)

上面代码的`widget` 就是那个小图标对应的对象。 从上面的截图来看，无论是应用通知，还是文件，文本消息。都是没有操作栏，最多绑定整体的点击事件，对于文本，文件，点击就会出现回复窗口:
```javascript
// 如果是Firefox，点击就会出现回复
if(Airdroid.Util.Browser.firefox || Airdroid.Util.Browser.safari){
  obj.onclick = function(){
    // 跳转到相应地址
    self.quickReply();
  };
}
```
## 3. Safari 实现消息过来出现桌面通知
`Safari` 浏览器的实现流程:
1. 通知消息

![png](23.png)

2. 文本消息

![png](24.png)

也是没有`reply`， 只有组件上的那个才有， 但是点击的桌面通知的时候，还是相当于点击`reply`

![png](25.png)

出现小小的回复框

![png](26.png)

输入之后，直接按回车，就发送

3. 文件消息

![png](27.png)

也没有`reply`操作按钮，但是点击整个桌面通知，就相当于`reply`操作，即弹出一个回复窗口, 也可以在 `popup` 页面的 `receive tab`里面看到:

![png](28.png)

点击上面的“点击下载“，就会调用一个显示页面去

![png](29.png)

封装的方式，跟之前一样， 我们看下 `Safari` 是怎么创建通知的:
```javascript
// safari出现桌面通知
createInSafari: function(notificationObj){
    var self = this;
    if (!Notification) {
        Airdroid.log('Notifications not available');
        return;
    }
    var n_options = notificationObj.getOptions();
    // 这时候要进行去重判断, 条件是如果包名一样，并且是普通通知
    // 如果是普通通知，那么弹出框的id就是包名
    var key = n_options.pushType == Airdroid.PUSH_TYPE.NOTIFICATION ? n_options.apkId : notificationObj.getId();

    var notification = new Notification(n_options.title, {
        'body': n_options.message,
        'tag': key
    });

    self._notificationsObj[key] = notificationObj;


    notification.onclick = function() {
        console.log("点击通知＝＝＝");
        if (n_options.onclick) {
            n_options.onclick();
        }

        notification.close();
    };

    notification.onclose = function() {
        console.log("关闭通知＝＝＝");
        if (n_options.onclose) {
            n_options.onclose();
        }

        delete self._notificationsObj[key];
    };
},
```
跟 `Firefox` 一样，无论是应用通知，还是文件，文本消息。都是都是每一操作栏，最多绑定整体的点击事件:
```javascript
// 如果是Firefox，点击就会出现回复
if(Airdroid.Util.Browser.firefox || Airdroid.Util.Browser.safari){
  obj.onclick = function(){
    // 跳转到相应地址
    self.quickReply();
  };
}
```

## 总结
本节主要是讲了这三个浏览器是怎么显示桌面通知的



















