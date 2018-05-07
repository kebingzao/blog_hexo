---
title: 前端工具集(1) -- jsonp原生实现
date: 2018-05-06 01:08:31
tags: js
categories: 前端工具集
---
### 前言

做了好几年的前端了，也用了很多的工具函数，比如 jquery，underscore，lodash 之类的。
但是其实很多时候为了减少体积，其实很多的工具都是自己写的，之前都是放在 evernote 里面。
这次为了充实我的blog，我只能厚颜无耻都重新贴出来了 XD

### jsonp 原理 （简单列一下，不打算科普，因为打字好蛋疼）

jsonp是一种跨域通信的手段，它的原理其实很简单：
1. 首先是利用script标签的src属性来实现跨域（跨过同源策略）。
2. 然后由服务器端注入参数之后再返回，实现服务器端向客户端通信。
3. 由于使用script标签的src属性，因此只支持get方法

### 前端实现工具类
<!--more-->
{% codeblock lang:js %}
// jsonp 方法
window.util.jsonp = function (url, data, successCb, failCb, alwaysCb) {
    var callbackName, script;
    var toUrlParam = function(obj) {
        var arr = [];
        for (var key in obj) {
            arr.push(key + '=' + encodeURIComponent(obj[key]));
        }
        return arr.join('&');
    };
    if (typeof url === 'object') {
        var tmp = url;
        url = tmp.url;
        data = tmp.data;
        successCb = tmp.success;
        failCb = tmp.fail;
        alwaysCb = tmp.always;
    } else if (typeof data === 'function') {
        successCb = data;
        data = {};
    }
    data.callback = 'jsonp' + new Date().getTime();
    callbackName = data.callback;
    url += '?' + toUrlParam(data);
    script = document.createElement("script");
    window[callbackName] = function (response) {
        try {
            successCb && successCb(response);
        } catch (e) {
            failCb && failCb(e);
        } finally {
            alwaysCb && alwaysCb();
            delete window[callbackName];
            script.parentNode.removeChild(script);
        }
    };
    script.src = url;
    //出错处理
    script.onerror = function () {
        failCb && failCb({error:"error"});
        alwaysCb && alwaysCb();
        delete window[callbackName];
        script.parentNode.removeChild(script);
    }
    document.body.appendChild(script);
};
{% endcodeblock %}

调用：
{% codeblock lang:js %}
util.jsonp("https://xxx.xxx.com/device/webunbind/",{"q": token}, 
    function(){
        document.getElementById("result").innerHTML = "unbound Done";
    }, function(){
        document.getElementById("result").innerHTML = "unbound Fail"
    });
{% endcodeblock %}

### 服务端实现

{% codeblock lang:js %}
var http = require('http');
var urllib = require('url');

var port = 8080;
var data = {'data':'world'};

http.createServer(function(req,res){
    var params = urllib.parse(req.url,true);
    if(params.query.callback){
        console.log(params.query.callback);
        //jsonp
        var str = params.query.callback + '(' + JSON.stringify(data) + ')';
        res.end(str);
    } else {
        res.end();
    }
    
}).listen(port,function(){
    console.log('jsonp server is on');
});
{% endcodeblock %}

