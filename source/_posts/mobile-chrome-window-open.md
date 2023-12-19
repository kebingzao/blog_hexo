---
title: 移动端 chrome window.open 失效的问题
date: 2021-04-26 11:39:08
tags:  
- js
- chrome
categories: 前端相关
---
## 前言
之前有发生过一个比较奇葩的问题， 就是我们的测试人员有发生过， 移动端的 chrome 点击下载链接失效的情况。 但是 pc 端的 chrome 是正常的。 并且移动端的非chrome 浏览器， 比如 Firefox 或者 Safari 也是正常的。

## 排查
后面调试了一下，发现这个跳转用的是 `window.open` 语法:
```text
window.open(url)
```
我们知道 window.open 的触发必须得有用户点击触发才行(具体可以看: {% post_link browser-trigger %}), 所以我们也是绑定到 click 事件中的。 所以应该不是这个问题， 因为如果是这个问题，其他浏览器也是不行的。

而且通过调试，我发现窗口是有弹出来的，但是一直卡在一个空的页面，并没有完整的跳转过去。 所以应该是 chrome 检测到这次的跳转不安全，所以禁止了。

而关于 window.open 还真有一个安全隐患，那就是子窗口可以得到父窗口 `opener` 句柄, 从而在用户不知道的情况下， 对父窗口的导航栏进行重定向， 这个其实对用户来说是一个安全隐患的， 我之前有写过这方面的安全和应用方法，具体可以看:
 - {% post_link a-target-blank %}
 - {% post_link window-open-param %}

他虽然是一个安全隐患，但是他并不是毫无用处的， 还是有其他用处的， 但是对于本例来说， 可能确实有这种问题，所以我就改成
```text
var newWnd = window.open();
newWnd.opener = null;
newWnd.location = url;
```
改成这样子，果然 移动端的 chrome 浏览器就可以正常了。 

当然如果只是点击下载的问题，那么可以不使用 window.open， 改成 a 标签跳转 或者 iframe 下载也是可以的。 

但是在一些手机浏览器，比如 Android 和 iOS 的 opera、iOS 的 UC，`window.open()` 为 null, 所以可以扩展为:
```text
export function safeOpenWindow(url, target = '_blank') {
  const newWnd = window.open()
  // 这边要注意 Android和iOS的opera、iOS的UC，window.open() 为 null
  if (newWnd) {
    newWnd.opener = null
    newWnd.location = url
  } else {
    const aLink = document.createElement('a')
    aLink.setAttribute('target', target)
    aLink.setAttribute('href', url)
    aLink.setAttribute('rel', 'nofollow noopener noreferrer')
    document.body.appendChild(aLink)
    aLink.click()
    aLink.remove()
  }
}
```








