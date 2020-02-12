---
title: 浏览器 extension 插件开发系列(09) -- popup以及其他前端页面的启动
date: 2020-02-11 16:48:29
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
在本插件中，`popup`页面以及其他前端页面的启动都差不多。

ps: 也有一些小差别，就是`Safari`下 `popup`页面和其他的前端页面获取`background`对象的形式不一样。{% post_link browser-extension-7 %}

## popup.html
以 `popup.html` 页面为例, 代码如下:
<!--more-->
```html
<!doctype html>
<head>
<meta charset="UTF-8">
<title>AirDroid</title>
<!-- build:css css/popup.min.css -->
<link rel="stylesheet" type="text/css" href="js/lib/bootstrap/css/bootstrap.css"/>
<link rel="stylesheet" type="text/css" href="js/lib/bootstrap/css/bootstrap-theme.css"/>
<link rel="stylesheet" type="text/css" href="styles/css/popup.css"/>
<!-- /build -->
</head>
<body>
<!-- build:js js/libs.min.js -->
<script type="text/javascript" src="js/lib/jquery/jquery.js"></script>
<script type="text/javascript" src="js/lib/jquery/jquery.toast.js"></script>
<script type="text/javascript" src="js/lib/jquery/jquery.clipboard.js"></script>
<script type="text/javascript" src="js/lib/underscore/underscore.js"></script>
<!-- /build -->
<!-- build:js js/popup.min.js -->
<script type="text/javascript" src="js/util/tplHelper.js"></script>
<!--<script type="text/javascript" src="tpls/jst.js"></script>-->
<script type="text/javascript" src="js/web/base.js"></script>
<script type="text/javascript" src="js/web/popup.js"></script>
<script type="text/javascript" src="js/lib/bootstrap/js/bootstrap.js"></script>
<!-- /build -->
</body>
```
这边除了 `base.js` 和 `popup.js` 这个两个业务逻辑js，其他都是第三方库，因此只要看这两个js就行了。其中 `popup` 又是继承 `base`，因此先看 `base` 这个基类。

