---
title: webrtc 的 signal 服务器 VerneMQ 的集群设置
date: 2018-06-07 23:20:49
tags: 
    - mqtt
    - vernemq
categories: webrtc相关
---
通过{% post_link vernemq-verify %} 我们已经将VerneMQ 的权限校验设置成redis校验了。我们知道VerneMQ作为一个开源的基于MQTT的broker服务，它是支持分布式集群的。也就是说客户端可以连接到任何的节点，并且从其他的节点中接收消息。
举个例子：有两台服务器A和B，这两台服务器构成一个集群，然后 client1 连上A服务器，并订阅(sub)这个主题"say"， 这时候client2 连上B服务器，并给主题"say" 发布（pub）了一个消息，这时候client1就会收到这一条消息了。
官网的文档说的还是挺详细的 https://vernemq.com/docs/clustering/ ，我这边主要是根据我的理解简单说下几个要注意的点。
<!--more-->
### 几个要注意的点
####  设置cookie的值
所有的集群节点都应该设置相同的cookie值，这个是为了安全。就是在 vernemq.conf 的 distributed_cookie：
{% codeblock lang:js %}
## Cookie for distributed node communication.  All nodes in the
## same cluster should use the same cookie or they will not be able to
## communicate.
## IMPORTANT!!! SET the cookie to a private value! DO NOT LEAVE AT DEFAULT!
##
## Default: vmq
##
## Acceptable values:
##   - text
distributed_cookie = vmq
{% endcodeblock %}
默认的cookie是 vmq，这个后面要改成你自己特有的。

#### 设置每个节点的名称
集群还涉及到另一个方面，就是每个node的名称， 在集群中，所有的节点的名称都应该是唯一的。
这个设置也是在 vernemq.conf 中的 nodename 来设置：
{% codeblock lang:js %}
## Name of the Erlang node
##
## Default: VerneMQ@127.0.0.1
##
## Acceptable values:
##   - text
nodename = VerneMQ@127.0.0.1
{% endcodeblock %}
默认是 VerneMQ@127.0.0.1，这个格式是固定的，前面是一个string类型的名称，用@符号隔开，后面是一个ip地址，这个ip地址也不能随便乱写，一般是这个节点的外网IP，如果是同一个内网环境下(或者是同一个VPC环境下)，那么就要用内网ip。

#### 加入集群的指令：
{% codeblock lang:js %}
vmq-admin cluster join discovery-node=<OtherClusterNode>
{% endcodeblock %}
最后面的参数部分就是 节点的名称了

#### 离开集群的指令：
{% codeblock lang:js %}
vmq-admin cluster leave node=<NodeThatShouldGo> (only the first step!)
{% endcodeblock %}
至于离开集群的话，这边分为几个情况：
#####  这个node当前正在使用
那么在这个node在停止使用的时候，要把他里面的队列迁移到其他的节点。 所以这边要分为几个步骤:
首先先执行：
{% codeblock lang:js %}
vmq-admin cluster leave node=<NodeThatShouldGo>
{% endcodeblock %}
停止节点的mqtt订阅，让他不会再去接收新连接，但他也不会中断现有的连接，现在他还不会从集群中离开（相当于隐居幕后），现有用户还是可以继续发布和接收消息。这个是为了有一个宽限期，为了让现有用户不会受到影响， 这个时间你可以自己判断，可以是一天，也可以是一个小时，反正就是要等这些现有用户的连接都释放了。
接着执行：
{% codeblock lang:js %}
vmq-admin cluster leave node=<NodeThatShouldGo> -k
{% endcodeblock %}
到这一步，就会把这台node的所有的mqtt订阅器全部断开了，就是不订阅了， 当然如果一开始就不想要这些连接了，也可以直接执行这一步，然后接下来，就会将这一台的剩余的队列迁移到其他的节点node，并重新建立起mqtt订阅器。 不过可能会漏掉一些离线队列。
最后执行：
{% codeblock lang:js %}
vmq-admin cluster leave node=<NodeThatShouldGo> -k -i 5 -t 120
{% endcodeblock %}
这时候vernemq就会抛送一个异常，用于一个可配置的超时后的剩余离线队列，默认时间为60s，这个是可以配置的，就是 -t， 同时集群离开的指令会在console.log 中打印出来，上面的指令就是将超时时间，设置为 120s，和 每 5s 打印一次日志 （这时候用 tail -f 实时跟进迁移过程），过了这个超时时间之后，vernemq就会强制将剩下的离线队列迁移到其他的集群节点。做完之后，这个节点就正式离开了集群。

