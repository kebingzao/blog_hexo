---
title: web站点 开放第三方API流程(7) - 嵌入包含广告的第三方iframe
date: 2018-08-01 20:25:04
tags: 
    - js
    - open api
categories: 前端相关
---
我们都知道，google的广告在 ip地址下其实是显示不出来的，那么我们怎么在ip地址下的页面显示google的广告呢？
很简单，把google的广告部署在线上，然后作为一个iframe嵌入到你的ip地址的页面去。这样就可以正常显示了。
这时候所有的配置项都可以作为参数放到url上。 具体实作如下：
以下是子页面的所有的代码：
<!--more-->
**index.html**:
{% codeblock lang:html %}
<!DOCTYPE html>
<html lang="en-us">
    <head>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
    <title>recommend</title>
    <script type="text/javascript" src="../jschannel.js"></script>
    <script type="text/javascript" src="openMethodList.js"></script>
    <script type="text/javascript" src="../airdroid.js"></script>
    <script type="text/javascript" src="openApi.js"></script>
    <style>
        body {
            margin:0;
            padding:0;
            overflow: hidden;
        }
    </style>
    </head>
    <body>
        <script type='text/javascript'>
            var getUrlParam = function (name) {
                name = name.replace(/[\[]/, "\\\[").replace(/[\]]/, "\\\]");
                var regexS = "[\\?&]" + name + "=([^&#]*)";
                var regex = new RegExp(regexS);
                var results = regex.exec(window.location.href);
                if (results == null) {
                    return "";
                } else {
                    return results[1];
                }
            };
            var recommendObj = getUrlParam("setting");
            if(recommendObj){
                recommendObj = JSON.parse(decodeURIComponent(recommendObj));
                var dfpSetting = recommendObj.options;
                var googletag = googletag || {};
                googletag.cmd = googletag.cmd || [];
                (function() {
                    var gads = document.createElement('script');
                    gads.async = true;
                    gads.type = 'text/javascript';
                    var useSSL = 'https:' == document.location.protocol;
                    gads.src = (useSSL ? 'https:' : 'http:') +
                    '//www.googletagservices.com/tag/js/gpt.js';
                    var node = document.getElementsByTagName('script')[0];
                    node.parentNode.insertBefore(gads, node);
                })();
                googletag.cmd.push(function() {
                    googletag.defineSlot('/xxxxxx/' + dfpSetting.name, dfpSetting.size , dfpSetting.divId).addService(googletag.pubads());
                    googletag.pubads().enableSingleRequest();
                    googletag.pubads().collapseEmptyDivs();
                    googletag.enableServices();
                    var reLoadCount = 0;
                    // check recommend render
                    googletag.pubads().addEventListener('slotRenderEnded', function(event) {
                        console.log('Slot has been rendered:');
                        if (event.isEmpty) {
                            console.log("Slot render empty");
                            if(reLoadCount < dfpSetting.reLoadMaxCount){
                                setTimeout(function () {
                                    reLoadCount += 1;
                                    console.log("Slot reload");
                                    googletag.pubads().refresh();
                                }, 2000);
                            }else{
                                console.log("Slot Empty");
                                AirDroidHelper.Service.removeRecommendDom();
                            }
                        }else{
                            AirDroidHelper.Service.showRecommendDom();
                        }
                    });
                });
                document.body.style.width = dfpSetting.css.width;
                document.body.style.height = dfpSetting.css.height;
                var divDom = document.createElement("div");
                divDom.id = dfpSetting.divId;
                divDom.innerHTML = "";
                divDom.style.width = dfpSetting.css.width;
                divDom.style.height = dfpSetting.css.height;
                document.body.appendChild(divDom);
            }
            if(AirDroidHelper){
                AirDroidHelper.init({accessKey:"xxxx"},function(){
                    googletag.cmd.push(function() {
                        googletag.display(dfpSetting.divId);
                    });
                },function(error,msg){
                    console.debug("error:" + error + " msg : " + msg);
                })
            }
        </script>
    </body>
</html>
{% endcodeblock %}
**openMethodList.js**:
{% codeblock lang:js %}
(function () {
    window.AirdroidRuntime = {};
    AirdroidRuntime.getMethodList = function() {
        return {
            // 服务端增加接口，只需要在列表里添加接口名就行。
             SERVER_METHODS : [
                "removeRecommendDom",
                "showRecommendDom"
             ]
        }
    }
})();
{% endcodeblock %}
这边只增加了两个方法，一个是告诉父页面，广告已经显示出来了，一个是告诉父页面，可以移除广告了
接下来封装后的 **openApi.js**:
{% codeblock lang:js %}
(function(window){
    // 初始化 API 对象
    var airdroidApi = new window.AirdroidApi();
    // 这里每增加一个方法，都要去 airdroid.js 的 方法列表里面添加
    // 因为该通信是异步的，因此每次的结果都要回调
    window.AirDroidHelper = {
        isFunction:function(obj){
            return airdroidApi.isFunction(obj);
        },
        // 初始化，传入accesskey
        init:function(params,successCb,failCb){
            var self = this;
            airdroidApi.init({
                params : {
                    accessKey: params.accessKey
                },
                // 设置超时时间，因为连接的超时时间是36秒，因此这边要设的更长
                success: function(result){
                    if(self.isFunction(successCb)){
                        successCb();
                    }
                },
                error: function(errorType,msg){
                    console.debug("error：" + errorType + ",msg：" + msg);
                    if(self.isFunction(failCb)){
                        failCb(errorType,msg);
                    }
                }
            })
        },
        // 服务请求管理接口
        Service: {
            // 移除recommend dom
            removeRecommendDom: function () {
                return airdroidApi.removeRecommendDom({
                    success: function(){}
                });
            },
            // 显示recommend dom
            showRecommendDom: function () {
                return airdroidApi.showRecommendDom({
                    success: function(){}
                });
            }
        }
    };
})(window);
console.debug('Airdroid Open Api  ready !!!');
{% endcodeblock %}
**airdroid.js** 这个js是不会变的，跟之前一样。
父页面controlMap 要给他加一个：
{% codeblock lang:js %}
// dfp recommend 接口
"xxxx": {
    token: "xxxx",
    // 所允许的方法
    openMap: {
      // 服务请求
      SERVER_METHODS:[
        "removeRecommendDom",
        "showRecommendDom"
      ]
    }
},
{% endcodeblock %}
同时在 父页面的 **apiprovider.js** 要提供这两个方法的调用：
{% codeblock lang:js %}
removeRecommendDom: function(trans, params){
    do something
},
showRecommendDom: function(trans, params){
    do something
},
{% endcodeblock %}

这样就可以了

---
完整系列：
{% post_link web-open-api-1 %}
{% post_link web-open-api-2 %}
{% post_link web-open-api-3 %}
{% post_link web-open-api-4 %}
{% post_link web-open-api-5 %}
{% post_link web-open-api-6 %}
{% post_link web-open-api-7 %}