## base.js 代码
```javascript
;
(function () {
    //todo 这边为了兼容Firefox extension 下的前后台调用，所有在前台调用后台的函数，都必须是用defer方式来调用的
    var fireFoxEvent =  (self && self.port) || {};
    window.BasePage = {
        // 统计信息
        rollingLog: [],
        funCbObj: {},
        extend: function (child) {
            return $.extend(true, {}, this, child);
        },

        init: function () {
            var self = this,
                    args = arguments;
            $(document).ready(function () {
                // 这时候要根据浏览器来获取背景对象
                if (window.chrome) {
                    var bg = chrome.extension.getBackgroundPage();
                    self.setUpBackgroundPage(bg);
                    self.inited(args);
                } else if (window.safari) {
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
                    }
                } else {
                    // 这个是要传过来的事件。所有的事件都可以自己定
                    fireFoxEvent.on('page_data', function(data) {
                        // 传过来 AirDroid对象
                        self.setUpBackgroundPage(data.detail);
                        // 接下来绑定可以传递到前台的触发事件
                        _.each(_.values(self.EventType),function(item){
                            fireFoxEvent.emit("page_register_handle",{name:item});
                        });
                        self.inited(args);
                    });
                    // 触发页面初始化事件，让后台传递AirDroid数据过来
                    fireFoxEvent.emit('page_init');
                }
            });
        },
        // 初始化结束的处理事件
        inited: function(args){
            var self = this;
            // 绑定事件
            self.delegateEvents(self.events);
            var browser = "";
            if(window.chrome){
                browser = "chrome";
            }else if(window.safari){
                browser = "safari";
            }else {
                browser = "firefox";
            }
            $("body").addClass(browser);
            self.afterInit && self.afterInit.apply(self, args);
        },
        //设置弹出框的大小
        setBodySize: function(width, height){
            var self = this;
            if(self.Airdroid.Util.Browser.chrome){

            }else if(self.Airdroid.Util.Browser.safari){
                width && (safari.extension.popovers[0].width = width);
                height && (safari.extension.popovers[0].height = height);
            }else {
                // firefox
                this.eventObj.emit("resize",{
                    width: width,
                    height: height
                })
            }
        },
        // 设置背景对象
        setUpBackgroundPage: function(data, dispatcher){
            var self = this;
            this.Airdroid = data.Airdroid;
            window.Airdroid = this.Airdroid;
            // 事件对象
            if(window.chrome){
                this.eventObj = data;
            }else if(window.safari){
                this.eventObj = data;
                if(dispatcher){
                    // 说明是非popover页面
                    // 同时要绑定对应的事件触发器
                    var addSafariTriggerHandle = function(obj){
                        if(obj.funListenerObjs){
                            _.each(obj.funListenerObjs,function(funId,name){
                                // 重新定义函数
                                obj[name] = function(){
                                    console.log("调用函数，函数名为%s, id为 %s", name, funId);
                                    var defer = $.Deferred();
                                    // 加入回调队列
                                    var deferDoneId = _.uniqueId("defer_done_" + name + "_");
                                    var deferFailId = _.uniqueId("defer_fail_" + name + "_");
                                    self.funCbObj[deferDoneId] = function(data){
                                        self.log(deferDoneId);
                                        defer.resolve(data);
                                    };
                                    self.funCbObj[deferFailId] = function(data){
                                        self.log(deferDoneId);
                                        defer.resolve(data);
                                    };
                                    console.log("参数为＝＝》" + JSON.stringify(Array.prototype.slice.apply(arguments)))
                                    // 触发函数
                                    safari.self.tab.dispatchMessage(funId,{
                                        arg: Array.prototype.slice.apply(arguments),
                                        done: deferDoneId,
                                        fail: deferFailId
                                    });
                                    return defer;
                                };
                            })
                        }
                        _.each(obj,function(value,key){
                            if(_.isObject(value)){
                                addSafariTriggerHandle(value);
                            }
                        })
                    };
                    addSafariTriggerHandle(this.Airdroid);
                }
            }else{
                // firefox 事件触发器
                this.eventObj = fireFoxEvent;
                // 同时要绑定对应的事件触发器
                var addFireFoxTriggerHandle = function(obj){
                    if(obj.funListenerObjs){
                        _.each(obj.funListenerObjs,function(funId,name){
                            // 重新定义函数
                            // 这时候，要把组件的监听对象，传到page页面去
                            self.eventObj.emit('page_bind_handle',funId);
                            obj[name] = function(){
                                var defer = $.Deferred();
                                var deferDoneId = _.uniqueId("defer_done_" + name + "_");
                                var deferFailId = _.uniqueId("defer_fail_" + name + "_");
                                var deferAlwayId = _.uniqueId("defer_alway_" + name + "_");
                                // 一次性监听
                                self.eventObj.once(deferDoneId,function(data){
                                    //self.log(deferDoneId);
                                    defer.resolve(data.detail);
                                });
                                self.eventObj.once(deferFailId,function(data){
                                    //self.log(deferFailId);
                                    defer.reject(data.detail);
                                });
                                self.eventObj.once(deferAlwayId,function(data){
                                    //self.log(deferAlwayId);
                                    defer.reject(data.detail);
                                });

                                // 触发函数
                                self.eventObj.emit(funId,{
                                    arg: Array.prototype.slice.apply(arguments),
                                    done: deferDoneId,
                                    fail: deferFailId,
                                    always: deferAlwayId
                                });
                                return defer;
                            };
                        })
                    }
                    _.each(obj,function(value,key){
                        if(_.isObject(value)){
                            addFireFoxTriggerHandle(value);
                        }
                    })
                };
                addFireFoxTriggerHandle(this.Airdroid);
            }
            this.util= this.Airdroid.Util;
            this.server = this.Airdroid.Server;
            this.account = this.Airdroid.Account;
            this.EventType = this.Airdroid.Event.TYPE;
        },
        // 事件处理
        addListener: function(name, fun){
            var self = this;
            if(this.eventObj){
                if(window.chrome || window.safari){
                    this.eventObj.addEventListener(name, function (event) {
                        _.isFunction(fun) && fun(event.detail);
                    },false);
                }else{
                    // firefox
                    this.eventObj.on(name, function(event){
                        _.isFunction(fun) && fun(event.detail);
                    })
                }
            }
        },
        // 统计记录
        log: function(message){
            var self = this;
            var line;
            if (message instanceof Object || message instanceof Array) {
                line = message;
            } else {
                line = new Date().toLocaleString() + ' - ' + message;
            }

            console.log(line);
            self.rollingLog.push(JSON.stringify(line));

            if (self.rollingLog.length > 200) {
                self.rollingLog.shift();
            }
            // 输出来
//            var rollingDom = $("#rollingOutput");
//            if(rollingDom.length == 0){
//                $("body").append(rollingDom = $("<div id='rollingOutput'></div>"));
//            }
//            rollingDom.html(rollingDom.html() + "<br>" + message);
        },
        // 绑定事件
        delegateEvents: function (events) {
            var $parent, method, matches, eventName, selector;
            events = events || {};
            $parent = $('body');
            for (var key in events) {
                if (events.hasOwnProperty(key)) {
                    method = this[events[key]];
                    if (!method) throw new Error('Event "' + events[key] + '" does not exist');
                    matches = key.match(/^(\S+)\s*(.*)$/);
                    eventName = matches[1];
                    selector = matches[2];
                    method = _.bind(method, this);
                    if (selector === '') {
                        $parent.bind(eventName, method);
                    } else {
                        $parent.delegate(selector, eventName, method);
                    }
                }
            }
        }
    };
})();
```
其实`base`主要就做两件事:
1. 获取背景页的对象
2. 建立前端与背景页相互通信的事件驱动

