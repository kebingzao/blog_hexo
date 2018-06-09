---
title: 前端工具集(4) -- 禁止backspace后退页面
date: 2018-06-09 17:30:30
tags: js
categories: 前端工具集
---
之前有在项目中遇到按下Backspace键让浏览器后退的问题， 主要逻辑就是当敲Backspace键时，事件源类型为密码或单行、多行文本的，或者是可编辑DIV，并且readonly属性不为true和enabled属性不为false的，则退格键生效，其他情况都失效。
网上其实有很多解决方法，但是或多或少都不够全面，后面自己修改了以下，用的这个版本会比较全面一点：
<!--more-->
{% codeblock lang:js %}
//禁止backspace后退
function BanBackSpace (e) {
    var ev = e || window.event;//获取event对象
    var obj = ev.target || ev.srcElement;//获取事件源
    var t = obj.type || obj.getAttribute('type');//获取事件源类型
    var isEditDiv = obj.nodeName === 'DIV' && obj.getAttribute("contenteditable") === "true";//判断是否是可编辑的DIV
    //获取作为判断条件的事件类型
    var vReadOnly = obj.readOnly;
    var vDisabled = obj.disabled;
    //处理undefined值情况
    vReadOnly = (vReadOnly == undefined) ? false : vReadOnly;
    vDisabled = (vDisabled == undefined && !isEditDiv) ? true : vDisabled;
    //当敲Backspace键时，事件源类型为密码或单行、多行文本的，
    //并且readOnly属性为true或disabled属性为true的，则退格键失效
    var flag1 = ev.keyCode == 8 && (t == "email" || t == "password" || t == "text" || t == "textarea" || isEditDiv) && (vReadOnly == true || vDisabled == true);
    //当敲Backspace键时，事件源类型非密码或单行、多行文本的，则退格键失效
    var flag2 = ev.keyCode == 8 && t != "password" && t != "text" && t != "textarea" && t != "email" && !isEditDiv;
    //判断
    if (flag2 || flag1) return false;
}
{% endcodeblock %}
如果是整个页面直接禁止掉的话，那么就是：
{% codeblock lang:js %}
window.onload=function(){
 //禁止后退键 作用于Firefox、Opera
 document.onkeypress=BanBackSpace;
 //禁止后退键 作用于IE、Chrome
 document.onkeydown=BanBackSpace;
}
{% endcodeblock %}