##### 假设这个node已经停止使用了：
指令跟上面一样，不过有一点要注意的是，这时候这个节点上面的离线队列（QoS1 和 QoS2 都会用于接收离线消息），就不能被迁移了，只能被舍弃了。
离线队列就是用来存储离线消息的，至于这个消息能不能被离线存储是由client在publish 消息的时候所指定 Qos的级别来解决。
服务质量（QoS）级别是一种关于发送者和接收者之间信息投递的保证协议。MQTT中有三种QoS级别：
- 至多一次（0）
- 至少一次（1）
- 只有一次（2）

也就是说 Qos1 和 Qos2 都是要有回执的。 也就是说一旦 client 发送了一个 qos 为2的消息。 这时候如果接受者（离线设备）不在的话，那么broker（对于本例就是 vernemq）就会将消息存储起来。
一直到接受者上线之后，再发给他。这个就是离线消息队列。

#### 显示当前集群的情况
{% codeblock lang:js %}
vmq-admin cluster show
{% endcodeblock %}

#### 为了便于防火墙设置，可以设置erlang 节点通信的端口范围：
比如：
erlang.distribution.port_range.minimum = 6000
erlang.distribution.port_range.maximum = 7999
上面只是设置订阅和发布的端口。 如果真的用来做分布式分发订阅的话，那么下面这个就得设置：
listener.vmq.clustering = 0.0.0.0:44053
这边也要设置为外网ip。 这个端口不需要每个节点都要设置成一样， 节点会相互嗅探。