### 1.获取背景页的对象。

获取各自浏览器的背景页对象(以下伪代码)  {% post_link browser-extension-7 %}
```javascript
var bg = getBrowserBackgroundPage();  // 这时候就可以得到Airdroid这个对象
setUpBackgroundPage(bg);  // 将获得的这个背景页的Airdroid对象赋给当前前端页面的Airdroid变量，还有一些其他的，比如server，account之类的
inited(args) // base初始化结束，轮到popup初始化
```
第一个步骤在之前的章节中有说过，这边就不提了。这边主要讲第二个步骤，即`setUpBackgroundPage`。
在这个步骤中，除了将背景页的`Airdroid对象`赋给`前端的Airdroid对象`之后。更要对事件驱动的对象进行设置。即 `eventObj`。

说到这个对象 `eventObj`，就必须得说道`base`要做的第二个事情，就是前端和背景页相互通信的事件驱动。

说是相互通信，但是为了更好的兼容各个浏览器，目前全部采用前端监听的方式, 由背景页来触发这个事件。  即 `addListener` 这个方法。比如监听用户登入，登出，消息过来之类的

```javascript
// 手机通知推送过来
self.addListener(self.EventType.notification_change, function (data) {
    ...
});

self.addListener(self.EventType.push_change, function (data) {
    ...
});

self.addListener(self.EventType.signed_in, function (data) {
    // 如果不是在前端登的，比如后台自动登录，那么要切换页面
});

self.addListener(self.EventType.signed_out, function (data) {
    // 如果在其他地方登录，就重新刷新
});
```
而 `addListener` 代码如下:
```javascript
addListener: function(name, fun){
    var self = this;
    if(this.eventObj){
        if(window.chrome || window.safari){
            this.eventObj.addEventListener(name, function (event) {
                _.isFunction(fun) && fun(event.detail);
            },false);
        }else{
            // firefox
            this.eventObj.on(name, function(event){
                _.isFunction(fun) && fun(event.detail);
            })
        }
    }
},
```
代码还是很好理解的， 就是对于`Chrome`和`Safari`，就采用 `addEventListener`的方式来监听。如果是`Firefox`的话，就采用`on`的方式来监听。

