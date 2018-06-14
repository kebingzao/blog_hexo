---
title: 前端工具集(9) -- 正则匹配文本中的链接并转化为a便签
date: 2018-06-15 01:01:27
tags: js
categories: 前端工具集
---
前段时间在做pc内嵌页的时候，有遇到过一个需求，就是要把用户聊天记录中的链接匹配出来，并转化为A标签，这个就涉及到链接的正则匹配了。刚开始试着找了一下网上的正则匹配表达式，后面发现还是不全。
后面只好自己写了一个方法，总算可以满足大部分的情况，不过还是有一点不足的是，就是还是不能匹配ip地址的链接，刚开始有试着兼容了一下这个模式，后面发现要兼容这个东西的坑太多了，就算能兼容少部分的情况，但是很多情况下还是会有bug，再加上平时ip地址的链接还是比较少的，因此就先不兼容了。代码如下：

{% codeblock lang:js %}
function Linkify = function (text, opts) {
    var target, href;
    opts = opts || {};
    target = opts.target || '_blank';
    // 这个正则不匹配 ip 地址， 比如  http://192.168.40.124:8090/issues/?filter=11603
    // be 后缀因为 youtube 短链接是 be 后缀的，会比较常用。如： https://youtu.be/uFngFoR_3KE https://321.show/rAN0Q5M5wp6
    var urlReg = /((http|https|ftp)\:\/\/)?([a-zA-Z0-9]([a-zA-Z0-9\-]{0,61}[a-zA-Z0-9])?\.)+(com|me|io|org|net|de|uk|fr|us|cn|hk|tw|gl|be|show|au)(([:\/?])([\w\-\.,@?^=%&amp;:/~\+#*]+[\w\-\@?^=%&amp;/~\+#]{0,1}))?/g;
    var originText = text;
    var encryObj =[];
    text = text.replace(urlReg, function (match) {
        if(originText.substr(arguments[arguments.length-2]-1,1) == "@"){
            // 如果是邮箱，就不管
            return match;
        }
        if (match.indexOf('http') > -1) {
            // http/https 协议开头
            href = match;
        }  else {
            // 没有协议
            href = 'http://' + match;
        }
        // 使用约定的一个特殊字符来代替
        encryObj.push('<a class="linkable" href="' + href + '" target="' + target + '">' + match + '</a>');
        return ("$=A={{"+ (encryObj.length - 1) +"}}=A=$");
    });
    // 设置html代码格式
    text = text.replace(/\n/g, '<br>');
    // 这里不能转义，不然多字符的时候，会折行
    //text = text.replace(/\s/g, '&nbsp;');
    // 最后再转回来
    for (var i = 0; i < encryObj.length; i++){
        text = text.replace(new RegExp("\\$=A=\\{\\{" + i + "\\}\\}=A=\\$", "g"), encryObj[i]);
    }
    return text;
};
{% endcodeblock %}

调用的话：
{% codeblock lang:js %}
var text = '我的网站 www.sample.com.欢迎光临。';
console.log(Linkify(text));
最后得到：
我的网站 <a class="linkable" href="http://www.sample.com" target="_blank">www.sample.com</a>.欢迎光临。
{% endcodeblock %}