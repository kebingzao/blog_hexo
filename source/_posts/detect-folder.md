---
title: 前端工具集(6) -- 上传的时候判断所选文件是否为文件夹
date: 2018-06-09 18:14:26
tags: js
categories: 前端工具集
---
之前做项目，有做到一个就是上传的功能，其中Chrome和Firefox支持文件夹上传的，但是Safari还不支持文件夹上传，因此需要去判断当前用户选择上传的文件是不是一个文件夹，如果是文件夹的话，那么就不让他上传，因此才有这个方法。
代码如下：
<!--more-->
{% codeblock lang:js %}
import BrowserUtil from './browser'
import OSUtil from './os'

/**
 * 判断文件是否为文件夹 (异步)
 * @参考资料 http://hs2n.wordpress.com/2012/08/13/detecting-folders-in-html-drop-area/
 * @param f 要检测的文件
 * @param callback 用来获取判断结果的回调
 */
export default function isFolder (f, callback) {
    var reader;
    callback = callback || function(){};
    if (!f.type) {
        // 如果是mac下safari的话，不用对size进行判断，因为mac 下 safari的文件夹是有大小的
        // 而其他浏览器，比如firefox，文件夹大小都是为0的
        if((OSUtil.MacOS && BrowserUtil.safari) ||
            (f.size % 4096 == 0 && f.size <= 102400)){
            try {
                reader = new FileReader();
                reader.onerror = function (e) {
                    callback(true);
                };
                reader.onload = function (e) {
                    callback(false);
                };
                reader.readAsDataURL(f);
            } catch (NS_ERROR_FILE_ACCESS_DENIED) {
                callback(true);
            }
        }else{
            callback(false);
        }
    } else {
        callback(false);
    }
}
{% endcodeblock %}
需要注意的一点就是，如果读取的是大的文本文件的话，会变成异步操作
{% codeblock lang:js %}
isFolder(file_, function (isFolder) {
    if (isFolder) {//文件夹
        console.log("is foloder")
    } else {
        console.log("is file")
    }
});
{% endcodeblock %}