---
title: 前端工具集(8) -- 兼容IE的脚本加载器
date: 2018-06-15 00:43:49
tags: js
categories: 前端工具集
---
前段时间在做pc内嵌页的时候，有遇到过异步加载语言文件，并且因为有兼容到XP系统，即IE6内核。所以就写了一个兼容IE6文档模式打开的脚本加载器。

{% codeblock lang:js %}
/**
*  脚本加载器 (兼容ie6，兼容文档系统打开方式)
*/
function LoadScript (url,successCb){
    var header = document.getElementsByTagName("head")[0];
    var loader = document.createElement('script');
    // 成功事件
    var successed = function(){
        successCb && successCb();
    };
    if (loader.readyState) { //IE
        loader.onreadystatechange = function(){
            if (loader.readyState == "loaded" || loader.readyState == "complete") {
                loader.onreadystatechange = null;
                successed();
            }
        }
    } else { //Others
        loader.onload = function(){
            successed();
        };
    }
    // 加载语言文件
    loader.src = url;
    header.appendChild(loader);
};
{% endcodeblock %}

调用的话：
{% codeblock lang:js %}
var path = baseUrl + "lang/en.js";
LoadScript(path, function(){
    console.log("load lang success");
});
{% endcodeblock %}