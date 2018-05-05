---
title: 解决ie6 下png 透明背景图片 有锯齿的bug
date: 2018-05-05 16:52:10
tags: ie
categories: 前端相关
---

之前在做pc端桌面应用内嵌页的时候，因为要兼容到XP，所以其实就相当于要兼容到IE6

后面发现在 ie 6 下，不支持透明背景的图片，一般的PNG透明都是 png 32，但是ie6 不支持 png32，只支持png 8.

![angry png](ie-png-fliter/angry.jpg)

比如原来的样式是这样写的：

{% codeblock lang:css %}
.icon-cloud {
  width: 20px;
  height: 20px;
  position: absolute;
  top: 25px;
  left: 20px;
  display: none;
  background-image: url(../img/cloud.png);
}
{% endcodeblock %}

在ie下会变成：
![old png](ie-png-fliter/1.png)

即出现很严重的锯齿

解决方法就是用 filter

{% codeblock lang:css %}
.icon-cloud {
  width: 20px;
  height: 20px;
  position: absolute;
  top: 25px;
  left: 20px;
  display: none;
  background-image: url(../img/cloud.png);
  // ie 6 7 下用filter，不然会有锯齿
  .lt-ie9 &{
    background: none;
    filter: progid:DXImageTransform.Microsoft.AlphaImageLoader(src='../../img/cloud.png');
  }
}
{% endcodeblock %}

结果图就是：
![new png](ie-png-fliter/2.png)

这样就解决了















