---
title: ie8,ie9 使用 XDomainRequest 进行跨域
date: 2018-06-14 00:48:52
tags: ie
categories: 前端相关
---
之前有写过一篇关于ajax的封装库：{% post_link js-ajax %} 里面有提到如果是ie10以下的ajax请求，都是用ActiveXObject这个对象来进行异步请求的。而且只能进行同域下的异步请求。
### ActiveXObject对象
ActiveXObject 这个对象是 ie5 才加进去的，也就是如果要在 ie5 及以上（ie5， ie6， ie7， ie8， ie9）要执行同域下的异步请求的话， 那么就用这个。
<!--more-->
{% codeblock lang:js %}
if(window.ActiveXObject)//IE
{
    try {
        //IE6以及以后版本中可以使用
        xmlHttp = new ActiveXObject("Msxml2.XMLHTTP");
    }
    catch (e) {
        //IE5.5以及以后版本可以使用
        xmlHttp = new ActiveXObject("Microsoft.XMLHTTP");
    }
}
else if(window.XMLHttpRequest)//其他
{
     xmlHttp = new XMLHttpRequest();
}
{% endcodeblock %}

### XDomainRequest对象
但是如果是用进行跨域请求的话，就不能用ActiveXObject这个对象来，而是只能用XDomainRequest对象。
XDomainRequest 是 IE 8 引入的，为了跨域，IE10 以上的浏览器支持使用 XMLHTTPRequest 对象，进行跨域资源共享(CORS)访问。IE11 废弃了 XDomainRequest 对象，并且此对象在IE11的Edge模式下不可用。所以一般会用到 XDomainRequest， 也就是 IE8 和 IE 9的跨域请求才会用到。 到了 ie10 就开始用 xhr 来进行跨域了（不过 ie10 跨域的时候，不支持 withCredentials）。
{% blockquote https://developer.mozilla.org/en-US/docs/Web/API/XDomainRequest %}
XDomainRequest is an implementation of HTTP access control (CORS) that worked in Internet Explorer 8 and 9. It was removed in Internet Explorer 10 in favor of using XMLHttpRequest with proper CORS; if you are targeting Internet Explorer 10 or later, or wish to support any other browser, you need to use standard HTTP access control.
{% endblockquote %}
而且使用 XDomainRequest 是有使用限制的：
1. 必须使用 HTTP 或 HTTPS 协议访问目标 URL （也就是本地文件打开的页面就不能进行ajax请求了，因为是 file:// 协议）。
2. 只能使用 HTTP 的 GET 方法和 POST 方法访问目标 URL
3. 请求中不能加入自定义的报头
4. 身份验证和cookie不能和请求一起发送
5. 请求URL必须和主页URL采用相同的协议，而且客户端和服务端请求的协议一定要一致，不能一个是http，一个是https(<font color=red>这一点我踩过坑，就是web站点是http协议的，然后服务端接口是https的，然后就报错了</font>)
6. 支持的事件有：onerror，onload，onprogress，ontimeout
7. 提供的方法：abort，open，send
8. 提供的属性：contentType， responseText，timeout

而且IE8，在用户使用InPrivate浏览模式浏览网站的时候，所有的XDomainRequest 将会失败并报错。

### 封装post跨域的函数
早期项目在ie8，ie9进行post跨域的时候，用的就是这个函数，至于为啥没有get的函数，是因为get全部用jsonp的方式来实现跨域请求
{% codeblock lang:js %}
function IEPost (url, originalParams, sucFunc, errFunc, isToText) {
    var xdr = new XDomainRequest();
    //IE Post地址需加上验证信息，因为COOKIE带不过去
    xdr.open("post", url);
    xdr.onload = function () {
       if ((isToText || false) == true) {
            var result = xdr.responseText;
       } else {
            var result = $.parseJSON(xdr.responseText);
       }
       if (_.isFunction(sucFunc)) {
            sucFunc(result);
       }
    };
    var failCb = function(){
        if (_.isFunction(errFunc)) {
            errFunc();
        }
        console.log("IE post error");
    };
    xdr.onerror = failCb;
    xdr.ontimeout = failCb;
    // 一定要加上这个事件，不然其他事件都不能用了
    xdr.onprogress = function() {};
    xdr.timeout = 10000;
    xdr.send(originalParams);
}
{% endcodeblock %}

### 使用jquery插件[jQuery-ajaxTransport-XDomainRequest](https://github.com/MoonScript/jQuery-ajaxTransport-XDomainRequest)
后面发现jquery有个插件 jQuery-ajaxTransport-XDomainRequest 直接可以无缝兼容它的 ajax 方法，不需要进行额外处理，也就是说，当jquery检测到当前环境是ie8，ie9的时候，就会切换成用XDomainRequest这个对象进行跨域请求。

{% blockquote https://github.com/MoonScript/jQuery-ajaxTransport-XDomainRequest %}
Implements automatic Cross Origin Resource Sharing support using the XDomainRequest object for IE8 and IE9 when using the $.ajax function in jQuery 1.5+.
CORS requires the Access-Control-Allow-Origin header to be present in the AJAX response from the server.
In order to use XDomainRequest in Internet Explorer, the request must be:

Only GET or POST
When POSTing, the data will always be sent with a Content-Type of text/plain
Only HTTP or HTTPS
Protocol must be the same scheme as the calling page
Always asynchronous
Working example here: http://jsfiddle.net/MoonScript/Q7bVG/show/
{% endblockquote %}

代码跟正常的ajax请求一样：
{% codeblock lang:js %}
// GET
$.getJSON('http://jsonmoon.jsapp.us/').done(function(data) {
  console.log(data.name.first);
});

// POST
$.ajax({
  url: 'http://frozen-woodland-5503.herokuapp.com/cors.json',
  data: 'this is data being posted to the server',
  contentType: 'text/plain',
  type: 'POST',
  dataType: 'json'
}).done(function(data) {
  console.log(data.name.last);
});
{% endcodeblock %}