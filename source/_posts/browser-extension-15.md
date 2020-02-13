---
title: 浏览器 extension 插件开发系列(15) -- chrome多文件上传（拖拽上传或者点击上传）
date: 2020-02-13 11:27:16
tags: 
- js
- 浏览器插件
- Chrome 插件
categories: 
- 前端相关
- 浏览器 extension 插件开发系列
---
## 前言
在这个插件的`Chrome`插件版本，是允许多文件上传的，包括拖拽上传和点击上传。(`Firefox` 和 `Safari` 不支持)。截图如下:

![png](1.png)
<!--more-->
![png](2.png)

![png](3.png)

## 上传
在 `popup` 页面的部分代码如下 `popup.js`:
```javascript
<input id="input_file" multiple="multiple" class="file-upload" type="file" name="file">
```
```javascript
// 文件上传
"change #input_file": "fileInputChangeHandle",
"dragenter #input_file": "fileDragEnterHandle",
"dragleave #input_file": "fileDragLeaveHandle",
"drop #input_file": "fileDragLeaveHandle",
```
```javascript
// 文件上传框改变事件, 包括拖拽上传(暂不支持文件夹上传)
fileInputChangeHandle: function (e) {
    var self = this;
    var did = self._dom.selectDevicePage.selectDeviceButton.attr("data-did");
    // 可以多选，所以要遍历
    _.each($(e.currentTarget)[0].files, function(inputFile){
        var itemDom = $(new Tpl('push_msg/file_item').render());
        var itemProgress = itemDom.find(".progress");
        var itemBar = itemDom.find(".progress-bar");
        var itemSpan = itemDom.find(".progress span");
        $(".upload-list").append(itemDom);
        var file = new self.Airdroid.File(inputFile);
        // 初始化上传条目
        // 开始上传
        file.upload(did)
                .progress(function (percentComplete) {
                    itemBar.css('width', percentComplete + '%');
                    itemSpan.text(percentComplete + '%');
                })
                .done(function () {
                    itemSpan.text(file.name + '上传成功');
                    itemProgress.removeClass("progress-striped");
                    //$.toast('Push Successly', 'success');
                })
                .always(function () {
                });
    });
},
// 拖拽移进去
fileDragEnterHandle: function(){
    $("#tab_pane_file").addClass("drag");
},
// 拖拽移进去
fileDragLeaveHandle: function(){
    $("#tab_pane_file").removeClass("drag");
},
```
而最主要的上传逻辑在 `file.js` 这个 model 上:
```javascript
// 文件上传操作
(function () {
    var File = function (file) {
        this.file = file;
        this.name = file.name;
        this.size = file.size;
    };

    File.prototype = {
        upload: function (deviceId) {
            var self = this,
                file = this.file, uploadUrl,tokenObj;
            var deviceObj = Airdroid.Account.getDeviceObjById(deviceId);
            var defer = $.Deferred();
            // 这边先模仿pc端的发送文件流程
            Airdroid.Server.getUploadToken({
                // 上传文件本地名称（filename用des加密）
                filename: Airdroid.Util.Des.encrypt(file.name),
                // 国籍信息（服务器根据这个字段判断是否是中国来选择存储地区）
                country: Airdroid.Account.getAccountCounty(),
                // 账号id
                account_id: Airdroid.Account.getAccountId(),
                // 文件发送端的key, 区别于pc端，即channelId
                from: Airdroid.PushManage.getChannelId(),
                // 发送端设备的push前缀(100,200)
                from_type: Airdroid.PushManage.DEVICE_PUSH_OPT.CHROME.key,
                // 文件接收端的key
                to: deviceObj.getId(),
                // 接收端设备的push前缀(100,200)
                to_type: Airdroid.PushManage.DEVICE_PUSH_OPT.ANDROID.key,
                // 文件大小（单位：byte）
                size: file.size,
                // 设备类型
                device_type: Airdroid.PushManage.DEVICE_PUSH_OPT.CHROME.type,
                // 文件hash值（md5加密）
                hash: ""
            }).done(function(resp){
                  try{
                      resp = JSON.parse(Airdroid.Util.Des.decrypt(resp));
                      switch(resp.cloud){
                          case "q":
                              // 如果是七牛
                              uploadUrl = "http://upload.qiniu.com";
                              tokenObj = {
                                  token: resp.data.token,
                                  key: resp.data.key
                              };
                              break;
                          case "a":
                              // 如果是亚马逊
                              uploadUrl = resp.data.form_action;
                              tokenObj = {
                                  key: resp.data.key,
                                  acl: resp.data.acl,
                                  success_action_redirect: resp.data.success_action_redirect,
                                  "X-Amz-Credential": resp.data.x_amz_credential,
                                  "X-Amz-Algorithm": resp.data.x_amz_algorithm,
                                  "X-Amz-Date": resp.data.x_amz_date,
                                  Policy: resp.data.policy,
                                  "X-Amz-Signature": resp.data.x_amz_signature
                              };
                              break;
                      }
                      // 接下来开始上传
                      xhrUploader(file, uploadUrl, tokenObj).done(function () {
                                    console.log('done');
                                    defer.resolve.apply(this, arguments);
                                })
                                .fail(function () {
                                    console.log('fail');
                                    defer.reject.apply(this, arguments);
                                })
                                .progress(function (percentComplete) {
                                    defer.notify.apply(this, arguments);
                                    console.log('uploaded:', percentComplete);
                                });
                  }catch(e){

                  }
            });
            return defer;
        }
    };
    // 文件上传helper对象
    var xhrUploader = function (file, uploadUrl, tokenObj) {
        if (!file) throw Error('file is required');
        var formData, k, defer, xhr;

        defer = $.Deferred();
        xhr = new XMLHttpRequest();

        formData = new FormData();
        for (k in tokenObj) {
            formData.append(k, tokenObj[k]);
        }
        formData.append('file', file);

        xhr.open("POST", uploadUrl, true);
        xhr.upload.onprogress = function (e) {
            var percentComplete;
            if (e.lengthComputable) {
                percentComplete = Math.floor((e.loaded / e.total) * 100);
                defer.notify(percentComplete);
            }
        };
        xhr.onreadystatechange = function () {
            if (xhr.status === 200) {
                defer.resolve.apply(this, arguments);
            }
        };
        xhr.onerror = function () {
            defer.reject.apply(this, arguments);
        };
        xhr.send(formData);
        return defer;
    };
    Airdroid.File = File;
})();
```
这样就可以实现文件上传了。同时 `manifest.json` 要对 `upload url` 有授权:

![png](4.png)

## 总结
到现在基本上一个比较完整的包含 `Chrome`， `Firefox`， `Safari`, 并且包含不少功能的浏览器扩展程序就完成了。当然当初在做这个的时候，遇到的问题比写在文章的肯定还要多。所以后面会针对各个浏览器遇到的问题，再做一些总结和收集。





