---
title: 浏览器 extension 插件开发系列(10) -- 事件驱动模型
date: 2020-02-11 17:29:30
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
这个插件开发最难的地方就是消息通信的事件驱动模型。而事件驱动模型，在这个项目可以分为3部分：
1. 前端页面调用背景页AirDroid对象的方法（比如调用登录方法）
2. 前端页面监听一个事件（监听登录成功事件），背景页触发这个事件
3. 背景页监听一个事件（监听登录成功事件），背景页和前端页面都触发这个事件

其中不同的浏览器有不同的实现方法，主要代码在前端的 `base.js` 和 背景页的 `event.js`, 其中 `base.js` 在{% post_link browser-extension-9 %} 已经分析了一部分，接下来我们结合以上三种情况来分析 `event.js`。
<!--more-->
## 1.前端页面调用背景页AirDroid对象的方法
这个在 {% post_link browser-extension-7 %} 已经讲了一些了，我们这一节做更详细的分析。
### 1.1 Chrome 浏览器
在`Chrome`中，可以直接通过`chrome.extension.getBackgroundPage()` 获取背景页的`window对象`。因此可以直接通过这个`window对象`来调用这个上下文环境的方法，比如调用登录方法:
```javascript
var bg = chrome.extension.getBackgroundPage();
bg.Airdroid.Account.signIn(mail, pwd);
```
### 1.2 Safari 浏览器
Safari 有两种情况:
1. 一种情况是`popup`页面，这个页面的处理方式跟`Chrome`的很像，就是可以直接获取后台的`window对象`。
```javascript
var bg = safari.extension.globalPage.contentWindow;
bg.Airdroid.Account.signIn(mail, pwd);
```
2. 另一种情况就是其他页面，比如`消息过来的回复页面（reply）`，这种不是`popup`页面的页面，是获取不到背景页的上下文环境。

只能通过监听，触发的方式来传数据。 即如果我要在`reply页面`调用一个`背景页server对象`的`回复接口sendIMMsg`,那么我必须要去触发一个`sendIMMsg_event` 事件，并监听一个叫`sendIMMsg_event_done`的数据回调事件，然后背景页这边本来就有监听这个`sendIMMsg_event`的事件，当背景页监听事件被触发之后，执行完逻辑之后（即执行`sendIMMsg`这个方法），就会反过来触发`sendIMMsg_event_done`这个事件，从而在前端页面得到这个事件的回调。

也就是如果我要触发背景页的一个对象的方法，那么在前端页面要监听一个事件，并要触发另一个事件:
前端页面代码:
```javascript
// 监听成功和失败事件
safari.self.addEventListener('message', function(e) {
   var eventName = e.name;
   if(eventName == 'sendIMMsg_event_done'){
       // sendIMMsg 这个方法在后台执行成功的回调
   } else if(eventName == 'sendIMMsg_event_fail'){
       // sendIMMsg 这个方法在后台执行失败的回调
   }                    
}, false);


// 触发这个事件
safari.self.tab.dispatchMessage('sendIMMsg_event',{
   arg: agruments
});
```
而背景页的代码如下:
```javascript
// 监听前端过来的事件
safari.application.addEventListener('message', function(e) {
     if(e.name == 'sendIMMsg_event'){
         var arg = e.message.arg;
         // 开始执行真正的sendIMMsg方法
         Airdroid.Server.sendIMMsg.apply(null, arg).done(function(data){
               // 调用成功, 触发sendIMMsg_event_done方法，把结果带过去
               e.target.page.dispatchMessage('sendIMMsg_event_done',data);
         }).fail(function(data){
              // 调用失败, 触发sendIMMsg_event_fail方法，把结果带过去
               e.target.page.dispatchMessage('sendIMMsg_event_fail',data);
         })
     }
})
```
大概就是这样，从而形成一个闭环。
1. 前端触发`sendIMMsg_event`
2. 背景页收到`sendIMMsg_event`事件，并执行真正的`sendIMMsg`方法, 并返回方法的done对象
3. 再触发到前端页面所监听的`done`和`fail`方法，并把结果带过去。
4. 前端页面触发这个 done 的方法，并且对背景页带过来的结果值进行处理

通过这个可以看到，其实前端页面只是想调用一个背景页的方法而已，但是过程却变得非常复杂。如果想调用十个，甚至几十个，那不是代码要写死了。而且通过这个例子可以看到，要调用的背景页的方法都要返回`defer`对象，这样才会有回调结果。

接下来我们优化这个流程，思路如下：

因为前端页面有可能调用Airdroid变量内的任何一个方法。 所以首先刚开始的时候，就要把背景页的Airdroid对象传过去。而且因为我们是通过 `e.target.page.dispatchMessage( event , data);` 来传递数据的。这个data不能太复杂，有两个局限性，一个是`function`方法带不过来，另一个是太复杂的对象也是传不过来，有了这两个限制，因此对象里面的`function`根本没法传过去。

为了解决这个问题，我们又重新`fake`了一个`Airdroid对象`，并把里面的`function`全部去掉，但是有新增了`funListenerObjs`对象。里面就是把要去掉的`function`的名字弄成一个唯一值。并放到这个`map`里面。同时把这个唯一值作为`key`，`function`的方法体作为`value`，放到一个全局的`window.funObjectObj`对象里面。以便后面如果前端页面要调用这个方法的时候，可以从这个`全局map`里面得到这个唯一值的方法体并执行。

部分代码如下: `event.js`:
```javascript
// 存放方法的唯一值和对应的方法提
window.funObjectObj = [];
// 过滤出函数
self.addHandleTriggerToFrontInSafari(Airdroid);
// 接下来把函数去掉，并传入一个复制体
// 首先把Airdroid克隆一份
var fakeAirdroid = $.extend(true,{}, Airdroid);
var removeFunction = function(obj){
    _.each(obj,function(value,key){
        if(_.isFunction(value)){
            delete obj[key];
        }else if(_.isObject(value)){
            removeFunction(value);
        }
    })
};
removeFunction(fakeAirdroid);
console.log("airdroid==>" + JSON.stringify(fakeAirdroid));
// 这边要监听来自非popup页面的数据传递，比如 reply，option 这种单独的页面
safari.application.addEventListener('message', function(e) {
    console.log("函数调用过来了，名字为：" + e.name);
    // 针对这些页面的数据初始化，这些页面不像popup这种页面可以直接访问后台的全局数据，而是必须要将数据进行传递
    if (e.name == 'page_init') {
        // todo 这边不能直接把Airdroid这个变量传递过去，数据量会太大，
        // todo 因此只能像firefox那样，如果是函数就用字符串来替换, 而且这边还要转化成字符串，不然会出现对象复制不了的情况
        e.target.page.dispatchMessage('page_data', JSON.stringify({
            "Airdroid": fakeAirdroid
        }));
    }else{
        // 接下来就是执行那些函数了
        if(window.funObjectObj[e.name]){
            console.log("开始执行传过来的函数");
            var result = e.message;
            // 真正的参数,而且是argument的形式
            var arg = result.arg;
            // 如果有回调，就触发调用 这边已经绑定了上下文环境了，所以这边直接为null即可
            var funResult = window.funObjectObj[e.name].apply(null,arg);
            // 判断是否是Deferred对象
            if(_.isFunction(funResult.done)){
                funResult.done(function(data){
                    result.done && e.target.page.dispatchMessage(result.done,data);
                }).fail(function(data){
                    result.fail && e.target.page.dispatchMessage(result.fail,data);
                }).always(function(data){
                    result.always && e.target.page.dispatchMessage(result.always,data);
                })
            }else{
                // 返回return的
                result.done && e.target.page.dispatchMessage(result.done,funResult);
            }
        }
    }
}, false);
```
然后过滤的这个方法如下： `addHandleTriggerToFrontInSafari`:
```javascript
// 跟非popover页面通信，比如reply页面的时候，兼容chrome和firefox，让其可以像popover那样，可以直接调用后台的函数(这里的后台函数必须要defer形式)
// todo 这边禁止一种用法，就是数组里面的项不能是函数，不然会有问题
addHandleTriggerToFrontInSafari: function(obj){
    var self = this;
    if(Airdroid.Util.Browser.safari){
        if(_.isObject(obj)){
            _.each(obj,function(value,key){
                // 如果是函数，就保留，并监听，设置一个唯一值
                if(_.isFunction(value)){
                    obj.funListenerObjs = obj.funListenerObjs || {};
                    var uniqFunId = _.uniqueId("fun_");
                    // 加入监听队列，并监听
                    obj.funListenerObjs[key] = uniqFunId;
                    // 加入window对象，并绑定执行上下文
                    window.funObjectObj[uniqFunId] = value.bind(obj);
                }else if(_.isObject(value)){
                    self.addHandleTriggerToFrontInSafari(value);
                }
            })
        }
    }
},
```
通过这个方式，就可以看到，如果本来 `Airdroid对象` 长这样：
```javascript
Airdroid = {
     name: 'xxx',
     server: {
         baseUrl: 'http://xxx.airdroid.com',
         sendMsg: function(){
            // 发送消息
         }
     },
     account: {
         username: 'kkk',
         signIn: function(){
           // 登录操作 
         }
    }
}
```
那么经过上述 `addHandleTriggerToFrontInSafari` 和 `fake` 和 `removeFunction` 操作，就会变成:
```javascript
fakeAirdroid = {
     name: 'xxx',
     server: {
         baseUrl: 'http://server.airdroid.com',
         funListenerObjs: {
             sendMsg: 'fun_1'
         }
     },
     account: {
         username: 'kkk',
          funListenerObjs: {
            signIn: 'fun_2'
         }
    }
}
```
然后全局的`funObjectObj`为:
```javascript
window.funObjectObj = {
     'fun_1': function(){
            // 发送消息
     },
      'fun_2': function(){
            // // 登录操作
     }
}
```
`fakeAirdroid` 里面已经没有函数了，每一个有函数的对象底下都有一个叫`funListenerObjs`的对象，里面就是`{key,value} => {对应的函数名，对应函数名的唯一值}`, 而全局对象`window.funObjectObj`也会有这个结构 `{key, value} => {某一个函数的唯一值,  这个唯一值所对应函数的函数体}`。

