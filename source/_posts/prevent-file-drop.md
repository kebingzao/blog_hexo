---
title: 前端工具集(5) -- 文件拖入禁止浏览器响应打开
date: 2018-06-09 17:51:04
tags: js
categories: 前端工具集
---
之前项目中有出现一种情况，就是项目的网页上，有拖拽上传的操作，但是有时候用户drop的地方如果放错，比如我拖拽一张图片，然后放到页面放错了，不是放到指定的drop容器里面，而是其他元素，这时候浏览器就会将这张图片显示出来，以文档模式打开，并覆盖当前页面，这种体验是很糟糕的。因此我们需要禁止掉浏览器的这种默认行为。
具体代码如下：
<!--more-->
{% codeblock lang:js %}
/**
 * 当有文件拖入的时候，浏览器默认会打开拖入的文件，导致用户跳出我们的应用，为了防止
 * 这个问题的产生，可以对目标元素做一些特殊处理。这样当文件拖到这个目标元素上的时候，
 * 浏览器就不会打开文件了。
 * Tip:可以传入 document 对象，这样无论文件拖到页面任何位置，浏览器都不会打开文件了。
 * @param target {HTMLDocument|HTMLElement|jQuery} 目标
 * @param [onDrop] {Function}
 */
function PreventDefaultFileDrop (target, onDrop) {
    var targetEl;
    // ie9及以下，不支持拖放，所以直接返回false
    if(navigator.appName == "Microsoft Internet Explorer" && parseInt(navigator.appVersion.split(";")[1].replace(/[ ]/g, "").replace("MSIE","")) <= 9){
        return false;
    }
    if (target instanceof jQuery) {
        targetEl = target[0];
    } else if (target instanceof Element || target instanceof Document) {
        targetEl = target;
    } else {
        console.error('Target must be a HTMLDocument|HTMLElement|jQuery.');
    }

    targetEl.ondragover = function () {
        return false;
    };
    targetEl.ondrop = function (e) {
        e.preventDefault();
        onDrop && onDrop();
    };
}
{% endcodeblock %}
如果是直接禁止掉整个页面的话，用法就是：
{% codeblock lang:js %}
PreventDefaultFileDrop(document)
{% endcodeblock %}