---
title: IE 6, IE 7 调试神器 firebug lite
date: 2018-06-10 01:23:07
tags: ie
categories: 前端相关
---
之前在做一个pc端内嵌页项目的时候，因为要兼容XP系统，而XP系统的webview内核是 ie6，所以在调试IE 6， IE 7 的时候，发现调试非常难调，尤其是webview调试，根本没有所谓的调试窗口，后面发现有一个神器，就是firebug lite版，可以让你轻松的调试ie。
其实就是在html标签后面嵌入一段js：
{% codeblock lang:html %}
<script type="text/javascript">
    javascript:(function(F,i,r,e,b,u,g,L,I,T,E){
        if(F.getElementById(b))return;
        E=F[i+'NS']&&F.documentElement.namespaceURI;
        E=E?F[i+'NS'](E,'script'):F[i]('script');
        E[r]('id',b);
        E[r]('src',I+g+T);
        E[r](b,u);
        (F[e]('head')[0]||F[e]('body')[0]).appendChild(E);
        E=new Image;
        E[r]('src',I+L);
    })(document,'createElement','setAttribute','getElementsByTagName','FirebugLite','4','firebug-lite.js','releases/lite/latest/skin/xp/sprite.png','https://getfirebug.com/','#startOpened');
</script>
{% endcodeblock %}
只要放在 html 闭合标签的后面，让其加载，然后就可以出现调试窗口框了
<!--more-->
在IE 7 下 表现为：
![1](firebug-lite/1.png)
在webview 也很吊，甚至可以输入命令
![1](firebug-lite/2.png)
以后调试ie6,ie7 就靠他了