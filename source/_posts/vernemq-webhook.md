---
title: VerneMQ 通过设置 webhook 来获取每次连接的各个事件节点
date: 2018-07-15 23:20:49
tags: 
    - mqtt
    - vernemq
categories: webrtc相关
---
之前在一个 webrtc 的项目中，使用VerneMQ作为signal服务器进行通信的时候，发现每次连接的log都很难去跟进。而且 VerneMQ 的console.log 只能打出运行的一些基本状态，没法打印出每次连接的每一个事件节点的log，比如 pub 和 sub 之类的事件。
后面看来一下官方的文档 [webhookplugins](https://vernemq.com/docs/plugindevelopment/webhookplugins.html) 才发现可以通过设置webhook来得到每一次连接的各个事件节点。
## 操作流程：
首先是将 配置的这个打开：
{% codeblock lang:js %}
plugins.vmq_webhooks = on
{% endcodeblock %}
然后刚开始测试的时候，可以动态添加：
{% codeblock lang:js %}
[kbz@VM_16_13_centos ~]$ sudo vmq-admin webhooks register hook=on_deliver  endpoint="http://xxx.airdroid.com/mqtt/webhook"
Done
{% endcodeblock %}
注意，这样添加的好处，就是进程不需要重启，就可以生效。但是缺点就是 如果进程重启了，那么这些设置就都没有了。
如果要想持久化，就要添加到 vernemq.conf 这个文件里面。因为我们只是测试，所以先不需要添加到 vernemq.conf 文件中。
<!--more-->
就这样，我们直接添加了所有能够添加的 webhook， 查看目前所支持的 [webhook 事件](https://github.com/erlio/vernemq/blob/320192be1cde04d7656f42a12a4f63b0880437cb/apps/vmq_webhooks/src/vmq_webhooks_plugin.erl#L333)
这时候就查看：
{% codeblock lang:js %}
[kbz@VM_16_13_centos ~]$ sudo vmq-admin webhooks show
+------------------+------------------------------------------------+-------------+
|       hook       |                    endpoint                    |base64payload|
+------------------+------------------------------------------------+-------------+
|  on_unsubscribe  |http://xxx.airdroid.com/mqtt/webhook|    true     |
|   on_subscribe   |http://xxx.airdroid.com/mqtt/webhook|    true     |
|   on_register    |http://xxx.airdroid.com/mqtt/webhook|    true     |
|    on_publish    |http://xxx.airdroid.com/mqtt/webhook|    true     |
|on_offline_message|http://xxx.airdroid.com/mqtt/webhook|    true     |
|    on_deliver    |http://xxx.airdroid.com/mqtt/webhook|    true     |
| on_client_wakeup |http://xxx.airdroid.com/mqtt/webhook|    true     |
|on_client_offline |http://xxx.airdroid.com/mqtt/webhook|    true     |
|  on_client_gone  |http://xxx.airdroid.com/mqtt/webhook|    true     |
|auth_on_subscribe |http://xxx.airdroid.com/mqtt/webhook|    true     |
| auth_on_register |http://xxx.airdroid.com/mqtt/webhook|    true     |
| auth_on_publish  |http://xxx.airdroid.com/mqtt/webhook|    true     |
+------------------+------------------------------------------------+-------------+
{% endcodeblock %}
如果是要注销掉这个webhook的话，那么就是：
{% codeblock lang:js %}
$ vmq-admin webhooks deregister hook=auth_on_register endpoint="http://xxx.airdroid.com/mqtt/webhook"
{% endcodeblock %}
接下来就是 写对应的接口了。 go 代码如下：
{% codeblock lang:js %}
// 获取webhook
func (c *Mqtt) Webhook() revel.Result {
   res := WebHookRes{}
   res.Result = "ok"
   // 获取头部
   hookName := c.Request.Header.Get("vernemq-hook")
   //log.Info("vernemq hook:", hookName)
   // 获取数据
   params, err := utils.GetJsonParam(c.Request)
   if err != nil {
      log.Error("vernemq hook: json err", err)
      return utils.EchoResult(c.Controller, res)
   }

   utils.SetNotExist(params, "mountpoint", "")
   utils.SetNotExist(params, "clientId", "")
   utils.SetNotExist(params, "username", "")
   utils.SetNotExist(params, "peerAddr", "")
   utils.SetNotExist(params, "peerPort", "")
   utils.SetNotExist(params, "qos", "")
   utils.SetNotExist(params, "topic", "")
   utils.SetNotExist(params, "topics", "")
   utils.SetNotExist(params, "payload", "")
   utils.SetNotExist(params, "retain", "")
   utils.SetNotExist(params, "password", "")
   utils.SetNotExist(params, "cleanSession", "")

   if params["payload"] != "" {
      params["payload"], err = crypto.Base64DecodeToStr(params["payload"])
      if err != nil {
         params["payload"] = ""
         log.Error("payload decode err", err)
      }
      // 接下来判断是否是一个json 格式的
      tmp := make(map[string]interface{})
      err := json.Unmarshal([]byte(params["payload"]), &tmp)
      if err != nil {
         params["payload"] = ""
         log.Error("payload no json string", err)
      }
      if len(params["payload"]) > 100 {
         params["payload"] = params["payload"][:100]
      }
   }

   if params["topic"] != "" {
      if len(params["topic"]) > 100 {
         params["topic"] = params["topic"][:100]
      }
   }

   if params["topics"] != "" {
      if len(params["topics"]) > 100 {
         params["topics"] = params["topics"][:100]
      }
   }

   data := models.MqttWebHookData{
      HookName:     hookName,
      CreateDate:   utils.GetNowDatetime(),
      MountPoint:   params["mountpoint"],
      ClientId:     params["clientId"],
      Username:     params["username"],
      PeerAddr:     params["peerAddr"],
      PeerPort:     params["peerPort"],
      Qos:          params["qos"],
      CleanSession: params["cleanSession"],
      Topic:        params["topic"],
      Topics:       params["topics"],
      Payload:      params["payload"],
      Retain:       params["retain"],
      Ip:           util.ClientIp(c.Request.Request),
   }
   log.Infof("vernemq hook(%v): info:%v", hookName, params)
   //log.Infof("vernemq hook(%v): data:%v", hookName, data)
   // 插入log
   models.MqttWebHookModel.InsertMqttWebHookLogs(data)
   return utils.EchoResult(c.Controller, res)
}
{% endcodeblock %}
全部返回 **ok**, 数据很简单就是取出来， 然后入库。
测试程序如下：
{% codeblock lang:js %}
var mqtt = require('mqtt');
// var client  = mqtt.connect('mqtt://test.mosquitto.org');
var clientId = '1528366636';
var username = '100';
var pwd = 'xxx';
var client  = mqtt.connect('tcp://test-xxx.airdroid.com:1883',{
    clientId: clientId,
    username: username,
    password: pwd
});

client.on('connect', function () {
    console.log("connected");
    client.subscribe('say_name');
    setTimeout(function(){
        client.publish('say_name', 'hello')
    },10);
});

client.on('message', function (topic, message) {
    // message is Buffer
    console.log(topic);
    console.log(message.toString());
    // client.end()
});

console.log("start");
{% endcodeblock %}
而且通过看 log， 也可以看到：
{% codeblock lang:js %}
[Info] [2018-07-11 03:38:04] [vernemq hook(auth_on_register): info:map[username:100 qos: topics: peerPort:53097 clientId:1528366636 password:xxx cleanSession:1 peerAddr:125.77.202.250 topic: payload: retain: mountpoint:]]
[Info] [2018-07-11 03:38:04] [vernemq hook(on_client_wakeup): info:map[topics: payload: mountpoint: qos: peerAddr: peerPort: topic: retain: clientId:1528366636 username:]]
[Info] [2018-07-11 03:38:04] [vernemq hook(on_register): info:map[topic: topics: payload: retain: clientId:1528366636 peerPort:53097 mountpoint: username:100 peerAddr:125.77.202.250 qos:]]
[Info] [2018-07-11 03:38:04] [vernemq hook(auth_on_subscribe): info:map[peerAddr: qos: topic: payload: retain: clientId:1528366636 topics:[{&#34;qos&#34;:0,&#34;topic&#34;:&#34;say_name&#34;}] mountpoint: username:100 peerPort:]]
[Info] [2018-07-11 03:38:04] [vernemq hook(on_subscribe): info:map[username:100 peerAddr: qos: retain: topics:[{&#34;qos&#34;:0,&#34;topic&#34;:&#34;say_name&#34;}] clientId:1528366636 peerPort: topic: payload: mountpoint:]]
[Info] [2018-07-11 03:38:04] [vernemq hook(auth_on_publish): info:map[qos:0 topics: payload:hello retain:0 username:100 mountpoint: clientId:1528366636 topic:say_name peerAddr: peerPort:]]
[Info] [2018-07-11 03:38:04] [vernemq hook(on_publish): info:map[topics: clientId:1528366636 qos:0 payload:hello retain:0 peerPort: username:100 mountpoint: topic:say_name peerAddr:]]
[Info] [2018-07-11 03:38:04] [vernemq hook(on_deliver): info:map[mountpoint: peerAddr: peerPort: retain: payload:gogogo username:100 clientId:1528366636 topic:say_name qos: topics:]]
[Info] [2018-07-11 03:38:04] [vernemq hook(on_deliver): info:map[topic:say_name payload:hello username:100 clientId:1528366636 qos: topics: retain: mountpoint: peerAddr: peerPort:]]
[Info] [2018-07-11 03:38:07] [vernemq hook(on_client_gone): info:map[username: peerAddr: peerPort: qos: topics: mountpoint: clientId:1528366636 topic: payload: retain:]]
{% endcodeblock %}
而且入库也是可以查到：
![1](vernemq-webhook/1.png)
这时候就可以从库里面查到某一个client连接的所有的触发的事件节点顺序。
## 添加到配置文件的话
{% codeblock lang:js %}
plugins.vmq_webhooks = on
vmq_webhooks.mywebhook1.hook = on_unsubscribe
vmq_webhooks.mywebhook1.endpoint = http://xxx.airdroid.com/mqtt/webhook
vmq_webhooks.mywebhook2.hook = on_subscribe
vmq_webhooks.mywebhook2.endpoint = http://xxx.airdroid.com/mqtt/webhook
vmq_webhooks.mywebhook3.hook = on_register
vmq_webhooks.mywebhook3.endpoint = http://xxx.airdroid.com/mqtt/webhook
vmq_webhooks.mywebhook4.hook = on_publish
vmq_webhooks.mywebhook4.endpoint = http://xxx.airdroid.com/mqtt/webhook
vmq_webhooks.mywebhook5.hook = on_offline_message
vmq_webhooks.mywebhook5.endpoint = http://xxx.airdroid.com/mqtt/webhook
vmq_webhooks.mywebhook6.hook = on_deliver
vmq_webhooks.mywebhook6.endpoint = http://xxx.airdroid.com/mqtt/webhook
vmq_webhooks.mywebhook7.hook = on_client_wakeup
vmq_webhooks.mywebhook7.endpoint = http://xxx.airdroid.com/mqtt/webhook
vmq_webhooks.mywebhook8.hook = on_client_offline
vmq_webhooks.mywebhook8.endpoint = http://xxx.airdroid.com/mqtt/webhook
vmq_webhooks.mywebhook9.hook = on_client_gone
vmq_webhooks.mywebhook9.endpoint = http://xxx.airdroid.com/mqtt/webhook
vmq_webhooks.mywebhook10.hook = auth_on_subscribe
vmq_webhooks.mywebhook10.endpoint = http://xxx.airdroid.com/mqtt/webhook
vmq_webhooks.mywebhook11.hook = auth_on_register
vmq_webhooks.mywebhook11.endpoint = http://xxx.airdroid.com/mqtt/webhook
vmq_webhooks.mywebhook12.hook = auth_on_publish
vmq_webhooks.mywebhook12.endpoint = http://xxx.airdroid.com/mqtt/webhook
{% endcodeblock %}
然后重启一下即可：
{% codeblock lang:js %}
[kbz@VM_0_6_centos ~]$ sudo service vernemq restart
Restarting vernemq (via systemctl):                        [  OK  ]
{% endcodeblock %}
## 事件节点的含义
这些事件节点，主要涉及到三个事件，register， publish， subscribe
### [session 的生命周期](https://vernemq.com/docs/plugindevelopment/sessionlifecycle.html)
![1](vernemq-webhook/2.png)
### [publish 事件流](https://vernemq.com/docs/plugindevelopment/publishflow.html)
![1](vernemq-webhook/3.png)
### [subscribe 事件流](https://vernemq.com/docs/plugindevelopment/subscribeflow.html)
![1](vernemq-webhook/4.png)
