那么这样做有什么好处呢？

第一个是，有了`fakeAirdroid对象`，那么就可以通过 `e.target.page.dispatchMessage( event , data);`  这种方式把`Airdroid对象`传到前端页面:
```javascript
e.target.page.dispatchMessage('page_data', JSON.stringify({
      "Airdroid": fakeAirdroid
}));
```
这样子前端页面也有了`Airdroid对象`了。那么是不是也可以跟`Chrome`一样，直接在通过这个`Airdroid对象`调用背景页的方法呢？ 显然是不行的。因为这个`Airdroid对象`其实跟`背景页的Airdroid对象`不是同一个，是一个`fake` 的对象，而且还差很多，因为它没有任何的`function`方法。

那么要怎么做到让这个`Airdroid对象`跟在`Chrome`浏览器一样，可以直接用 `Airdroid.acccount.signIn` 直接调用背景页的登录方法呢？ 这个秘诀其实就是在前端页面的js，`base.js` 中的`setUpBackground`方法中处理的:
```javascript
init: function () {
    var self = this, args = arguments;
    $(document).ready(function () {
        // 这时候要根据浏览器来获取背景对象
        if (window.chrome) {
            // chrome
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
            // firefox
        }
    });
},

// 设置背景对象
setUpBackgroundPage: function(data, dispatcher){
    var self = this;
    this.Airdroid = data.Airdroid;
    window.Airdroid = this.Airdroid;
    // 事件对象
    if(window.chrome){
        // chrome
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
        // firefox
    }
    this.util= this.Airdroid.Util;
    this.server = this.Airdroid.Server;
    this.account = this.Airdroid.Account;
    this.EventType = this.Airdroid.Event.TYPE;
},
```
从上述代码可以看到，在前端页面的`document.ready` 事件中。会去触发 `safari.self.tab.dispatchMessage('page_init');` 这个事件，让背景页把`Airdroid对象`传过来, 这时候背景页的监听就会把`fakeAirdroid`传到前端，然后前端再调用`setUpBackgroundPage`进行设置.

判断一个页面是否是`Safari`的`非popup页面`，就看有没有 `safari.extension.globalPage` 这个对象。 所以背景页 `event.js` 的代码如下:
```javascript
safari.application.addEventListener('message', function(e) {
    console.log("函数调用过来了，名字为：" + e.name);
    // 针对这些页面的数据初始化，这些页面不像popup这种页面可以直接访问后台的全局数据，而是必须要将数据进行传递
    if (e.name == 'page_init') {
        // todo 这边不能直接把Airdroid这个变量传递过去，数据量会太大，
        // todo 因此只能像firefox那样，如果是函数就用字符串来替换, 而且这边还要转化成字符串，不然会出现对象复制不了的情况
        e.target.page.dispatchMessage('page_data', JSON.stringify({
            "Airdroid": fakeAirdroid
        }));
    }else{
       //...
    }
})
```
而前端页面 `base.js` 对应的代码片段为:
```javascript
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
        //...
    }
}, false);
```
所以这时`setUpBackgroundPage` 这个方法的`data`参数，其实就是`fakeAirdroid对象`，它里面是没有函数的，只有一个`funListenerObjs`对象。
所以我们要在这个对象做文章，因为这个对象的每一条记录都是对应这个对象所在对象的一个函数。 那么就很简单了，我们只要把他还原成函数就行了。 所以对应的代码片段就是:
```javascript
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
```
上述代码说明我们会去遍历`fakeAirdroid对象`下的每一个`funListenerObjs`对象，然后把它还原成函数，但是请注意，这个函数名虽然跟背景页的那个函数名一样，
但是函数体可是不一样的，我们在函数体里面定义了两个`defer`回调，一个是`done`，一个是`fail`，并且把他们对应的`id`保存在一个变量`funCbObj`中。

最后这个函数返回也是一个`defer`对象。 所以经过这一步之后，`fakeAirdroid`会变成什么情况呢？
```javascript
fakeAirdroid = {
     name: 'xxx',
     server: {
         baseUrl: 'http://xxx.airdroid.com',
         funListenerObjs: {
             sendMsg: 'fun_1'
         },
         sendMsg: function(){
             console.log("调用函数，函数名为%s, id为 %s", name, funId);
             var defer = $.Deferred();
              // 加入回调队列
             var deferDoneId = _.uniqueId("defer_done_sendMsg_");
             var deferFailId = _.uniqueId("defer_fail_sendMsg_");
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
             safari.self.tab.dispatchMessage('fun_1',{
                   arg: Array.prototype.slice.apply(arguments),
                   done: deferDoneId,
                   fail: deferFailId
              });
              return defer;

         }
     },
     account: {
         username: 'kkk',
          funListenerObjs: {
            signIn: 'fun_2'
         },
        signIn: function(){
             console.log("调用函数，函数名为%s, id为 %s", name, funId);
             var defer = $.Deferred();
              // 加入回调队列
             var deferDoneId = _.uniqueId("defer_done_signIn_");
             var deferFailId = _.uniqueId("defer_fail_signIn_");
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
             safari.self.tab.dispatchMessage('fun_2',{
                   arg: Array.prototype.slice.apply(arguments),
                   done: deferDoneId,
                   fail: deferFailId
              });
              return defer;

         }
    }
}
```
发现多了两个方法，一个是`server`下的`sendMsg`方法，一个是`account`的`signIn`方法。现在我们跟在`Chrome`一样调用登录方法 
```javascript
Airdroid.account.signIn().done(function(){
// 登录成功
}).fail(function(){
// 登录失败
})
```
方法开始执行，首先就到方法体内，然后在变量`funCbObj`里面加入两个方法，一个是`done`方法，一个是`fail`方法，同时这两个方法的key值都是唯一值（每一次方法触发都会生成两个唯一值）。然后触发消息 `func_2`, 并把`arg`参数，`done`对应的方法`id`，`fail`对应的方法`id`带过去。
```javascript
safari.self.tab.dispatchMessage('fun_2',{
   arg: Array.prototype.slice.apply(arguments),
   done: deferDoneId,
   fail: deferFailId
});
```
接下来就到了背景页的逻辑处理了:
```javascript
safari.application.addEventListener('message', function(e) {
  console.log("函数调用过来了，名字为：" + e.name);
  // 针对这些页面的数据初始化，这些页面不像popup这种页面可以直接访问后台的全局数据，而是必须要将数据进行传递
  if (e.name == 'page_init') {
      // init
  }else{
      // 接下来就是执行那些函数了
      if(window.funObjectObj[e.name]){
          console.log("开始执行传过来的函数");
          var result = e.message;
          // 真正的参数,而且是argument的形式
          var arg = result.arg;
          // 如果有回调，就触发调用 这边已经绑定了上下文环境了，所以这边直接为null即可
          var funResult = window.funObjectObj[e.name].apply(null,arg);
          // 判断是否是Deferred对象
          if(_.isFunction(funResult.done)){
              funResult.done(function(data){
                  result.done && e.target.page.dispatchMessage(result.done,data);
              }).fail(function(data){
                  result.fail && e.target.page.dispatchMessage(result.fail,data);
              }).always(function(data){
                  result.always && e.target.page.dispatchMessage(result.always,data);
              })
          }else{
              // 返回return的
              result.done && e.target.page.dispatchMessage(result.done,funResult);
          }
      }
  }
}, false);
```
可以看到，在背景页的消息处理中，如果这个消息名字不是 `page_init`, 那么统一从`全局对象funObjectObj`来取。而本例子，`e.name` 就是 `fun_2`, `funObjectObj` 的值为:
```javascript
window.funObjectObj = {
 'fun_1': function(){
        // 发送消息
 },
  'fun_2': function(){
        // // 登录操作
 }
}
```
刚好有这个方法，所以就执行函数体（经过上述一连串的处理，最后这个函数体就是`signIn`方法的函数），最后判断是否返回`defer`对象。如果有的话，就触发  `e.target.page.dispatchMessage(result.done,data);` 和 `e.target.page.dispatchMessage(result.fail,data);` 然后把返回值带过去，因为`doneId` 和 `failId` 都传过来了，所以可以正常返回到前端页面。

