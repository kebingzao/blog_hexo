---
title: IE6,IE7 加载图片不显示的问题
date: 2018-11-02 15:54:56
tags: ie
categories: 前端相关
---
## 前言
之前在做pc端内嵌页，有涉及到 IE6, IE7 的兼容。其中遇到一个情况，就是： 点击某一张图片，然后通过更换 css class 类名，将其变成另一张图片。
原来的图片是一张静态的 png 图片，点击播放之后，就变成另一张动态的 gif 图片。 这个 icon 的点击代码如下：
```html
<a class="item-audio-state" href="javascript:;"></a>
```
用的是 a 标签，因为 IE6 下只有 a 便签才支持 hover 效果。 而且为了兼容IE6还加上了 filter 来兼容：
<!--more-->
```html
.item-state-2 & {
  .retina-image("../img/voice_send.png", "../img/2x/voice_send.png");
  // ie 6 7 下用filter，不然会有锯齿
  .lt-ie9 & {
    background: none;
    filter: progid:DXImageTransform.Microsoft.AlphaImageLoader(src='../../img/voice_send.png');
  }
}
    
// 如果是播放中，那么就要换成播放的按钮
.item-state-12 & {
  .retina-image("../img/voice_send_playing.gif", "../img/2x/voice_send_playing.gif");
}
```
而且这个 filter 对 gif 图片还没有效果。 但是问题就出现在如果点击这个按钮的话，播放中的那个gif并没有显示出来？？？ 我的 windows 的 webview 内核是 IE7。
但是我一旦将 gif 图片换掉，换成 png 图片，就又可以显示？？？ 难道是 gif 图片有问题？？后面查了一下，原来是我在点击 icon 的那个 click 事件上，没有加上 **return false** 导致的。
```html
// 点击音频播放的那个播放键
    changeAudioStateHandle: function (e) {
        var self = this;
        var sthis = $(e.currentTarget);
        ...
        return false;
    },
```
那为啥要加上 **return false** 才能表现正常呢？ 后面查了一下文章，发现是这样子的：

IE6,7只有在用
```html
<a onclick="switch_image()" href="javascript:void(0);"></a>
```
这样动态加载图片是才会出现这种情况。这个问题是ie6中一个底层机制的bug，之后的版本已经解决了。 据说 **<a href="javascript:void(0)">** 或者 **<a href=#">** 这样使用a标签的话并不能阻止a标签最后触发一个什么行为， 导致ie6会错误的认为页面刷新或者重定向了，并且中断了当前所有连接，这样新图片的加载就被aborted了。
但是事实上 IE7 也有类似的问题，但是不完全一样， 可能 bug 没有完全修复完成吧 ，因为如果是换成 png 格式的话，是没问题的，但是换成 gif 就会显示不出来了。

最简单的解决方法有两个:
- 一个是在触发事件的最后返回 **return false**
- 另外一个就是用div替换a标签来用。 

我用的是第一种方式，就是在触发事件最后加上 **return false**， 就可以正常切换为 gif 图片了


