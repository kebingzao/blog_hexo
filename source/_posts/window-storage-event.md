---
title: 相同站点在同一个浏览器的多个 tab 页面进行消息通信的方式
date: 2020-12-18 16:05:57
tags: js
categories: 前端相关
---
## 前言
之前做的一个项目，有一个需求是这样子：

站点有一个 websocket 的消息推送机制，并且允许多开页面，但是因为 websocket 当初设计机制的关系，导致就算同一个账号信息开了多个浏览器 tab 页面，也只有最后一个开的 tab 页面的 websocket 通道是 alive 的，其他页面的 websocket 都会被关闭， 这个就导致只有最后一个页面的消息是实时同步的， 其他页面的消息都没有办法更新。 但是产品又要求， 既然允许多开好几个 tab 页面，那么这几个页面的消息同步都应该要一样。

最直接的方式就是改 websocket 通道的机制，允许同一个账号的多个通道存在，但是因为历史包袱以及一些其他原因，一旦改了这个机制，可能会伤筋动骨（好几个项目都是基于这个设计）。 所以退而求其次， 当有一个页面的 websocket 通道收到消息的，要广播到其他的页面， 让其他的页面也知道，并且要把数据传过去。

## 实操
这个就涉及到多个 tab 页面的信息传递， 之前我在做 chrome extension 扩展的时候， 到是可以做到。 但是这个又不是扩展应用。 另外还有一个 api 是 postMessage 也用于不同页面的消息传递， 不过 postMessage 只能用于内嵌页面，也不适合这种场景。

不过后面还是找到了方法，那就是用 localstorage 的方式来处理。
<!--more-->
我们知道如果是同一个域名的话，localstorage 是会共享的。 所以这个就为数据获取和共享提供了可行性。 举个例子：

比如我在 A 页面控制台执行：
```text
localStorage.setItem("name","zach")
```
我在同一个域名的另一个 tab 页面 B 页面的控制台执行:
```text
localStorage.getItem("name")
"zach"
```
这时候是可以取到数据的

![1](1.png) 

所以数据共享是没问题的。 那么怎么通知呢，尤其是当数据变化的时候，刚好 storage 提供了一个监听方法，当 storage 变化的时候，可以通过监听事件来触发， 具体看: [Window: storage event](https://developer.mozilla.org/en-US/docs/Web/API/Window/storage_event)

写法是这样子:
```text
window.addEventListener('storage', () => {
  // When local storage changes, dump the list to
  // the console.
  console.log(JSON.parse(window.localStorage.getItem('sampleList')));
});
```
或者是
```text
window.onstorage = () => {
  // When local storage changes, dump the list to
  // the console.
  console.log(JSON.parse(window.localStorage.getItem('sampleList')));
};
```

接下来我们实践一下， 在页面 B 的控制台监听这个函数:
```text
window.addEventListener('storage', () => {
  // When local storage changes, dump the list to
  // the console.
  console.log("change after:" + window.localStorage.getItem('name'));
});
```

![1](3.png) 

这时候当我 setItem 改变 name 的值的时候，这边就会触发，并打印。 removeItem 行为也会改变。

也就是说，每次我想通知其他 tab 页面的站点的时候，只需要让其他页面监听这个事件，然后要广播的那个页面再去修改 localstorage 的值就行了。 因为一旦修改，就会被广播。

## 缺陷
后面测试发现有一种情况会同步不了， 就是同一个 pc 上的，都是 chrome 浏览器，如果是不同用户登录下的行为。 我发现虽然都是同一个域名， 但是 localstorage 并不会共享。 

比如我用用户 A 登录了chrome浏览器，打开了站点 foo.com， 然后写入一条 localstorage 记录。 这时候再用 B 用户登录了chrome 浏览器， 也打开了 foo.com 站点，这时候他的 localstorage 其实的空的，并不会跟 A 用户的信息有关联。

所以我猜测chrome 在不同用户登录下的信息都是隔离的。 不过这种情况正常情况下，也比较少人会去这么干。

## 总结
通过这种方式，我们就可以实现同域名的站点在同个浏览器多开之后的消息传递机制了。