---
### 实践
准备来两台服务器，一台在中国，一台在美西。(以下外网ip是随便写的，不是真实ip)
中国外网ip: 193.112.122.122
中国内网ip: 172.16.16.13
美西外网ip: 49.51.51.51
美西内网ip: 172.26.16.13
经过来一番折腾： 终于搭起来了，主要修改的配置如下(中国区)：
{% codeblock lang:js %}
listener.vmq.clustering = 172.16.16.13:44053
nodename = VerneMQ@172.16.16.13
distributed_cookie = vmq_xxx
queue_deliver_mode = balance
erlang.distribution.port_range.minimum = 6000
erlang.distribution.port_range.maximum = 7999
{% endcodeblock %}
美西这一台修改的配置基本上一样，就是内网地址换成它自己的。
这个是最后成功的配置，其实刚开始不是这么配置的，刚开始没有改这个 listener.vmq.clustering， 而是直接将 nodename 改成外网地址的 VerneMQ@193.112.122.122，结果发现连启动都启动不起来。后面发现不能用外网地址，因为我们两台虽然放在不同的区，一个中国区，一个美西区，但是由于我们有用了一个 VPC 对等连接的服务([腾讯云的服务](https://cloud.tencent.com/document/product/215/5000)) ，因此虽然跨区，但是通过这个服务，这两台还是在处于同一个VPC下，因此这时候外网ip就变得不可靠，因此这边要改成内网ip。
当改为内网地址之后
nodename = VerneMQ@172.16.16.13
nodename = VerneMQ@172.26.16.13
就可以启动了。但是当我在将 26 这一台加入到 16 这一台的集群的时候，
{% codeblock lang:js %}
vmq-admin cluster join discovery-node=VerneMQ@172.26.16.13
Done
{% endcodeblock %}
发现虽然成功了， 但是当我在查看集群状态的时候：从 16 这一台来查看的时候，发现26这一台的状态是 false的
{% codeblock lang:js %}
[kbz@VM_16_13_centos ~]$ sudo vmq-admin cluster show

+--------------------+-------+
|        Node        |Running|
+--------------------+-------+
|VerneMQ@172.26.16.13| false |
|VerneMQ@172.16.16.13| true  |
+--------------------+-------+
{% endcodeblock %}
但是从 26 这一台的指令来看的话：
{% codeblock lang:js %}
[kbz@VM_16_13_centos ~]$ sudo vmq-admin cluster show
+--------------------+-------+
|        Node        |Running|
+--------------------+-------+
|VerneMQ@172.26.16.13| true  |
|VerneMQ@172.16.16.13| false |
+--------------------+-------+
{% endcodeblock %}
换成16 这一台的状态是 false的。那就神奇了。后面查了一下，发现conf漏了一个配置：
{% codeblock lang:js %}
listener.vmq.clustering = 172.16.16.13:44053
{% endcodeblock %}
就是这个配置要换成本机的内网ip才行。 这样改完之后，就可以集群进去了，两台服务器的指令都显示状态正常了。
{% codeblock lang:js %}
[kbz@VM_16_13_centos ~]$ sudo vmq-admin cluster show
+--------------------+-------+
|        Node        |Running|
+--------------------+-------+
|VerneMQ@172.26.16.13| true  |
|VerneMQ@172.16.16.13| true  |
+--------------------+-------+
{% endcodeblock %}
而且要注意，这里面涉及到的端口，比如 44053 6000-7999 4369 端口都要开启权限才行。

---
### 测试
既然集群搭起来了，接下来就是测试了，写了一个测试程序
{% codeblock lang:js %}
var mqtt = require('mqtt');
var clientId = '1528361231';
var username = '101';
var pwd = '38b3eff8baf56627478ec76a704e9b52';
var client1  = mqtt.connect('mqtt://193.112.122.122:1883/',{
    clientId: clientId,
    username: username,
    password: pwd
});

var client2  = mqtt.connect('mqtt://49.51.51.51:1883/',{
    clientId: clientId,
    username: username,
    password: pwd
});

client1.on('connect', function () {
    console.log("client1 connected");
    client1.subscribe(clientId +'/' + username + '/say');
});

client1.on('message', function (topic, message) {
    console.log("client1:" + topic);
    console.log("client1:" + message.toString());
});

client2.on('connect', function () {
    console.log("client2 connected");
    setTimeout(function(){
        console.log("client2 publish");
        client2.publish(clientId +'/' + username + '/say', 'Hello mqtt cluster')
    },2000)
});

client2.on('message', function (topic, message) {
    console.log("client2:" + topic);
    console.log("client2:" + message.toString());
});
console.log("start");
{% endcodeblock %}
初始化了两个客户端（client1 和 client2 分别属于不同的服务器）， 刚开始 client1 连上的时候，开始订阅， 然后 client2 连上之后，2s 之后，开始 publish 一条消息给client1,我们的理想情况，就是 client1 可以收到 client2 publish 的消息。
但是试了一下， client2 publish 之后， client1 一直没有收到消息
{% codeblock lang:js %}
F:\airdroid_code\nodejs\mqtt>node app2.js
start
client2 connected
client1 connected
client1 connected
client2 publish
client2 connected
client1 connected
{% endcodeblock %}
查了以下两台的log，都报了这个错误：
{% codeblock lang:js %}
2018-06-07 08:52:17.150 [debug] <0.28504.1>@vmq_queue:state_change:948 transition from offline --> online because of add_session
2018-06-07 08:52:17.580 [debug] <0.216.0>@plumtree_broadcast:exchange:500 started plumtree_metadata_manager exchange with 'VerneMQ@172.26.16.13' (<0.28505.1>)
2018-06-07 08:52:18.942 [debug] <0.28504.1>@vmq_queue:state_change:948 transition from online --> wait_for_offline because of cleanup
2018-06-07 08:52:18.942 [debug] <0.28501.1>@vmq_mqtt_fsm:connected:405 stop due to disconnect
2018-06-07 08:52:18.942 [debug] <0.28501.1>@vmq_ranch:teardown:134 session normally stopped
{% endcodeblock %}
发现两台都报上面那个错误，试了一下，如果两个client端都连接同一个ip的话，发现竟然也是一样的情况，那说明不是集群的问题，那为啥只连一个client是没有问题的呢？后面怀疑可能跟两条连接都是同一个 client 和 username 一样导致的，不然没法说明为什么会这样。所以后面 client1 和 client2 用不同的账号来连接，代码如下：
{% codeblock lang:js %}
var mqtt = require('mqtt');
var clientId1 = '1528361231';
var username1 = '101';
var pwd1 = '38b3eff8baf56627478ec76a704e9b52';
var client1  = mqtt.connect('mqtt://193.112.122.122:1883/',{
    clientId: clientId1,
    username: username1,
    password: pwd1
});
var clientId2 = '1528361747';
var username2 = '102';
var pwd2 = 'ec8956637a99787bd197eacd77acce5e';
var client2  = mqtt.connect('mqtt://49.51.51.51:1883/',{
    clientId: clientId2,
    username: username2,
    password: pwd2
});

client1.on('connect', function () {
    console.log("client1 connected");
    client1.subscribe(clientId2 +'/' + username2 + '/say');
});

client1.on('message', function (topic, message) {
    console.log("client1 receive:" + topic);
    console.log("client1 receive:" + message.toString());
});

client2.on('connect', function () {
    console.log("client2 connected");
    setTimeout(function(){
        console.log("client2 publish");
        client2.publish(clientId2 +'/' + username2 + '/say', 'client2 publish to say mqtt cluster');
    },2000)
});

client2.on('message', function (topic, message) {
    console.log("client2 receive:" + topic);
    console.log("client2 receive:" + message.toString());
});

console.log("start");
{% endcodeblock %}
上面代码， client1 和 client2 是不同的账号，而且连的是不同的服务器。
而且这两个账号在redis的权限如下（为了避免干扰，先全部设置为#）：
{% codeblock lang:js %}
["","1528361231","101"]
{
    "passhash": "$2a$10$.UVQgAlmBYmHL4T0EGEq9u3poUrrCFQepee8IhrqqaQANNOtpree6",
    "subscribe_acl": [
        {
            "pattern": "#"
        }
    ],
    "publish_acl": [
        {
            "pattern": "#"
        }
    ]
}


["","1528361747","102"]
{
    "passhash": "$2a$10$o2EnyzkqmHE532m3C.MREuyVNSKf0wIkHudKFjSk3OUFPlYLm8xBW",
    "subscribe_acl": [
        {
            "pattern": "#"
        }
    ],
    "publish_acl": [
        {
            "pattern": "#"
        }
    ]
}
{% endcodeblock %}
然后 client1 订阅了一个主题，这个主题会有 client2 来pub， 而 client1 和 client2 是属于不同的服务器的
{% codeblock lang:js %}
F:\airdroid_code\nodejs\mqtt>node app3.js
start
client1 connected
client2 connected
client2 publish
client1 receive:1528361747/102/say
client1 receive:client2 publish to say mqtt cluster
{% endcodeblock %}
后面证明， client2 pub 的时候， client1 是可以收到的， 当前前提是 client2 有pub这个主题的权限， client1 有sub 这个主题的权限，所以接下来我们测一下有设置具体主题权限的情况下：
{% codeblock lang:js %}
client1.on('connect', function () {
    console.log("client1 connected");
    client1.subscribe('say');
});

client1.on('message', function (topic, message) {
    console.log("client1 receive:" + topic);
    console.log("client1 receive:" + message.toString());
});

client2.on('connect', function () {
    console.log("client2 connected");
    setTimeout(function(){
        console.log("client2 publish");
        client2.publish('say', 'client2 publish to say mqtt cluster');
    },2000)
});

client2.on('message', function (topic, message) {
    console.log("client2 receive:" + topic);
    console.log("client2 receive:" + message.toString());
});
{% endcodeblock %}
如果我把client2 设置成没有pub say 主题的权限的话，(下面它的pub权限是 hello，而不是上面要求的 say，所以本例client2 并没有对 say 主题的 pub 权限)
{% codeblock lang:js %}
["","1528361747","102"]
{
    "passhash": "$2a$10$o2EnyzkqmHE532m3C.MREuyVNSKf0wIkHudKFjSk3OUFPlYLm8xBW",
    "subscribe_acl": [
        {
            "pattern": "#"
        }
    ],
    "publish_acl": [
        {
            "pattern": "hello"
        }
    ]
}
{% endcodeblock %}
这时候 client2就会一直重连
{% codeblock lang:js %}
F:\airdroid_code\nodejs\mqtt>node app3.js
start
client1 connected
client2 connected
client2 publish
client2 connected
{% endcodeblock %}
服务器的log显示没有pub的权限：
{% codeblock lang:js %}
2018-06-07 09:53:53.462 [error] <0.31975.1>@vmq_mqtt_fsm:auth_on_publish:662 can't auth publish [<<"102">>,{[],<<"1528361747">>},0,[<<"say">>],<<"client2 publish to say mqtt cluster">>,false] due to error
{% endcodeblock %}
这个时候就会报没有pub权限的错误。
反过来相反，client2 有 pub say 主题的权限，但是 client1 没有 sub say 主题的权限，那么就会出现
{% codeblock lang:js %}
F:\airdroid_code\nodejs\mqtt>node app3.js
start
client1 connected
client2 connected
client2 publish
{% endcodeblock %}
这时候就会出现 client2 publish，然后没有响应的情况。
当然也可以client1 和 client2 都sub 同一个主题，这时候只要任何一个client publish 主题，两个client都会收到，相当于广播，当然，两个client 都要对这个主题有sub和pub 的权限：
{% codeblock lang:js %}
["","1528361747","102"]
{
  "passhash": "$2a$10$o2EnyzkqmHE532m3C.MREuyVNSKf0wIkHudKFjSk3OUFPlYLm8xBW",
  "subscribe_acl": [
    {
      "pattern": "say"
    }
  ],
  "publish_acl": [
    {
      "pattern": "say"
    }
  ]
}

["","1528361231","101"]
{
    "passhash": "$2a$10$.UVQgAlmBYmHL4T0EGEq9u3poUrrCFQepee8IhrqqaQANNOtpree6",
    "subscribe_acl": [
        {
            "pattern": "say"
        }
    ],
    "publish_acl": [
        {
            "pattern": "say"
        }
    ]
}
{% endcodeblock %}
代码如下：
{% codeblock lang:js %}
var mqtt = require('mqtt');
var clientId1 = '1528361231';
var username1 = '101';
var pwd1 = '38b3eff8baf56627478ec76a704e9b52';
var client1  = mqtt.connect('mqtt://193.112.122.122:1883/',{
    clientId: clientId1,
    username: username1,
    password: pwd1
});
var clientId2 = '1528361747';
var username2 = '102';
var pwd2 = 'ec8956637a99787bd197eacd77acce5e';
var client2  = mqtt.connect('mqtt://49.51.51.51:1883/',{
    clientId: clientId2,
    username: username2,
    password: pwd2
});

client1.on('connect', function () {
    console.log("client1 connected");
    client1.subscribe('say');
    setTimeout(function(){
        console.log("client1 publish");
        client1.publish('say', 'this message is publish by client1');
    },5000)
});

client1.on('message', function (topic, message) {
    console.log("client1 receive:" + topic);
    console.log("client1 receive:" + message.toString());
});

client2.on('connect', function () {
    console.log("client2 connected");
    client2.subscribe('say');
    setTimeout(function(){
        console.log("client2 publish");
        client2.publish('say', 'this message is publish by client2');
    },2000)
});

client2.on('message', function (topic, message) {
    console.log("client2 receive:" + topic);
    console.log("client2 receive:" + message.toString());
});

console.log("start");
{% endcodeblock %}
执行结果如下：
{% codeblock lang:js %}
F:\airdroid_code\nodejs\mqtt>node app3.js
start
client1 connected
client2 connected
client2 publish
client2 receive:say
client2 receive:this message is publish by client2
client1 receive:say
client1 receive:this message is publish by client2
client1 publish
client1 receive:say
client1 receive:this message is publish by client1
client2 receive:say
client2 receive:this message is publish by client1
{% endcodeblock %}

所以集群是可以的，没毛病!!!

