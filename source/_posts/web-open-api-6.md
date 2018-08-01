---
title: web站点 开放第三方API流程(6) - 父页面的sdk详解
date: 2018-08-01 20:11:20
tags: 
    - js
    - open api
categories: 前端相关
---
通过 {% post_link web-open-api-5 %} 可以知道 demo 中 sdk 的运行逻辑了。本节主要是完整的列一下 父页面的 sdk的 js，该讲的，前面就讲完了
由于开放的方法比较多，我这边就不所有的代码都列出来了。只简单列几个：
<!--more-->
父页面的 **apiprovider.js**:
{% codeblock lang:js %}
// 向目标 window 提供 api 接口方法，这些接口可以给第三方开发者使用。
// configs.window 为第三方应用的 window，如 iframe.contentWindow
// 依赖 [underscore.js, jsChannel]
(function () {
    var AirService = Airdroid.Service;
    var AirUtil = Airdroid.Util;
    // 必须包含 configs.window，指定能调用 api 的窗体。
    window.ApiProvider = function (iframeId,winObj) {
        var self = this;
        // 允许的token
        self._accessToken = null;
        // 允许开放的方法
        self._accessMothod = ['registerPushHandle'];
        self.chan = Channel.build({
            window: document.getElementById(iframeId).contentWindow,
            origin: "*",
            scope: "AirDroid_Web_OpenAPI",
            onReady:function(){
                self.init();
                // 将通道绑定到具体的窗体对象,使得既可以让子窗体监听父窗体的窗体变化，也可以让子窗体调用父窗体的渲染形式
                winObj && winObj.hookChan(self.chan);
            }
        });
        // 这边控制一些key和所能调用的方法
        // 主要是key和token和所能用的方法
        // 每一个方法都会带上token，这样才允许返回
        // 每次调用开发的接口，都必须带上init返回的token，并且只能调用所被允许的方法，目前在openjs做判断
        // todo 而且为了更安全，token 和 允许的请求方法的值，应该由服务端来返回, 这边为了方便展示，直接硬编码在这里
        // 不然如果直接修改openjs里面的方法列表，那么就可以调用这里的所有开发方法
        self.controlMap = {
            。。。省略其他的第三方服务配置
            // demo 接口
            "c81e728d9d4c2f636f067f89cc14862c": {
                token: "37886cb7eb532415f9b40e2bf58d7fd6",
                // 所允许的方法
                openMap: {
                    // 服务请求
                    SERVER_METHODS:[
                        "sayHello",
                        "getDeviceOpt",
                        "reconnectPhone"
                    ],
                    // 系统方法
                    SYSTEM_METHODS: [
                        "system.alert",
                        "system.confirm"
                    ]
                }
            }
        }
    };
    ApiProvider.prototype = {
        // 初始化
        init:function(){
            var self = this;
            // 主要是跟一些服务,数据相关的接口
            // 每个方法的第一个参数是 jschannel 提供的，具体请查看 jschannel 文档的 Transactions 部分。
            self.SERVER_APIS = {
                // 获取权限初始化，要得到对应的tokenKey
                init: function(trans,params){
                    trans.delayReturn(true);
                    // 获取accessKey
                    var accessKey = params.accessKey;
                    var mapItem = self.controlMap[accessKey];
                    if(mapItem){
                        self._accessToken = mapItem.token;
                        // 所允许被调用的方法，目前只对服务方法和系统方法做限制，其他先不限制
                        self._accessMothod = self._accessMothod.concat(mapItem.openMap.SERVER_METHODS.concat(mapItem.openMap.SYSTEM_METHODS || []));
                        trans.complete(self.getAccessToken());
                    }else{
                        throw {error: "forbidden", message: "Illegal access key"};
                    }
                },
                // 注册推送事件(这个事件很重要，是第三方消息推送的基础)
                registerPushHandle: function(trans,params){
                    var deviceId = params.deviceId;
                    var pushArr = params.pushArr;
                    // 加入到白名单中
                    Airdroid.MessageManage.addWhiteListListener(pushArr);
                    // 监听事件
                    _.each(pushArr,function(item){
                        // 先去掉旧的，再绑定新的事件
                        Airdroid.MessageManage.removeListener(item,undefined,"open_api_" + deviceId);
                        Airdroid.MessageManage.addListener(item,function(data){
                            if(!(data.data && data.data.deviceId)){
                                data.data = data.data || {};
                                data.data.deviceId = Airdroid.DeviceManage.getCurrentDeviceId();
                            }
                            if(data.data.deviceId == deviceId){
                                // 如果是该设备，就call
                                self.chan.call({
                                    method: item,
                                    params: data.data,
                                    success: function () {
                                        return true;
                                    },
                                    error: function () {
                                        return false;
                                    }
                                });
                            }
                        },"open_api_" + deviceId);
                    })
                },
                // 重新连接手机
                reconnectPhone: function(trans,params){
                    trans.delayReturn(true);
                    var deviceId = params.deviceId;
                    var deviceObj = Airdroid.DeviceManage.getDeviceObjById(deviceId);
                    // 获取返回结果
                    var getResultData = function(isConnect){
                        return {
                            result: isConnect ? 1 : 0,
                            isRemote : deviceObj.isRemote()
                        }
                    };
                    Airdroid.CenterManage.reconnectPhone(deviceObj,function(){
                        // 设备连接成功
                        trans.complete(getResultData(true));
                    },function(){
                        // 设备连接失败
                        trans.complete(getResultData(false));
                    });
                },
                。。。省略一堆开放的方法
                // lite 下 dfp recommend 传过来，移除recommend dom
                removeRecommendDom: function(trans, params){
                    Airdroid.RecommendManage.fireEvent(Airdroid.RecommendManage.EVENT_TYPE.remove_dom, {});
                },
                showRecommendDom: function(trans, params){
                    Airdroid.RecommendManage.fireEvent(Airdroid.RecommendManage.EVENT_TYPE.show_dom, {});
                },
                //====================widgetDemo 事例接口 start =======================
                sayHello: function(trans, params){
                    trans.delayReturn(true);
                    Airdroid.Util.alert(params.msg);
                    trans.complete();
                }
                //====================widgetDemo 事例接口 end =======================
            };
            // 主要是跟一些系统方法相关的开放接口
            self.SYSTEM_APIS = {
                // alert提示框
                "system.alert" : function(trans,params){
                    //创建提示框
                    var pp = $.extend({},params);
                    //重新初始化，然后触发监听事件
                    pp.yesCallback = function(){
                        //执行字符串函数
                        //监听回调函数
                        if (params.yesCallback) {
                            // 获取回调的参数
                            if(_.isObject(params.param)){
                                var newParam = params.param;
                            }else{
                                var newParam = {};
                            }
                            //触发回调函数
                            self.chan.call({
                                method:params.yesCallback,
                                params:newParam,
                                success:function () {
                                    //alert只执行一次，执行完之后应该要注销掉该监听方法
                                    self.chan.call({
                                        method:"removecbListen",
                                        params:params.yesCallback,
                                        success:function () {
                                            return true;
                                        },
                                        error:function () {
                                            return false;
                                        }
                                    });
                                    return true;
                                },
                                error:function () {
                                    return false;
                                }
                            });
                        }
                    };
                    AirUtil.alert(pp.msg,pp.yesCallback);
                    return true;
                },
                // confirm确认框
                "system.confirm" : function(trans,params){
                    var pp = $.extend({},params);
                    //重新初始化，然后触发监听事件
                    if(_.isObject(params.param)){
                        var newParams = params.param;
                    }else{
                        var newParams = {};
                    }
                    // 按 ok的回调
                    pp.yesCallback = function(){
                        //执行字符串函数
                        //监听回调函数
                        if (params.yesCallback) {
                            //触发回调函数
                            self.chan.call({
                                method:params.yesCallback,
                                params:newParams,
                                success:function () {
                                    //alert只执行一次，执行完之后应该要注销掉该监听方法
                                    self.chan.call({
                                        method:"removecbListen",
                                        params:params.yesCallback,
                                        success:function () {
                                            return true;
                                        },
                                        error:function () {
                                            return false;
                                        }
                                    });
                                    return true;
                                },
                                error:function () {
                                    return false;
                                }
                            });
                        }
                    };
                    // 按 cancel的回调
                    pp.cancelCallback = function(){
                        //执行字符串函数
                        //监听回调函数
                        if (params.cancelCallback) {
                            //触发回调函数
                            self.chan.call({
                                method:params.cancelCallback,
                                params:newParams,
                                success:function () {
                                    //alert只执行一次，执行完之后应该要注销掉该监听方法
                                    self.chan.call({
                                        method:"removecbListen",
                                        params:params.cancelCallback,
                                        success:function () {
                                            return true;
                                        },
                                        error:function () {
                                            return false;
                                        }
                                    });
                                    return true;
                                },
                                error:function () {
                                    return false;
                                }
                            });
                        }
                    };
                    AirUtil.confirm(pp.msg,pp.yesCallback,pp.cancelCallback);
                    return true;
                },
                // prompt 填充框
                "system.prompt" : function(trans,params){
                    var pp = $.extend({},params);
                    //重新初始化，然后触发监听事件
                    if(_.isObject(params.param)){
                        var newParams = params.param;
                    }else{
                        var newParams = {};
                    }
                    // 按 ok的回调
                    pp.yesCallback = function(){
                        //执行字符串函数
                        //获取输入值
                        var filedValue = arguments[0];
                        if (params.yesCallback) {
                            //触发回调函数
                            self.chan.call({
                                method:params.yesCallback,
                                params:{data:filedValue},
                                success:function () {
                                    self.chan.call({
                                        method:"removecbListen",
                                        params:params.yesCallback,
                                        success:function () {
                                            return true;
                                        },
                                        error:function () {
                                            return false;
                                        }
                                    });
                                    return true;
                                },
                                error:function () {
                                    return false;
                                }
                            });
                        }
                    };
                    // 按 cancel的回调
                    pp.cancelCallback = function(){
                        //执行字符串函数
                        //监听回调函数
                        if (params.cancelCallback) {
                            //触发回调函数
                            self.chan.call({
                                method:params.cancelCallback,
                                params:newParams,
                                success:function () {
                                    self.chan.call({
                                        method:"removecbListen",
                                        params:params.cancelCallback,
                                        success:function () {
                                            return true;
                                        },
                                        error:function () {
                                            return false;
                                        }
                                    });
                                    return true;
                                },
                                error:function () {
                                    return false;
                                }
                            });
                        }
                    };
                    // 验证参数回调
                    pp.validateCallback = function(value,callback){
                        //执行字符串函数
                        //监听回调函数
                        if (params.validateCallback) {
                            //触发回调函数
                            self.chan.call({
                                method:params.validateCallback,
                                // 将输入的值传过去
                                params:{data:value},
                                success:function (result) {
                                    //把返回值通过callback执行
                                    if(callback(result)){
                                        // 如果返回true，就释放该事件
                                        self.chan.call({
                                            method:"removecbListen",
                                            params:params.validateCallback,
                                            success:function () {
                                                return true;
                                            },
                                            error:function () {
                                                return false;
                                            }
                                        });
                                    }
                                    return true;
                                },
                                error:function () {
                                    return false;
                                }
                            });
                        }
                    };
                    AirUtil.prompt(pp.type,pp.title,pp.fieldName,pp.defaultValue,pp.yesCallback,pp.cancelCallback,pp.validateCallback);
                    return true;
                },
                // 显示全局蒙板
                "system.showBodyMask":function(tran,params){
                    AirUtil.showBodyMask();
                },
                // 隐藏全局蒙板
                "system.removeBodyMask":function(tran,params){
                    AirUtil.removeBodyMask();
                },
            };
            self.ALL_APIS = $.extend(true,{}, self.SERVER_APIS, self.SYSTEM_APIS);
            self.provideAll();
        },
        // 获取access token
        getAccessToken:function(){
            return this._accessToken;
        },
        // 绑定所有的接口
        provideAll: function () {
            var self = this;
            // 先绑定自己定义的开放接口
            self.provide(_.keys(self.ALL_APIS));
        },
        provide: function (apis) {
            var self = this;
            apis = apis || [];
            // todo 这边不绑定window系列的方法，window系列的方法会在AppWindow里面绑定，这里只绑定 service 和 system 系列的方法
            _.each(apis, function (apiName) {
                var isSupportedApi = _.has(self.ALL_APIS, apiName);
                if (isSupportedApi) {
                    self.chan.bind(apiName, function(tran,params){
                        // 除了init方法之后，每一次请求都会带accessToken
                        var token = params.accessToken;
                        // 如果token是对的，并且是在允许的方法之内，那么就允许
                        // 如果是init 和 registerPushHandle 这两个方法，也允许
                        if((token && self.getAccessToken() === token && self._accessMothod.indexOf(apiName) > -1)
                                || apiName === "init"){
                            // 是被允许开放的方法
                            return self.ALL_APIS[apiName].apply(this,arguments);
                        }else{
                            // 返回禁止信息
                            if(self.getAccessToken() !== token){
                                throw {error: "forbidden", message: 'token error'};
                            }else if(self._accessMothod.indexOf(apiName) < 0){
                                throw {error: "forbidden", message: apiName +' method forbidden'};
                            }
                        }
                    });
                }
            });
        },
        // 获取通道对象
        getChannelObj: function(){
            return this.chan;
        }
    };
}());
{% endcodeblock %}
从代码可以看到即所有的 server 系列和 system 系列的方法，都要通过 providerAll 方法重新包装一遍，这样才能在方法的入口处，对这两个系列的方法进行accessToken 校验和 方法授权允许校验。









