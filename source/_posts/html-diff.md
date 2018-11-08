---
title: 不同浏览器的html差异整理
date: 2018-11-08 16:37:55
tags: browser
categories: 前端相关
---
## 前言
这样主要是整理一下，之前做过的兼容一些浏览器的事情，而且因为IE比较特殊，而且各个版本差异化非常大，所以这边有针对IE进行归纳：{% post_link ie-compatible %}
而这边更多的是在html 和 JavaScript 的使用上来进行整理， css 会比较少。
## 自定义属性问题
**说明：**
IE下,可以使用获取常规属性的方法来获取自定义属性,也可以使用getAttribute()获取自定义属性;Firefox下,只能使用getAttribute()获取自定义属性.
**解决方法:**
统一通过**getAttribute()**获取自定义属性。
<!--more-->
## event.x与event.y问题
**说明：**
IE下,even对象有x,y属性,但是没有pageX,pageY属性;Firefox下,even对象有pageX,pageY属性,但是没有x,y属性.
**解决方法:**
使用
```javascript
mX = event.x ? event.x : event.pageX;
```
来代替IE下的event.x或者Firefox下的event.pageX.
## event.srcElement问题
**说明：**
IE下,event对象有srcElement属性,但是没有target属性;Firefox下,even对象有target属性,但是没有srcElement属性.
**解决方法:**
使用
```javascript
obj = event.srcElement ? event.srcElement : event.target;
```
来代替IE下的event.srcElement或者Firefox下的event.target。 请同时注意event的兼容性问题。
## innerText 问题
innerText在IE中能正常工作，但是innerText在FireFox中却不行. 需用textContent。
## body 对象
**说明：**
FF的 body 在 body 标签没有被浏览器完全读入之前就存在，而IE则必须在 body 完全被读入之后才存在,这会产生在IE下，文档没有载入完时，在body上appendChild会出现空白页面的问题
**解决方法:**
一切在body上插入节点的动作，全部在onload后进行
## nodeName 和 tagName 问题
**说明：**
在FF中，所有节点均有 nodeName 值，但 textNode 没有 tagName 值，在IE中，nodeName 的使用有问题
**解决方法:**
使用 tagName，但应检测其是否为空
## removeNode 方法
**说明：**
FF中节点自己没有 removeNode 方法
**解决方法:**
必须使用如下方法 node.parentNode.removeChild(node)
## 关于frame
**说明：**
在IE中可以用 window.testFrame 取得该frame，FF中不行
**解决方法:**
```javascript
window.top.document.getElementById("testFrame").src = 'xx.htm'
window.top.frameName.location = 'xx.htm'
```
## document.form.item 问题
**说明：**
代码中存在 document.formName.item("itemName") 这样的语句，不能在FF下运行
**解决方法:**
改用 document.formName.elements["elementName"]
## parentNode 问题
**说明：**
当html中节点缺失时，IE和FF对 parentNode 的解释不同,demo:
```html
<form>
<table>
<input/>
</table>
</form>
```
FF中 input.parentNode 的值为form，而IE中 input.parentNode 的值为空节点
## 调用子框架或者其它框架中的元素的问题
**说明：**
在IE中，可以用如下方法来取得子元素中的值
```javascript
document.getElementById("frameName").(document).elementName
window.frames["frameName"].elementName
```
**解决方法:**
在FF中则需要改成如下形式来执行，与IE兼容：
```javascript
window.frames["frameName"].contentWindow.document.elementName
window.frames["frameName"].document.elementName
```
## 对象宽高赋值问题
**说明：**
FireFox中类似 **obj.style.height = imgObj.height** 的语句无效
**解决方法:**
统一使用 **obj.style.height = imgObj.height + "px"**;




















