---
title: webrtc 视频流在 IOS 下会出现黑屏
date: 2022-08-16 14:38:32
tags: 
- webrtc
- browser
categories: webrtc相关
---
## 前言
之前在实作 webrtc 浏览器对浏览器的远程投屏的时候， 有发现了一种情况，就是在 IOS 下， 远程传输过来的视频流 (mediaStream), 不管是在 chrome 还是 safari 下都没有办法播放, 不同版本的 IOS 表现还不一样:
1. IOS 14 及以下，会出现只播放第一帧， 就卡住
2. IOS 14 以上， 整个视频流直接黑屏

因为是自动播放，之前的代码是这样子的:
```javascript
video.play()?.catch(e => {
  // 刚开始初始化的时候，要禁音，不然 chrome 会报这个错误 Uncaught (in promise) DOMException: play() failed because the user didn't interact with the document first.
  video.muted = true
  video.play()
})
```

因为相当于也是自动播放， 在 PC 和 android 浏览器都正常，就是 IOS 浏览器不正常。

后面查了一下，发现还真是要加一个 IOS 系统特有的属性:[webrtc with firebase :how to fix black screen on ios/safari](https://stackoverflow.com/questions/60233093/webrtc-with-firebase-how-to-fix-black-screen-on-ios-safari)

在调用 `video.play()` 之前，要加上这个设置项:
```javascript
video.playsInline = true
```
就可以正常播放了

后面查了一下，原来在 IOS 系统上，如果要实现自动播放， 直接在标签上添加 `autoplay`, 或者 js 直接调用 `video.play()`, 是不行的。

还要加上 playsInline 属性才行， 其实就是允许视频内屏播放(没有这个属性也不一定是默认全屏播放，不过对于 IOS 来说，没有这个属性， 自动播放都不行)。



