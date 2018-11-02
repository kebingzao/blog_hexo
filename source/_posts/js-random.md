---
title: 前端工具集(12) -- 原生js实现生成随机字符串
date: 2018-11-02 17:43:06
tags: js
categories: 前端工具集
---
## 原理
利用**Math.random**和**toString**生成随机字符串。这里的技巧是利用了toString方法可以接收一个基数作为参数的原理，这个基数从2到36封顶。如果不指定，默认基数是10进制。
```javascript
function generateRandomAlphaNum(len) {
    var rdmString = "";
    for (; rdmString.length < len; rdmString += Math.random().toString(36).substr(2));
    return rdmString.substr(0, len);
}
```   
![1](js-random/1.png)