这时候前端页面就触发了:
```javascript
safari.self.addEventListener('message', function(e) {
    var eventName = e.name;
    console.log("消息过来了，名字为：" + eventName);
    if (eventName == 'page_data') {
        //
    }else{
        if(self.funCbObj[eventName]){
            self.funCbObj[eventName](e.message);
            delete self.funCbObj[eventName];
        }
    }
}, false);
```
因为`eventName`就是上述`doneId` 或者 `failId`，所以直接在`funCbOjb`进行触发，并把参数传过去。因此每一次触发这个方法，`doneId`和`failId`都是一次性的唯一值，所以可以在`funCbObj`执行完函数之后，直接删除掉。所以这时候就触发 `doneId` 或者 `failId` 函数，然后返回`defer`到:
```javascript
Airdroid.account.signIn().done(function(data){
// 登录成功 doneId defer 返回的data就会到这里
}).fail(function(){
// 登录失败 failId defer 返回的data就会到这里
})
```
这样子整个流程就完成了。这个结果跟在`Chrome`浏览器调用的接口是一样的。以上就是`Safari`下，非`popup`页面调用背景页`Airdroid对象`方法的情况。

### 1.3 Firefox 浏览器
`Firefox`的情况跟 `Safari`的 非`popup`页面非常像。也是不能直接获取背景页的上下文环境。而是都是通过监听和触发的方式来执行的。不过跟`Safari`不一样的是，`Firefox`的监听和触发不是 `safari.self.addEventListener('message', function(e) {})` 和 `safari.self.tab.dispatchMessage('page_init',{});` 而是 `self.port.on('page_data', function(data) {})` 监听 和 `self.port.emit('page_init',{}) `触发。如果是一次性的监听就是 `self.port.once('page_data', function(data) {})`