那么问题来了，`eventObj`这个对象是怎么来的，为什么会有两种不同的方法?? 其实这个对象`eventObj`就是在`setUpBackgroundPage`这个方法里面设置的,然后这边会根据不同的浏览器来处理：
1. `Chrome` 代码如下：
```javascript
this.eventObj = data;
```
代码最简单，直接将background的对象赋给`eventObj`就行了。

因为在前端页面用`background`的`window`对象进行`addEventListener`进行监听的话，如果是在背景页进行`window.dispatchEvent`进行触发的话，是可以生效的。

2. `Safari` 代码如下：
```javascript
this.eventObj = data;
if(dispatcher){
    // 说明是非popover页面
    // 同时要绑定对应的事件触发器
    var addSafariTriggerHandle = function(obj){
        if(obj.funListenerObjs){
            _.each(obj.funListenerObjs,function(funId,name){
                // 重新定义函数
                obj[name] = function(){
                    console.log("调用函数，函数名为%s, id为 %s", name, funId);
                    var defer = $.Deferred();
                    // 加入回调队列
                    var deferDoneId = _.uniqueId("defer_done_" + name + "_");
                    var deferFailId = _.uniqueId("defer_fail_" + name + "_");
                    self.funCbObj[deferDoneId] = function(data){
                        self.log(deferDoneId);
                        defer.resolve(data);
                    };
                    self.funCbObj[deferFailId] = function(data){
                        self.log(deferDoneId);
                        defer.resolve(data);
                    };
                    console.log("参数为＝＝》" + JSON.stringify(Array.prototype.slice.apply(arguments)))
                    // 触发函数
                    safari.self.tab.dispatchMessage(funId,{
                        arg: Array.prototype.slice.apply(arguments),
                        done: deferDoneId,
                        fail: deferFailId
                    });
                    return defer;
                };
            })
        }
        _.each(obj,function(value,key){
            if(_.isObject(value)){
                addSafariTriggerHandle(value);
            }
        })
    };
    addSafariTriggerHandle(this.Airdroid);
}
```
如果是`popup`页面的话，那就跟`Chrome`一样，直接 `eventObj=data`。 如果是其他的页面，即`dispatcher`是有值的。那么就要绑定对应的触发器,这个触发器是跟`background`的`events事件模型`相挂钩的，后面讲到事件模型的时候,会讲到。

3. Firefox 代码如下：
```javascript
// firefox 事件触发器
this.eventObj = fireFoxEvent;
// 同时要绑定对应的事件触发器
var addFireFoxTriggerHandle = function(obj){
    if(obj.funListenerObjs){
        _.each(obj.funListenerObjs,function(funId,name){
            // 重新定义函数
            // 这时候，要把组件的监听对象，传到page页面去
            self.eventObj.emit('page_bind_handle',funId);
            obj[name] = function(){
                var defer = $.Deferred();
                var deferDoneId = _.uniqueId("defer_done_" + name + "_");
                var deferFailId = _.uniqueId("defer_fail_" + name + "_");
                var deferAlwayId = _.uniqueId("defer_alway_" + name + "_");
                // 一次性监听
                self.eventObj.once(deferDoneId,function(data){
                    //self.log(deferDoneId);
                    defer.resolve(data.detail);
                });
                self.eventObj.once(deferFailId,function(data){
                    //self.log(deferFailId);
                    defer.reject(data.detail);
                });
                self.eventObj.once(deferAlwayId,function(data){
                    //self.log(deferAlwayId);
                    defer.reject(data.detail);
                });

                // 触发函数
                self.eventObj.emit(funId,{
                    arg: Array.prototype.slice.apply(arguments),
                    done: deferDoneId,
                    fail: deferFailId,
                    always: deferAlwayId
                });
                return defer;
            };
        })
    }
    _.each(obj,function(value,key){
        if(_.isObject(value)){
            addFireFoxTriggerHandle(value);
        }
    })
};
addFireFoxTriggerHandle(this.Airdroid);
```
`Firefox`的处理方式跟 `Safari` 不是 `popup` 页面的其他页面很像。也是跟`background`的`events模型`有关系,后面一起讲。

### 2.建立前端与背景页相互通信的事件驱动
具体查看： {% post_link browser-extension-10 %}




























