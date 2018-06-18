---
title: 使用input=file 上传的时候，选择同样的图片，第二次不会再触发onChange事件
date: 2018-06-19 00:38:30
tags: js
categories: 前端相关
---
之前在做一个项目的时候，有用到 input=file 上传， 并监听onchange事件。也就是当选择的文件有变化的时候，就会触发change事件。但是有一种，就是第二次选择的图片和第一次的图片一样。这时候，发现第二次选择之后，就不会再触发change事件了。
解决方法就是每次上传之后， 把这个 input 的 value 清空， 即 event.target.value = '';但是这种方法在ie10下会有问题。所以更好的方法就是在input上面套一层form，然后使用 form.reset() 方法来清除。
比如这样：
<!--more-->
{% codeblock lang:js %}
var resetFileInput = function () {
    $(e.currentTarget).parent("form")[0].reset();
};
{% endcodeblock %}

---

截取一部分项目对于这部分的上传处理代码，在外面套了一层form，直接选取里面的input组件进行监听：
{% codeblock lang:js %}
$folderSelectForm.find("input").change(
    function (e) {
        var selectedFiles = e.target.files;
        var selectedFilesLength = selectedFiles.length;
        //文件个数小于1，即用户没有选择文件
        if (selectedFilesLength < 1) {
            return;
        }
        var resetFileInput = function () {
            $(e.currentTarget).parent("form")[0].reset();
        };
        // 开始上传
        self.onFileInputChange(selectedFiles, resetFileInput);
    })
{% endcodeblock %}
上传的部分处理是<font color=red>onFileInputChange</font>,第二个参数就是上传成功的回调，这边涉及到大量的上下文代码，因为篇幅原因，只列了处理上传的部分代码：
{% codeblock lang:js %}
// 当选择文件上传的时候，文件改变事件
onFileInputChange: function (selectedFiles, allListCallBack) {
    var self = this;
    //调用上传前事件
    self.beforeLoad();
    // 文件上传列表队列
    var uploadListQueque = [];
    //遍历文件列表，加到上传队列
    _.each(selectedFiles, function (file_) {
        uploadListQueque.push(file_);
    });
    //获得下一批文件列表
    //因为如果一次性上传太多文件的话，会导致浏览器卡住，因此要每隔一段时间重新再添加
    var getNextListItems = function () {
        var toListItemAr = [];
        //maxListItemsOneTime：每次最多同时上传的文件个数
        for (var index = 0; index < self.maxListItemsOneTime; index++) {
            if (uploadListQueque.length > 0) {
                toListItemAr.push(uploadListQueque.shift());
            } else {
                break;
            }
        }
        return toListItemAr;
    };
    self.startUploadQueque(getNextListItems());
    if (uploadListQueque.length > 0) {
        self.listUploadItemsTimer = setInterval(function () {
            var listAr = getNextListItems();
            if (listAr.length > 0) {
                self.startUploadQueque(listAr);
            } else {
                allListCallBack();
                clearInterval(self.listUploadItemsTimer);
            }
        }, self.listItemTimeInterval);
    } else {
        allListCallBack();
    }
    self.afterLoad();
    return false;
},
{% endcodeblock %}