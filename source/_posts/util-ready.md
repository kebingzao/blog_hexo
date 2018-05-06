---
title: 前端工具集(2) -- 原生实现jquery的document.ready
date: 2018-05-06 01:35:41
tags: js
categories: 前端工具集
---
### 前言
之前有一个项目要用到document.ready的方法，
毕竟document.ready的效果要比监听原生的window.onload事件来的优秀（陈独秀？？）,但是又担心jquery的库太大。
因此就自己去扣 jquery的 document.ready, 然后拿过来用（还能再无耻一点吗）

### 前端实现

{% codeblock lang:js %}
// document ready

(function(loadCb){
    var DOMContentLoaded;
    if ( document.addEventListener ) {
        DOMContentLoaded = function() {
            document.removeEventListener( "DOMContentLoaded", DOMContentLoaded, false );
            loadCb();
        };
    } else if ( document.attachEvent ) {
        DOMContentLoaded = function() {
            // Make sure body exists, at least, in case IE gets a little overzealous (ticket #5443).
            if ( document.readyState === "complete" ) {
                document.detachEvent( "onreadystatechange", DOMContentLoaded );
                loadCb();
            }
        };
    }
    if ( document.readyState === "complete" ) {
        // Handle it asynchronously to allow scripts the opportunity to delay ready
        return setTimeout(loadCb, 1 );
    }
    // Mozilla, Opera and webkit nightlies currently support this event
    if ( document.addEventListener ) {
        // Use the handy event callback
        document.addEventListener( "DOMContentLoaded", DOMContentLoaded, false );
        // A fallback to window.onload, that will always work
        //window.addEventListener( "load", loadCb, false );
        // If IE event model is used
    } else if ( document.attachEvent ) {
        // ensure firing before onload,
        // maybe late but safe also for iframes
        document.attachEvent( "onreadystatechange", DOMContentLoaded );
        // A fallback to window.onload, that will always work
        //window.attachEvent( "onload", loadCb );
    }
})(function(){
    // document ready ，do something you want！！！
});
{% endcodeblock %}

---
ps：最后附上两者的差别

在jQuery中，load是所有Dom元素创建完毕、图片、Css等都加载完毕后才被触发，而ready则是Dom元素创建完毕后就被触发，这样可以提高网页的响应速度。在jQuery中可以用$(document).ready()来实现ready事件的调用,通过$(window).load()来实现load事件调用的时机。

比方说我们得网站上有个美女图片，点击图片就可以有某某效果，如果是ready的话，即使没有加载完，他也可以出来效果，但是onload却不可以，所以ready可以提高响应速度。我们一般也使用ready。ready的简写方式为$();

----

pps： 经过测试，使用onready 的速度 会比 使用 onload 的速度，快上几个数量级，从几百上千毫秒，提升到 几十毫秒

使用 
{% codeblock lang:js %}
console.time("==load==");
console.timeEnd("==load==");
{% endcodeblock %}
来跟踪加载时间


