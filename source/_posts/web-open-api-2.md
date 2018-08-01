---
title: web站点 开放第三方API流程(2) - 详解 jschannel
date: 2018-07-25 21:12:55
tags: 
    - js
    - open api
categories: 前端相关
---
本节主要是围绕这个 {% post_link web-open-api-1 %} 中的那个 demo演示模块 来介绍。既然是iframe两个不同域之间的通信，那么就要用到一个html5知识 -- **postMessage**。
## postMessage
平时做web开发的时候关于消息传递，除了客户端与服务器传值还有几个经常会遇到的问题：
- 页面和其打开的新窗口的数据传递
- 多窗口之间消息传递
- 页面与嵌套的iframe消息传递
- 上面三个问题的跨域数据传递

<!--more-->
这些问题都有一些解决办法，但html5引入的message的API可以更方便、有效、安全的解决这些难题。postMessage()方法允许来自不同源的脚本采用异步方式进行有限的通信，可以实现跨文本档、多窗口、跨域消息传递。
postMessage(data,origin)方法接受两个参数：
- data:要传递的数据
html5规范中提到该参数可以是JavaScript的任意基本类型或可复制的对象，然而并不是所有浏览器都做到了这点儿，部分浏览器只能处理字符串参数，所以我们在传递参数的时候需要使用JSON.stringify()方法对对象参数序列化，在低版本IE中引用json2.js可以实现类似效果。
- origin：字符串参数
指明目标窗口的源，协议+主机+端口号[+URL]，URL会被忽略，所以可以不写，这个参数是为了安全考虑，postMessage()方法只会将message传递给指定窗口，当然如果愿意也可以建参数设置为"*"，这样可以传递给任意窗口，如果要指定和当前窗口同源的话设置为"/"。

至于postMessage的更多用法，不在本篇的讨论范围之内。

## jschannel
这是一个js的插件，其实就是对postMessage的再封装。 事实上，这个插件就是web站点跟第三方iframe页面通信的关键, 它用来提供与iframe的通信通道。
文档：http://mozilla.github.io/jschannel/docs/
github项目地址：https://github.com/mozilla/jschannel
举一个例子：
**parent.html**
{% codeblock lang:html %}
<html> 
    <head>
        <script src="src/jschannel.js"></script>
    </head> 
    <body> 
        <iframe id="childId" src="child.html"></iframe> 
    </body> 
    <script> 
        var chan = Channel.build({ 
            window: document.getElementById("childId").contentWindow, 
            origin: "*", 
            scope: "testScope" 
        }); 
        chan.call({ 
            method: "reverse", 
            params: "hello world!", 
            success: function(v) { 
                console.log(v); 
            } 
        }); 
    </script>
 </html>
{% endcodeblock %}

**child.html**
{% codeblock lang:html %}
<html>
    <head> 
        <script src="src/jschannel.js"></script> 
        <script> 
            var chan = Channel.build({
                window: window.parent, 
                origin: "*", 
                scope: "testScope"
            }); 
            chan.bind("reverse", function(trans, s) { 
                return s.split("").reverse().join(""); 
            }); 
        </script> 
    </head> 
</html>
{% endcodeblock %}

这样子，在子页面加载完之后，就会跟父页面建立channel通道，并绑定一个 **reverse** 的事件,然后父对象跟子对象创建成功的时候，就会向子页面传递一个叫 **reverse** 的方法，这样就会触发 子页面的 **reverse** 的方法了。
反之亦然，如果在父页面绑定了事件，然后子页面call该对象的话，也是可以的。当然还有一些其他的用法，比如 trans 参数的意思等等，都会再后面说到。

---
完整系列：
{% post_link web-open-api-1 %}
{% post_link web-open-api-2 %}
{% post_link web-open-api-3 %}
{% post_link web-open-api-4 %}
{% post_link web-open-api-5 %}
{% post_link web-open-api-6 %}
{% post_link web-open-api-7 %}

