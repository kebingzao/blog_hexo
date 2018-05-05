---
title: 前端工具集(2) -- 原生实现jquery的document.ready
date: 2018-05-06 01:35:41
tags: js
categories: 前端工具集
---
### 前言
之前有一个项目要用到document.ready的方法，
毕竟document.ready的效果要比监听原生的document.onload事件来的优秀（陈独秀？？）,
(至于两者之间的差别，直接看实现的代码，懒得再解释了XD）但是又担心jquery的库太大。
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