跟`Safari` 非`popup`页面一样，还是先从背景页获取`fakeAirdroid`。先触发 `page_init` 事件，然后监听`page_data` 事件获取`fakeAirdroid对象`。所以部分 `base.js` 代码如下:
```javascript
//todo 这边为了兼容Firefox extension 下的前后台调用，所有在前台调用后台的函数，都必须是用defer方式来调用的
var fireFoxEvent =  (self && self.port) || {};


init: function () {
    var self = this, args = arguments;
    $(document).ready(function () {
        // 这时候要根据浏览器来获取背景对象
        if (window.chrome) {
            // chrome
        } else if (window.safari) {
            // safari
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
```
在背景页的代码片段就是 `events.js`:
```javascript
var fireFoxEvent = (self && self.port) || {};

// 初始化
init: function(){
    var self = this;
    if (window.chrome) {

    } else if (window.safari) {
        // safari
    } else {
        // 这时候要注册可以双向通行的事件
        // todo 这边如果已经在 background.js 中已经注册的事件了，就不要重复绑定了,不然前台会触发两次
        var isListener = [self.TYPE.signed_in, self.TYPE.signed_out];
        _.each(self.TYPE,function(value,key){
            self.dispatchEvent(self.FireFoxEvent.registerHandle, value);
            // 同时绑定所有监听的默认事件
            (isListener.indexOf(value) == -1) && Airdroid.Event.addEventListener(value, $.noop);
        });
        // 接下来开始添加AirDroid的function，让function转化为字符串，让前台去转化监听
        self.addHandleTriggerToFrontInFirefox(Airdroid);
        self.addEventListener(self.FireFoxEvent.page_init, function() {
            self.dispatchEvent(self.FireFoxEvent.page_data, {
                "Airdroid": Airdroid
            });
        });
    }
},

addEventListener: function(eventName, listener, isOnce) {
  var self = this;
  if(window.chrome || window.safari){
      this.eventListeners.push({ 'eventName': eventName, 'listener': listener });
      window.addEventListener(eventName, listener, false);
  }else {
      // firefox
      // 这边还要把这个事件回调到前台去
      var cb = function(data){
          listener(data);
          // 还要回调到前台去，这边注意一个问题，如果有很多个前台的话，不知道会不会发生事件覆盖
          (_.values(self.TYPE).indexOf(eventName) > -1) && fireFoxEvent.emit(eventName + "_front", data);
      };
      if(isOnce){
          fireFoxEvent.once(eventName,cb);
      }else{
          fireFoxEvent.on(eventName,cb);
      }
  }
},
// 触发事件
dispatchEvent: function(eventName, details) {
    if (window.chrome || window.safari) {
        window.dispatchEvent(new CustomEvent(eventName, { 'detail': details }));
    } else {
        // firefox
        // todo 因为Firefox的特殊性，他不能再同一个js里面，同时绑定事件，并触发，而是都要经过main.js绕一圈
        // 有一种方法就是使用addEventListener，但是这个又不能传到前台去
        // 这时候要判断该事件是哪一种事件，是双向的，还是单向的， 即属于FireFoxEvent里面的，还是TYPE 里面的
        if(_.values(this.TYPE).indexOf(eventName) > -1){
            // 这时候要到外面绕一圈回来
            fireFoxEvent.emit(eventName + "_firefox", { 'detail': details });
        }else{
            fireFoxEvent.emit(eventName, { 'detail': details });
        }
    }
},
// firefox 下监听页面初始化事件, 然后传数据过去
// 注意， Firefox 这边很变态，他这边不是传的整个 AirDroid，
// 而是 AirDroid 里面的 可以json格式化的字符串
// (也就是说，要注意两点，一点是不是对象传递，另一点就是function是传不过去的)
// https://developer.mozilla.org/en-US/Add-ons/SDK/Guides/Content_Scripts/using_port
// 接下来给所有对象的方法，统一加上 on和emit事件
// todo 注意，这种方法，对new 的对象是不行的，比如 File Device，在前台如果这样做会出错, 如果在初始化的时候，也要执行这个操作（如果这个对象需要在前台页面操作，比如 File，Device）
// 在特定浏览器中添加(目前就Firefox中用到)
// todo 这边禁止一种用法，就是数组里面的项不能是函数，不然会有问题
addHandleTriggerToFrontInFirefox: function(obj){
    var self = this;
    if(Airdroid.Util.Browser.firefox){
        if(_.isObject(obj)){
            _.each(obj,function(value,key){
                // 如果是函数，就保留，并监听，设置一个唯一值
                if(_.isFunction(value)){
                    obj.funListenerObjs = obj.funListenerObjs || {};
                    var uniqFunId = _.uniqueId("fun_");
                    // 加入监听队列，并监听
                    obj.funListenerObjs[key] = uniqFunId;
                    self.addEventListener(uniqFunId,function(event){
                        // 这边接收参数
                        // 传过来的实际参数
                        var result = event;
                        // 真正的参数,而且是argument的形式
                        var arg = result.arg;
                        // 如果有回调，就触发调用
                        var funResult = value.apply(obj,arg);
                        // 判断是否是Deferred对象
                        if(_.isFunction(funResult.done)){
                            funResult.done(function(data){
                                result.done && self.dispatchEvent(result.done,data);
                            }).fail(function(data){
                                        result.fail && self.dispatchEvent(result.fail,data);
                                    }).always(function(data){
                                        result.always && self.dispatchEvent(result.always,data);
                                    })
                        }else{
                            // 返回return的
                            result.done && self.dispatchEvent(result.done,funResult);
                        }
                    });
                }else if(_.isObject(value)){
                    self.addHandleTriggerToFrontInFirefox(value);
                }
            })
        }
    }
}
```
从上面的代码来看，`Firefox`在这一块的处理上还是跟`Safari`的非`popup`页面的处理上，还是相差很多的。 先 `init` 方法:
```javascript
var isListener = [self.TYPE.signed_in, self.TYPE.signed_out];
_.each(self.TYPE,function(value,key){
    self.dispatchEvent(self.FireFoxEvent.registerHandle, value);
    // 同时绑定所有监听的默认事件
    (isListener.indexOf(value) == -1) && Airdroid.Event.addEventListener(value, $.noop);
});
```
这部分的逻辑是`Firefox`注册双向通道的逻辑。这个后面会讲到。这边就看下面这个逻辑就行了，即
```javascript
// 接下来开始添加AirDroid的function，让function转化为字符串，让前台去转化监听
self.addHandleTriggerToFrontInFirefox(Airdroid);
self.addEventListener(self.FireFoxEvent.page_init, function() {
    self.dispatchEvent(self.FireFoxEvent.page_data, {
        "Airdroid": Airdroid
    });
});
```
可以看到会把`Airdroid这个变量`，通过`addHandleTriggerToFrontInFirefox`这个函数再转化一次。这个逻辑跟`Safari` 非`popup`页面的处理逻辑有点像，也是转化`Airdroid`里面的函数，这边有几个点要注意:
1. 他这边不是传的整个 `AirDroid`，而是 `AirDroid` 里面的可以json格式化的字符串(也就是说，要注意两点，一点是不是对象传递，另一点就是`function`是传不过去的), 具体可以看: [using port](https://developer.mozilla.org/en-US/Add-ons/SDK/Guides/Content_Scripts/using_port)
2. 给所有对象的方法，统一加上 `on`和`emit`事件。注意，这种方法，对`new` 的对象是不行的，比如 `File Device`，在前台如果这样做会出错, 如果在初始化的时候，也要执行这个操作（如果这个对象需要在前台页面操作，比如 `File，Device`）
3. 这边禁止一种用法，就是数组里面的项不能是函数，不然会有问题
4. `Firefox`是在转化函数的时候，就对函数的`uniqFunId`进行监听。而不是像`Safari`非`popup`页面那样要先放一个全局的`funObjectObj`对象，然后在统一的`safari.application.addEventListener('message', function(e) {})` 进行处理。

接下来就是对`page_init` 事件进行监听并回应，把`Airdroid`对象传过去:
```javascript
self.addEventListener(self.FireFoxEvent.page_init, function() {
    self.dispatchEvent(self.FireFoxEvent.page_data, {
        "Airdroid": Airdroid
    });
});
```
这个就涉及到`Firefox`一个最恶心的地方。就是`背景页的port`和`前端页面的port`不是同一个东西。 什么意思呢， 也就是说如果在前端页面触发一个事件 `self.port.emit("init');`, 然后背景页监听这个事件 `self.port.on("init");`, 其实你会发现，其实并没有卵用，反过来也是一样，也是没用的。其原因就是 这两个`self`变量不是同一个东西，这两个`port`也不是同一个东西。 也就说，一个是前端页面的上下文环境，一个是背景页的上下文环境。所以搭不上关系。

那么这个项目是怎么让他们搭得上关系的呢？其中奥秘就是在 `main.js` 中。在`Firefox`中，无论是背景页，还是`popup`页面都是在`main.js`中初始化的。 我们先看一下 `lib/main.js` 中的代码:
```javascript
var self = require('sdk/self');
var ss = require('sdk/simple-storage');
var windows = require('sdk/windows');
var tabs = require('sdk/tabs');
var Panel = require('sdk/panel').Panel;
var contextMenu = require('sdk/context-menu');
var prefs = require('sdk/simple-prefs').prefs;
var Request = require('sdk/request').Request;
var clipboard = require("sdk/clipboard");
var timers = require('sdk/timers');
var system = require("sdk/system");

var compatToolbarButton = require('./toolbar-button');

var {Cc, Ci} = require('chrome');
var ioSvc = Cc['@mozilla.org/network/io-service;1'].getService(Ci.nsIIOService);
var cookieSvc = Cc['@mozilla.org/cookieService;1'].getService(Ci.nsICookieService);
var cookieMgr = Cc['@mozilla.org/cookiemanager;1'].getService(Ci.nsICookieManager);
var mediator = Cc['@mozilla.org/appshell/window-mediator;1'].getService(Ci.nsIWindowMediator);

var trim = function(str) {  
    return str.replace(/^\s+|\s+$/g, '');
};

var firefoxVersion = system.version.match(/(\d+)/)[0];
var widget;
// 显示菜单栏图标
var setUpToolbarButton = function() {
    var icon = {
        '16': self.data.url('images/icon_16.png'),
        '32': self.data.url('images/icon_32.png')
    };

    if (firefoxVersion >= 30) {
        if (!widget) {
            widget = require('sdk/ui/button/toggle').ToggleButton({
                id: 'airdroid-widget',
                label: 'AirDroid',
                icon: icon,
                onChange: function(state) {
                    if (state.checked) {
                        var panel = onToolbarButtonClicked(function() {
                            widget.state('window', { checked: false });
                            panel.destroy();
                        });
                        panel.show({
                            position: widget
                        });
                    }
                }
            });
        } else {
            widget.icon = icon;
        }
    } else 
    if (firefoxVersion == 29) {
        if (!widget) {
            widget = require('sdk/widget').Widget({
                id: 'airdroid-widget',
                label: 'AirDroid',
                contentURL: icon['16'],
                onClick: function(view) {
                    view.panel = onToolbarButtonClicked();
                }
            });
        } else {
            widget.contentURL = icon['16'];
        }
    } else {
        // Set up the toolbar buttons on existing windows
        var enumerator = mediator.getEnumerator('navigator:browser');
        while (enumerator.hasMoreElements()) {
            compatToolbarButton.add(self, enumerator.getNext(), onToolbarButtonClicked, icon['16']);
        }

        // Set up the toolbar button on any new windows that get opened
        windows.browserWindows.on('open', function(window) {
            compatToolbarButton.add(self, mediator.getMostRecentWindow('navigator:browser'), onToolbarButtonClicked, icon['16']);
        });
    }
};

var onToolbarButtonClicked = function(onHide) {
    var options = {
        //'width': 355,
        //'height': 315,
        'contentURL': self.data.url('popup.html'),
        'contentScriptFile': [
            self.data.url('js/lib/jquery/jquery.js'),
            self.data.url('js/lib/jquery/jquery.toast.js'),
            self.data.url('js/lib/jquery/jquery.clipboard.js'),
            self.data.url('js/lib/underscore/underscore.js'),
            self.data.url('js/util/tplHelper.js'),
            self.data.url('js/model/device.js'),
            self.data.url('js/web/base.js'),
            self.data.url('js/web/popup.js'),
            self.data.url('js/lib/bootstrap/js/bootstrap.js')
        ]
    };

    if (onHide) {
        options.onHide = onHide;
    }

    var panel = Panel(options);

    attachListeners(panel);

    if (firefoxVersion >= 29) {
        return panel;
    } else {
        panel.show(mediator.getMostRecentWindow('navigator:browser').document.getElementById('airdroid'));
    }
};

var attachListeners = function(component) {
    component.port.on('resize', function(data){
        data.width && (component.width = data.width);
        data.height && (component.height = data.height);
    });

    component.port.once('page_init', function() {
        // 这边的page对应的是后台的 self.port对象
        // component 对应的是前台的 self.port对象
        // 因此会有这么一个流程
        // 先触发 component的page_init(base.js里面) 再触发page里面的page_init 再触发page的page_data,最后再触发component的page_data
        page.port.once('page_data', function(data) {
            component.port.emit('page_data', data);
        });
        page.port.emit('page_init');
    });
    // 绑定那些可以从后台触发到前台的事件，比如signed_in
    component.port.on("page_register_handle",function(data){
        var name = data.name;
        // 绑定对应页面的前台事件, 然后再对应到前台所绑定的函数
        page.port.on(name + "_front", function(data){
            component.port.emit(name,data);
        })
    });

    // 初始化方法
    component.port.on('page_bind_handle', function(result) {
        var funId = result;
        component.port.on(funId,function(data){
            var deferDoneId = data.done;
            var deferFailId = data.fail;
            var deferAlwayId = data.always;
            page.port.once(deferDoneId,function(data){
                component.port.emit(deferDoneId,data);
            });
            page.port.once(deferFailId,function(data){
                component.port.emit(deferFailId,data);
            });
            page.port.once(deferAlwayId,function(data){
                component.port.emit(deferAlwayId,data);
            });
            // 触发函数
            page.port.emit(funId,data);
        });
    });

    component.port.on('open_tab', function(tabInfo) {
        openTab(tabInfo);
        if (component.hide) {
            component.hide();
        }
    });
};

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
// 注册可以双向通信的事件通道
page.port.on('registerHandle', function(event) {
    var handleName = event.detail;
    // 这边开始注册一个事件, 名字为 name 后面加上 firefox 字样, 主要是让后台来触发，再回传到后台去，相当于绕了一圈，这样子会比较麻烦，但是为了兼容其他浏览器和一致性，只能这样了
    page.port.on(handleName + "_firefox", function(data){
        // 同时回传过去
        page.port.emit(handleName, data);
        // 同时也回传到前台. 注意，这边不能直接在同一个main.js 里面触发，还是要先绕回去
        // page.port.emit(handleName + "_front", data);
    });
});

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

var openTab = function(tabInfo) {
    var url = tabInfo.url;
    if (url.indexOf('http') == -1) {
        url = self.data.url(url);
    }

    tabs.open({
        'url': url,
        'onClose': function() {
            page.port.emit('tab_closed', tabInfo.id);
        }
    });
};

page.port.on('open_tab', function(tabInfo) {
    openTab(tabInfo);
});

page.port.on("get_active_tab", function(data){
    data = data.detail;
    page.port.emit(data.uid, {detail:{ 'title': tabs.activeTab.title, 'url': tabs.activeTab.url }});
});

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

//page.port.on("get_all_cookie", function(data){
//    data = data.detail;
//    var cookieString = cookieSvc.getCookieString(ioSvc.newURI(data.domain, null, null), null);
//    page.port.emit(data.uid, {detail: cookieString});
//});
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

setUpToolbarButton();
console.log("Hello World");
```
从上面代码可以看到。 在 `attachListeners` 方法中， `component`指的就是`popup`页面的上下文环境对象（即`self`），而`page`就是背景页的上下文环境对象:
```javascript
component.port.once('page_init', function() {
    // 这边的page对应的是后台的 self.port对象
    // component 对应的是前台的 self.port对象
    // 因此会有这么一个流程
    // 先触发 component的page_init(base.js里面) 再触发page里面的page_init 再触发page的page_data,最后再触发component的page_data
    page.port.once('page_data', function(data) {
        component.port.emit('page_data', data);
    });
    page.port.emit('page_init');
});
```
从这个代码可以看出。`component` 绑定了一个一次性事件 `page_init`. 当这个事件被触发之后，就在背景页那边绑定了一个一次性事件 `page_data`, 而这个监听的事件就是触发`component`的`page_data`事件。当绑定背景页的`page_data` 事件之后，接着就触发背景页的 `page_init` 事件。

看起来很复杂。 但主要遵守一点，就很容易理解。就是前端页面和后端页面不能直接通信，必须要通过`main.js`的这个中转站，以这个事件为例,我们要的结果就是： 
1. 前端页面监听`page_data`事件，并触发`page_init` 到背景页 (`base.js`)
2. 背景页收到前端触发的 `page_init` 事件，并把`Airdroid对象`通过 `page_data`事件发送到前端页面 (`event.js`)
3. 前端收到背景页触发的`page_data`事件，并得到他传过来的`Airdroid对象`。

但是真正的流程是这样的:
1. 前端页面监听`page_data`事件，并触发`page_init` 事件 (`base.js`)
```javascript
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
```
2. 中转页面(`main.js`) 收到这个前端页面的`page_init` 事件，并绑定背景页的 `page_data`事件，然后触发背景页的 `page_init` 事件
```javascript
component.port.once('page_init', function() {
    // 这边的page对应的是后台的 self.port对象
    // component 对应的是前台的 self.port对象
    // 因此会有这么一个流程
    // 先触发 component的page_init(base.js里面) 再触发page里面的page_init 再触发page的page_data,最后再触发component的page_data
    page.port.once('page_data', function(data) {
        component.port.emit('page_data', data);
    });
    page.port.emit('page_init');
});
```
3. 背景页收到中转页面（`main.js`）的触发的`page_init` 事件，并触发背景页的`page_data`事件，然后`Airdroid对象`传过去。
```javascript
self.addEventListener(self.FireFoxEvent.page_init, function() {
    self.dispatchEvent(self.FireFoxEvent.page_data, {
        "Airdroid": Airdroid
    });
});
```
4. 中转页面(`main.js`) 收到背景页的`page_data`事件（这个事件是在第二个步骤绑定的），然后触发前端页面的`page_data`事件，并把`data`页面传过去
```javascript
page.port.once('page_data', function(data) {
    component.port.emit('page_data', data);
});
```
5. 前端页收到中转页面（`main.js`）的触发的`page_data` 事件，并得到它传过来的`data`数据（其实就是从背景页过来的`Airdroid对象`）
```javascript
fireFoxEvent.on('page_data', function(data) {
    // 传过来 AirDroid对象
    self.setUpBackgroundPage(data.detail);
    // 接下来绑定可以传递到前台的触发事件
    _.each(_.values(self.EventType),function(item){
        fireFoxEvent.emit("page_register_handle",{name:item});
    });
    self.inited(args);
});
```
对比上面的流程，就会发现多了两个中转流程。而在`Firefox`下，前端页面和背景页的通信就是得这样。

终于费了好大的劲，把`Airdroid对象`从背景页传到前端页面来了。虽然这个`Airdroid对象`也是一个`fakeAirdroid对象`，因为没有函数和复杂对象（通过`addHandleTriggerToFrontInFirefox`方法），跟`Safari`的非`popup`页面情况差不多。那么接下来就是怎么在前端页面调用背景页的方法了。

方法原理跟`Safari`的非`popup`页面的方式差不多，也是把`Airdroid对象`下的`funListenerObjs`的方法`map`，重定义成函数。 `base.js`的代码片段如下:
```javascript
// 设置背景对象
setUpBackgroundPage: function(data, dispatcher){
    var self = this;
    this.Airdroid = data.Airdroid;
    window.Airdroid = this.Airdroid;
    // 事件对象
    if(window.chrome){
        // chrome
    }else if(window.safari){
        // safari
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
```
这边的`eventObj`事件对象就是 `self.port`. 你会发现大部分的逻辑都跟`Safari`的非`popup`页面差不多。都是重定义函数，并在函数体内，对`defer`返回值, 即 `done`，`fail`，`always`，进行了事件的一次性监听，最后再触发这个`funcId`。但是还是有几个细节不太一样：
1. `Firefox` 是直接在函数体里面进行`defer`返回值对象的绑定监听，不需要像`Safari`的非`popup`页面那样，还要借助一个`funCbObj`对象，然后统一在`addListener`那边进行处理。
2. 而且这边还需要把这个函数的监听，转到中间页面(`main.js`), 即 `self.eventObj.emit('page_bind_handle',funId);`

通过之前的分析，你会发现如果`Firefox`只是像`Safari`非`popup`页面那样处理的话，无论是函数的`funId`，还是重定义函数体内的`defer`返回值的绑定监听。其实都无法生效。因为根本到达不了背景页。背景页也没法把`defer`返回值的`函数id`触发到前端页面。

也就是说 在 `base.js` 页面这样写:
```javascript
self.eventObj.emit(funId,{
  arg: Array.prototype.slice.apply(arguments),
  done: deferDoneId,
  fail: deferFailId,
  always: deferAlwayId
});
```
这个根本到达不了背景页。而在背景页 `events.js` 这样写:
```javascript
var funResult = value.apply(obj,arg);
// 判断是否是Deferred对象
if(_.isFunction(funResult.done)){
    funResult.done(function(data){
        result.done && self.dispatchEvent(result.done,data);
    }).fail(function(data){
        result.fail && self.dispatchEvent(result.fail,data);
    }).always(function(data){
        result.always && self.dispatchEvent(result.always,data);
    })
}else{
    // 返回return的
    result.done && self.dispatchEvent(result.done,funResult);
}
```
这里的`dispatchEevnt`其实就是 `self.port.emit(result.done, { 'detail':  data});`, 这种触发也到达不了前端页面。

因此全部都要借助中转页面（`main.js`）才行。但是那么多的函数，而且每个函数都会有三个`defer`的返回监听，那不是写到手软？ 但是不是硬编码，其中还是有技巧的。 这个技巧就是在`main.js`中，绑定一个 `page_bind_handle` 监听，内容就是:
```javascript
// 初始化方法
component.port.on('page_bind_handle', function(result) {
    var funId = result;
    component.port.on(funId,function(data){
        var deferDoneId = data.done;
        var deferFailId = data.fail;
        var deferAlwayId = data.always;
        page.port.once(deferDoneId,function(data){
            component.port.emit(deferDoneId,data);
        });
        page.port.once(deferFailId,function(data){
            component.port.emit(deferFailId,data);
        });
        page.port.once(deferAlwayId,function(data){
            component.port.emit(deferAlwayId,data);
        });
        // 触发函数
        page.port.emit(funId,data);
    });
});
```
这个逻辑其实就是当前端页面触发 `page_bind_handle` 这个事件的话，会把一个函数的id名字传过来。然后前端页面再绑定`这个函数id`的监听，函数体就是，在背景页绑定`defer`的三个返回值`id`事件，然后触发的时候，再去触发前端页面对应的`defer`返回值`id`的监听。

看似不好理解，其实很巧妙。我们想要的结果是：当我们调用 `Airdroid.account.sign` 的时候, 发生以下步骤:
1. 在重新定义的`sign`方法中，绑定这个函数的`defer`三个返回值的一次性监听，然后触发这个函数所对应的函数`id`，并把`defer`是三个返回情况的`id`传过去
```javascript
self.eventObj.emit(funId,{
    arg: Array.prototype.slice.apply(arguments),
    done: deferDoneId,
    fail: deferFailId,
    always: deferAlwayId
});
```
2. 背景页面在收到这个事件之后，执行这个函数的函数体，并将函数体返回的`defer`情况，进行事件再触发，让其返回到前端
```javascript
self.addEventListener(uniqFunId,function(event){
    // 这边接收参数
    // 传过来的实际参数
    var result = event;
    // 真正的参数,而且是argument的形式
    var arg = result.arg;
    // 如果有回调，就触发调用
    var funResult = value.apply(obj,arg);
    // 判断是否是Deferred对象
    if(_.isFunction(funResult.done)){
        funResult.done(function(data){
            result.done && self.dispatchEvent(result.done,data);
        }).fail(function(data){
            result.fail && self.dispatchEvent(result.fail,data);
        }).always(function(data){
            result.always && self.dispatchEvent(result.always,data);
        })
    }else{
        // 返回return的
        result.done && self.dispatchEvent(result.done,funResult);
    }
});
```
这里的`dispatchEevnt`其实就是 `self.port.emit(result.done, { 'detail':  data});`, 而`addEventListener` 其实就是 `self.port.on(uniqFunId,cb);` 只是为了兼容各个浏览器，所以进行了封装。

3. 前端页面收到背景页那边执行函数之后的`defer`返回情况，再跟进返回的参数进行处理
```javascript
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
```
但是真正的执行情况是：
1. 前端页面在`addFireFoxTriggerHandle`函数中进行函数遍历之后，就会先把每一个`funId`都在中转页面（`main.js`）进行注册监听， 然后再进行函数重定义
```javascript
_.each(obj.funListenerObjs,function(funId,name){
    // 重新定义函数
    // 这时候，要把组件的监听对象，传到page页面去
    self.eventObj.emit('page_bind_handle',funId);
    obj[name] = function(){
        var defer = $.Deferred();
        ......
```
这时候在 `main.js` 就会触发 `page_bind_handle` 这个事件, 并监听前端的 `funId` 的这个事件
```javascript
component.port.on('page_bind_handle', function(result) {
    var funId = result;
    component.port.on(funId,function(data){
        var deferDoneId = data.done;
        var deferFailId = data.fail;
        var deferAlwayId = data.always;
        page.port.once(deferDoneId,function(data){
            component.port.emit(deferDoneId,data);
        });
        page.port.once(deferFailId,function(data){
            component.port.emit(deferFailId,data);
        });
        page.port.once(deferAlwayId,function(data){
            component.port.emit(deferAlwayId,data);
        });
        // 触发函数
        page.port.emit(funId,data);
    });
});
```
2. 接下来才是在前端触发这个监听的事件，当我们调用 `Airdroid.account.sign` 的时候, 在重新定义的`sign`方法中，绑定这个函数的`defer`三个返回值的一次性监听，然后触发这个函数所对应的函数`id`，并把`defer`是三个返回情况的`id`传过去
```javascript
self.eventObj.emit(funId,{
  arg: Array.prototype.slice.apply(arguments),
  done: deferDoneId,
  fail: deferFailId,
  always: deferAlwayId
});
```
3. 在中转页面（`main.js`）触发这个监听，在函数体内背景页绑定`data`的三个`defer`情况的返回回调`id`，并触发`背景页的函数funId监听`
```javascript
component.port.on(funId,function(data){
    var deferDoneId = data.done;
    var deferFailId = data.fail;
    var deferAlwayId = data.always;
    page.port.once(deferDoneId,function(data){
        component.port.emit(deferDoneId,data);
    });
    page.port.once(deferFailId,function(data){
        component.port.emit(deferFailId,data);
    });
    page.port.once(deferAlwayId,function(data){
        component.port.emit(deferAlwayId,data);
    });
    // 触发函数
    page.port.emit(funId,data);
});
```
4. 在背景页收到中转页面（`main.js`）的这个函数id触发，并执行函数体，执行完之后，触发`defer`的三种情况之一的回调事件触发
```javascript
self.addEventListener(uniqFunId,function(event){
  // 这边接收参数
  // 传过来的实际参数
  var result = event;
  // 真正的参数,而且是argument的形式
  var arg = result.arg;
  // 如果有回调，就触发调用
  var funResult = value.apply(obj,arg);
  // 判断是否是Deferred对象
  if(_.isFunction(funResult.done)){
      funResult.done(function(data){
          result.done && self.dispatchEvent(result.done,data);
      }).fail(function(data){
          result.fail && self.dispatchEvent(result.fail,data);
      }).always(function(data){
          result.always && self.dispatchEvent(result.always,data);
      })
  }else{
      // 返回return的
      result.done && self.dispatchEvent(result.done,funResult);
  }
});
```
5. 在中转页面（`main.js`）收到背景页的`defer`的回调事件触发，并触发`前端页面`的对应相同`id`的回调事件
```javascript
page.port.once(deferDoneId,function(data){
    component.port.emit(deferDoneId,data);
});
page.port.once(deferFailId,function(data){
    component.port.emit(deferFailId,data);
});
page.port.once(deferAlwayId,function(data){
    component.port.emit(deferAlwayId,data);
});
```
6. 前端页面收到中转页面的`defer`情况的回调事件，并对参数进行处理
```javascript
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
```

这样流程就结束了。这样子`Firefox`也可以在前端页面调用`背景页Airdroid对象`的方法了。经过我们这样一兼容，无论是`Chrome`，`Safari`，还是`Firefox`，调用方式都是一样的。 都是`Chrome` 那种调用方式:
```javascript
Airdroid.account.signIn(mail, pwd).done(function(){

}).fail(function(){

}).always(function(){

})
```
很明显这些方法都要是以`defer`的形式来返回的。其实如果是直接返回的话，那么也会调用`done`方法，但是前端还是得用`defer`的方式来写，这样才能保持一致。相对于`Chrome`和`Safari`的`popup`页面，`Safari`的非`popup`会复杂一点，而`Firefox`就更复杂了，还要再中转一次才行了。

## 2.前端页面监听一个事件，背景页触发这个事件
我们现在已经知道怎么从前端页面调用后台的对象以及里面的方法了。虽然各个浏览器的实现原理都不一样。但是总归是在写法上进行了统一。

但是仅仅这样子还是不够的，有时候需要在前端监听一些事件，以便应付背景页的特殊情况。比如在`popup`页面监听通知改变事件，监听push消息改变事件，登录，登出事件,这些都要背景页在状态变化的时候，要及时通知到前端页面才行。

所以 `popup.js` 的部分代码如下:
```javascript
// 监听一些全局的事件
addListenerGlobalEvent: function () {
    var self = this;
    // 通知推送过来
    self.addListener(self.EventType.notification_change, function (data) {
        // todo 
    });

    self.addListener(self.EventType.push_change, function (data) {
        // todo 
    });

    self.addListener(self.EventType.signed_in, function (data) {
        // todo 
    });

    self.addListener(self.EventType.signed_out, function (data) {
        // 如果在其他地方登录，就重新刷新
        // todo 
    });
},
```
在背景页的长连接通道如果有消息过来的话，这时候就会触发到前端页面，然后让前端页面直接调用背景页的方法获取数据，并进行渲染。

而 `addListener` 是在 `base.js` 进行封装的:
```javascript
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
```
而背景页对应的触发代码为 `events.js`:
```javascript
// 触发事件
dispatchEvent: function(eventName, details) {
    if (window.chrome || window.safari) {
        window.dispatchEvent(new CustomEvent(eventName, { 'detail': details }));
    } else {
        // firefox
        // todo 因为Firefox的特殊性，他不能再同一个js里面，同时绑定事件，并触发，而是都要经过main.js绕一圈
        // 有一种方法就是使用addEventListener，但是这个又不能传到前台去
        // 这时候要判断该事件是哪一种事件，是双向的，还是单向的， 即属于FireFoxEvent里面的，还是TYPE 里面的
        if(_.values(this.TYPE).indexOf(eventName) > -1){
            // 这时候要到外面绕一圈回来
            fireFoxEvent.emit(eventName + "_firefox", { 'detail': details });
        }else{
            fireFoxEvent.emit(eventName, { 'detail': details });
        }
    }
},
```
还是老样子，根据不同浏览器来分析。
### 2.1. Chrome 和 Safari
`Chrome` 和 `Safari` 在前端的监听方式都差不多, 在前端页面都是
```javascript
this.eventObj.addEventListener(name, function (event) {
    _.isFunction(fun) && fun(event.detail);
},false);
```
这里的`eventObj` 就是 `setUpBackgroundPage` 方法传过来的背景页的 `window` 对象。

注意： 对于`Safari` 非 `popup` 页面，这种绑定是不行的，因为他的`eventObj`对象其实是`fakeAirdroid`所在的对象，根本不是背景页的 `window`，根本通信不了背景页。之所以这么做，其实是因为我们只有在`popup`页面才会有监听的需求的，其他的前端页面暂时没有这个需求。

比如在 `reply` 页面(非 `popup`页面), 如果前端想要数据的话，就直接请求。比如当我打开`reply`进行回复的时候，就要有上一条的消息，这时候就会通过`addEventListener`的形式去获取, `replay.js` 部分代码如下:
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
而在背景页，就要处理这种情况，并把数据发过来, 比如背景页的 `notificationManage.js` 的部分代码如下:
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
var w = safari.application.openBrowserWindow();
w.activeTab.url = safari.extension.baseURI + spec.url + '&width=' + spec.width + '&height=' + spec.height;
```
这样就实现了`Safari` 非 `popup` 页面的数据请求机制。还是以事件驱动的方式来通信。

如果在`Chrome`或者`Safari` `popup` 页面进行监听的话，那么背景页的触发也是很简单。
```javascript
window.dispatchEvent(new CustomEvent(eventName, { 'detail': details }));
```
这样子前端页面的监听就会被触发到。从而实现背景页的通知功能。

### 2.2. Firefox
`Firefox` 会比较麻烦，因为还要过一次中转页面(`main.js`)。前端页面的逻辑也是比较简单的:
```javascript
this.eventObj.on(name, function(event){
    _.isFunction(fun) && fun(event.detail);
})
```
其中`eventObj` 就是 `self.port`, 背景页触发也是比较简单
```javascript
// firefox
// todo 因为Firefox的特殊性，他不能再同一个js里面，同时绑定事件，并触发，而是都要经过main.js绕一圈
// 有一种方法就是使用addEventListener，但是这个又不能传到前台去
// 这时候要判断该事件是哪一种事件，是双向的，还是单向的， 即属于FireFoxEvent里面的，还是TYPE 里面的
if(_.values(this.TYPE).indexOf(eventName) > -1){
    // 这时候要到外面绕一圈回来
    fireFoxEvent.emit(eventName + "_firefox", { 'detail': details });
}else{
    fireFoxEvent.emit(eventName, { 'detail': details });
}
```
可以看到，这个背景页有两个`if`分支，其中第二个分支 `fireFoxEvent.emit(eventName, { 'detail': details });` 其实就是前端页面调用背景页方法用的方法， 所以要实现事件触发监听，其实是在第一个if分支上。

从代码里面可以看到，他会被判断这个事件是否有在`this.TYPE`数组里面。 如果存在的时候，会去触发一个 `fireFoxEvent.emit(eventName + "_firefox", { 'detail': details });`, 为什么要这样处理呢？ 这个是因为要处理这个中转的问题。

 也就是说，如果前端页面监听了一个`notification_change`事件，要来监听通知的改变，那么等背景页触发这个通知改变的时候，怎么样才能把这个触发传到前端页面来呢？ 流程如下：
1. 背景页触发这个通知 
`events.js` 在初始化(`init` 函数)的时候，就会把`TYPE数组`的事件都注册监听
```javascript
var isListener = [self.TYPE.signed_in, self.TYPE.signed_out];
_.each(self.TYPE,function(value,key){
    self.dispatchEvent(self.FireFoxEvent.registerHandle, value);
    // 同时绑定所有监听的默认事件
    (isListener.indexOf(value) == -1) && Airdroid.Event.addEventListener(value, $.noop);
});
```
当背景页触发事件的时候
```javascript
Airdroid.Event.dispatchEvent(Airdroid.Event.FireFoxEvent.notifications_changed, _.keys(self._notificationsObj));
```
然后执行的时候，因为 `notifications_changed` 这个事件在 `this.TYPE` 里面，所以执行
```javascript
fireFoxEvent.emit(eventName + "_firefox", { 'detail': details });
```
2. 中转页面（`main.js`）收到这个触发(`notifications_changed_firefox`)，然后又把这个事件(`notifications_changed`)再回传回去（events初始化就已经绑定了）。
```javascript
// 注册可以双向通信的事件通道
page.port.on('registerHandle', function(event) {
    var handleName = event.detail;
    // 这边开始注册一个事件, 名字为 name 后面加上 firefox 字样, 主要是让后台来触发，再回传到后台去，相当于绕了一圈，这样子会比较麻烦，但是为了兼容其他浏览器和一致性，只能这样了
    page.port.on(handleName + "_firefox", function(data){
        // 同时回传过去
        page.port.emit(handleName, data);
        // 同时也回传到前台. 注意，这边不能直接在同一个main.js 里面触发，还是要先绕回去
        // page.port.emit(handleName + "_front", data);
    });
});
```
3. 背景页收到这个事件`notifications_changed`之后，触发执行之后，还要再回调到前端页面, 事件就是 `notifications_changed_front`
```javascript
addEventListener: function(eventName, listener, isOnce) {
  var self = this;
  if(window.chrome || window.safari){
      this.eventListeners.push({ 'eventName': eventName, 'listener': listener });
      window.addEventListener(eventName, listener, false);
  }else {
      // firefox
      // 这边还要把这个事件回调到前台去
      var cb = function(data){
          listener(data);
          // 还要回调到前台去，这边注意一个问题，如果有很多个前台的话，不知道会不会发生事件覆盖
          (_.values(self.TYPE).indexOf(eventName) > -1) && fireFoxEvent.emit(eventName + "_front", data);
      };
      if(isOnce){
          fireFoxEvent.once(eventName,cb);
      }else{
          fireFoxEvent.on(eventName,cb);
      }
  }
},
```
ps: 之所以绕了一圈重新回到 `notifications_changed` 回调事件，就是因为在 `Firefox`, 只要是事件触发，哪怕是都写在背景页后台中，都要到 `main.js` 绕一圈，不然没法触发。
4. 中转页面（`main.js`）收到这个触发，然后触发前端页面的这个触发
```javascript
// 绑定那些可以从后台触发到前台的事件，比如signed_in
component.port.on("page_register_handle",function(data){
    var name = data.name;
    // 绑定对应页面的前台事件, 然后再对应到前台所绑定的函数
    page.port.on(name + "_front", function(data){
        component.port.emit(name,data);
    })
});
```
5. 前端页面收到中转页面（`main.js`）的这个事件触发

所以整个流程，其实最重要的地方就是两个中转页面地方的处理：
一个是背景页面的`registerHandle`事件注册, 同时要把`TYPE`数组里面的除登入，登出之外的其他事件都注册（比如`push_change`，`notification_change`）,在 `events.js` 的代码片段就是:
```javascript
// 这时候要注册可以双向通行的事件
// todo 这边如果已经在 background.js 中已经注册的事件了，就不要重复绑定了,不然前台会触发两次
var isListener = [self.TYPE.signed_in, self.TYPE.signed_out];
_.each(self.TYPE,function(value,key){
    self.dispatchEvent(self.FireFoxEvent.registerHandle, value);
    // 同时绑定所有监听的默认事件
    (isListener.indexOf(value) == -1) && Airdroid.Event.addEventListener(value, $.noop);
});
```
背景页中转页面（`main.js`）的处理:
```javascript
// 注册可以双向通信的事件通道
page.port.on('registerHandle', function(event) {
    var handleName = event.detail;
    // 这边开始注册一个事件, 名字为 name 后面加上 firefox 字样, 主要是让后台来触发，再回传到后台去，相当于绕了一圈，这样子会比较麻烦，但是为了兼容其他浏览器和一致性，只能这样了
    page.port.on(handleName + "_firefox", function(data){
        // 同时回传过去
        page.port.emit(handleName, data);
        // 同时也回传到前台. 注意，这边不能直接在同一个main.js 里面触发，还是要先绕回去
        // page.port.emit(handleName + "_front", data);
    });
});
```
第二个就是前端页面要去绑定这个`page_register_handle`事件, `base.js` 代码片段如下：
```javascript
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
```
对应的中转页面（main.js）的代码片段为:
```javascript
// 绑定那些可以从后台触发到前台的事件，比如signed_in
component.port.on("page_register_handle",function(data){
    var name = data.name;
    // 绑定对应页面的前台事件, 然后再对应到前台所绑定的函数
    page.port.on(name + "_front", function(data){
        component.port.emit(name,data);
    })
});
```
可以看到，当获取到背景页过来的`Airdroid对象`的时候，它对每一个`EventType`的项都触发了一次`page_register_handle`事件，让背景页监听触发的事件。
而`EventType`数组也是从`Airdroid`对象获取的，在`setUpBackgroundPage`中设置。
```javascript
this.EventType = this.Airdroid.Event.TYPE;
```
其实就是`events.js`里面的`TYPE`
```javascript
TYPE: {
    // 登录事件
    "signed_in": "signed_in",
    // 登出事件
    "signed_out": "signed_out",
    // push事件改变事件
    "push_change": "push_change",
    // 普通通知改变事件
    "notification_change": "notification_change"
},
```
这里面就是所有能在前端页面进行监听的事件列表，也就是说，如果你要在前端页面进行一个新的事件监听，那么就要在这里面再添加一个新的事件。所以在前端页面这样执行完之后。
```javascript
_.each(_.values(self.EventType),function(item){
    fireFoxEvent.emit("page_register_handle",{name:item});
});
```
那么在中转页面（`main.js`）就多了四个监听了分别是:
```javascript
page.port.on("signed_in_front", function(data){
    component.port.emit('signed_in',data);
})
page.port.on("signed_out_front", function(data){
    component.port.emit('signed_out',data);
})
page.port.on("push_change_front", function(data){
    component.port.emit('push_change',data);
})
page.port.on("notification_change_front", function(data){
    component.port.emit('notification_change',data);
})
```
所以当中转页面（`main.js`）把`notifications_changed` 重新触发到背景页的时候:
```javascript
var cb = function(data){
    listener(data);
    // 还要回调到前台去，这边注意一个问题，如果有很多个前台的话，不知道会不会发生事件覆盖
    (_.values(self.TYPE).indexOf(eventName) > -1) && fireFoxEvent.emit(eventName + "_front", data);
};
if(isOnce){
    fireFoxEvent.once(eventName,cb);
}else{
    fireFoxEvent.on(eventName,cb);
}
```
`cb` 方法里面会去判断`notifications_changed`这个事件是否在`TYPE`里面，如果在里面，说明这个事件是可以传递到前端页面的。所以才要触发`notifications_changed_front`事件到中间页面（`main.js`），从而再从中间页面（`main.js`）才触发`notification_change`到前端页面去。

所以`Firefox`的前端页面监听背景页触发，就是这样子。

## 3.背景页监听一个事件，背景页和前端页面都触发这个事件
前面两点已经讲述了怎么在前端页面调用背景页的`Airdroid对象方法`，以及在前端事件监听，由背景页触发这个事件。但是还有一种情况，我们还要在处理，就是如果在背景页触发某个事件，不仅前端页面有收到（如果前端页面有监听这个事件），背景页也会收到（如果背景页也有监听这个事件）。

举个例子，针对登录成功`signed_in`这个事件监听，不仅前端页面有监听，在背景页也有监听, 比如前端的 `popup.js` 的代码如下:
```javascript
self.addListener(self.EventType.signed_in, function (data) {
  // 如果不是在前端登的，比如后台自动登录，那么要切换页面
  self.log("登录了==");
  if(!self._isSigning && self._dom.signInPage.inputEmail.is(":visible")){
  self.afterSignIn();
  self.signInEnableDom();
}});
```
然后背景页也有监听，比如 `background.js` 的代码如下:
```javascript
// 监听登录事件
Airdroid.Event.addEventListener(Airdroid.Event.TYPE.signed_in, function () {
  console.info("响应登陆事件");
  Airdroid.Cache.setCacheType();
  // 登录之后，初始化右键菜单
  Airdroid.Account.getDevicesObjList().done(function(data){
    Airdroid.ContextMenus.init(data);
  })
});
```
当`account`的`signIn`方法登录成功的时候。触发这个登录事件
```javascript
Airdroid.Event.dispatchEvent(Airdroid.Event.TYPE.signed_in);
```
这时候就要上述两个监听，都要可以触发到。 其实前端监听其实就是第二点。所以第三点主要讲的是，怎么在背景页也可以触发到。

从第二点我们可以知道，所有可以监听的事件都在`events`的这个数组里面:
```javascript
TYPE: {
  // 登录事件
  "signed_in": "signed_in",
  // 登出事件
  "signed_out": "signed_out",
  // push事件改变事件
  "push_change": "push_change",
  // 普通通知改变事件
  "notification_change": "notification_change"
},
```
按照惯例，我们还是按照不同的浏览器来分析
### 3.1 Chrome 和 Safari
都是背景页环境，直接
```javascript
window.addEventListener(eventName, listener, false);

window.dispatchEvent(new CustomEvent(eventName, { 'detail': details }));
```
这样就可以了。
### 3.2 Firefox
`Firefox` 其实就会比较麻烦，在第二点的时候，有讲过，当触发:
```javascript
Airdroid.Event.dispatchEvent(Airdroid.Event.FireFoxEvent.notifications_changed, _.keys(self._notificationsObj));
```
这个事件的时候,其实会触发`notifications_changed_firefox`, 然后逻辑跑到中间页面（`main.js`），在中间页面（`main.js`）触发`notifications_changed`事件，然后逻辑再跑回背景页。这时候就会触发背景页所监听的`notifications_changed`事件，这时候就执行了背景页所监听的事件，同时执行完之后，再接着触发前端页面的`notifications_changed`事件。
在 `events.js` 的 `addEventListener` 方法:
```javascript
addEventListener: function(eventName, listener, isOnce) {
    var self = this;
    if(window.chrome || window.safari){
        // todo
    }else {
        // firefox
        // 这边还要把这个事件回调到前台去
        var cb = function(data){
            listener(data);
            // 还要回调到前台去，这边注意一个问题，如果有很多个前台的话，不知道会不会发生事件覆盖
            (_.values(self.TYPE).indexOf(eventName) > -1) && fireFoxEvent.emit(eventName + "_front", data);
        };
        if(isOnce){
            fireFoxEvent.once(eventName,cb);
        }else{
            fireFoxEvent.on(eventName,cb);
        }
    }
},
```
所以这时候也是有触发背景页的监听的。

## 总结
通过三个需求点，我们主要梳理了这三个浏览器的事件驱动模式，尤其是 `Firefox` 复杂了很多，而且我们抽象成统一的写法，这样子在做业务的时候，我们就不需要考虑不同浏览器的不同写法了，因为对外暴露的方法都是统一的，我们不需要在这个层面去考虑浏览器兼容性。





