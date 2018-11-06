---
title: VerneMQ 集群状态如果一台挂了，其他也会reset的问题
date: 2018-11-06 11:36:28
tags: 
    - mqtt
    - vernemq
categories: webrtc相关
---
## 前言
为了防止webrtc 的 mqtt 服务有问题，导致我们的webrtc服务出问题， 我们有针对vernemq 程序做了监控，就是每隔几分钟就会去 pub 这几台 vernemq 服务。
如果pub 失败的话，就会去重启这一台有问题的机器。
但是之前有出现了一个很严重的情况，就是重启了之后，还是pub失败，然后就一直重启，一直重启，不断循环？？
后面看了一下，其中两台的log（广州和香港），好像看不出来，是正常的 connet 连接失败？？？
<!--more-->
```javascript
2018-09-10 02:49:39.617 [debug] <0.229.0>@plumtree_broadcast:exchange:500 started plumtree_metadata_manager exchange with 'VerneMQ@172.19.0.6' (<0.1621.0>)
2018-09-10 02:49:39.622 [debug] <0.229.0>@plumtree_broadcast:schedule_lazy_tick:700 0ms mailbox traversal, schedule next lazy broadcast in 10000ms, the min interval is 10000ms
2018-09-10 02:49:39.653 [debug] <0.1621.0>@plumtree_metadata_exchange_fsm:exchange:168 completed metadata exchange with 'VerneMQ@172.19.0.6'. nothing repaired
2018-09-10 02:49:46.399 [warning] <0.1628.0>@vmq_mqtt_fsm:check_user:566 can't authenticate client {[],<<"0a91a0ffdc00216b037226c46b46">>} due to error
2018-09-10 02:49:46.399 [debug] <0.1628.0>@vmq_ranch:teardown:134 session normally stopped
2018-09-10 02:49:46.563 [warning] <0.1630.0>@vmq_mqtt_fsm:check_user:566 can't authenticate client {[],<<"0a91a0ffdc00216b037226c46b46">>} due to error
2018-09-10 02:49:46.563 [debug] <0.1630.0>@vmq_ranch:teardown:134 session normally stopped
2018-09-10 02:49:49.618 [debug] <0.229.0>@plumtree_broadcast:exchange:500 started plumtree_metadata_manager exchange with 'VerneMQ@172.26.16.13' (<0.1632.0>)
```
也就是说，重启了之后，还是 连接不了 ？？？
## 解决
后面我看了一下整个集群情况：
```html
[kbz@VM_0_6_centos ~]$ sudo vmq-admin cluster show
+--------------------+-------+
| Node |Running|
+--------------------+-------+
|VerneMQ@172.26.16.13| false |
|VerneMQ@172.16.16.13| true |
| VerneMQ@172.19.0.6 | true |
+--------------------+-------+
```
发现 172.26.16.13 这一台的集群竟然挂了，而这一台是硅谷那一台，所以我们就到这一台服务器上去：
```html
[kbz@VM_16_13_centos ~]$ sudo vmq-admin cluster show
Node 'VerneMQ@172.26.16.13' not responding to pings.
```
发现确实挂掉了，而且没有重启。
查了一下log，发现已经挂了两天了，而且supervisor也没法启动起来：
```html
2018-09-08 02:55:17.082 [error] <0.2516.117> gen_server vmq_cluster_mon terminated with reason: no such process or port in call to [{gen,do_for_proc,2,[{file,[103,101,110,46,101,114,108]},{line,261}]},{gen_event,rpc,2,[{file,[103,101,110,95,101,118,101,110,116,46,101,114,108]},{line,197}]},{vmq_cluster_mon,terminate,2,[{file,[47,111,112,116,47,118,101,114,110,101,109,113,47,100,105,115,116,100,105,114,47,66,85,73,76,68,47,49,46,52,46,48,47,95,98,117,105,108,100,47,100,101,102,97,117,108,116,47,108,105,98,47,118,109,113,95,115,101,114,118,101,114,47,115,114,99,47,118,109,113,95,99,108,117,115,116,101,114,95,109,111,110,46,101,114,108]},{line,153}]},{gen_server,try_terminate,3,[{file,[103,101,110,95,115,101,114,118,101,114,46,101,114,108]},{line,629}]},{gen_server,terminate,7,[{file,[103,101,110,95,115,101,114,118,101,114,46,101,114,108]},{line,795}]},{proc_lib,init_p_do_apply,3,[{file,[112,114,111,99,95,108,105,98,46,101,114,108]},{line,247}]}]
2018-09-08 02:55:17.082 [error] <0.2516.117> CRASH REPORT Process vmq_cluster_mon with 0 neighbours exited with reason: no such process or port in call to gen_server:terminate/7 line 800
2018-09-08 02:55:17.082 [debug] <0.2518.117> Supervisor vmq_cluster_node_sup started vmq_cluster_mon:start_link() at pid <0.2516.117>
2018-09-08 02:55:17.083 [error] <0.2518.117> Supervisor vmq_cluster_node_sup had child vmq_cluster_mon started with vmq_cluster_mon:start_link() at <0.2516.117> exit with reason noproc in context child_terminated
2018-09-08 02:55:17.083 [error] <0.2518.117> Supervisor vmq_cluster_node_sup had child vmq_cluster_mon started with vmq_cluster_mon:start_link() at <0.2516.117> exit with reason reached_max_restart_intensity in context shutdown
2018-09-08 02:55:17.083 [error] <0.280.0> Supervisor vmq_server_sup had child vmq_cluster_node_sup started with vmq_cluster_node_sup:start_link() at <0.2518.117> exit with reason shutdown in context child_terminated
```
后面手动重启了一下，发现集群就正常了：
```html
[kbz@VM_16_13_centos ~]$ sudo service vernemq start
Starting vernemq (via systemctl): [ OK ]
[kbz@VM_16_13_centos ~]$ sudo vmq-admin cluster show
+--------------------+-------+
| Node |Running|
+--------------------+-------+
|VerneMQ@172.26.16.13| true |
|VerneMQ@172.16.16.13| true |
| VerneMQ@172.19.0.6 | true |
+--------------------+-------+
```
这时候发现另外两台（广州，香港）的 connect，pub， sub 都正常了。

## 原因分析
这种情况很奇怪，一台集群中的机器挂掉了，竟然会影响另外两台机器的 connect，pub，sub，直接就 reset，连重启都没有用？？？
而且这种情况是可以重现的，这三台机器，一旦某一台从集群中挂掉，并且没有重启，另外两台的connect，pub，sub都会拒绝掉？？
后面查了一下官网的文档，发现有一种情况非常符合：[clustering netsplits](http://vernemq.com/docs/clustering/netsplits.html)
对应的issue： [can't register client due to reason not_ready](https://github.com/vernemq/vernemq/issues/36)
{% blockquote http://vernemq.com/docs/clustering/netsplits.html %}
VerneMQ is able to detect a network partition, and by default it will stop serving CONNECT, PUBLISH, SUBSCRIBE, and UNSUBSCRIBE requests. A properly implemented client will always resend unacked commands and messages are therefore not lost (QoS 0 publishes will be lost). However, the time window between the network partition and the time VerneMQ detects the partition much can happen. Moreover, this time frame will be different on every participating cluster node. In this guide we're referring to this time frame as the Window of Uncertainty.
{% endblockquote %}
解决方法就是开启这几个配置： 
- **allow_register_during_netsplit**
- **allow_publish_during_netsplit**
- **allow_subscribe_during_netsplit**
- **allow_unsubscribe_during_netsplit**

```html
## Allow new client connections even when a VerneMQ cluster is inconsistent.
##
## Default: off
##
## Acceptable values:
##   - on or off
allow_register_during_netsplit = on

## Allow message publishs even when a VerneMQ cluster is inconsistent.
##
## Default: off
##
## Acceptable values:
##   - on or off
allow_publish_during_netsplit = on

## Allow new subscriptions even when a VerneMQ cluster is inconsistent.
##
## Default: off
##
## Acceptable values:
##   - on or off
allow_subscribe_during_netsplit = on

## Allow clients to unsubscribe when a VerneMQ cluster is inconsistent.
##
## Default: off
##
## Acceptable values:
##   - on or off
allow_unsubscribe_during_netsplit = on
```
其实就是当检测到网络分区的时候，是否还允许集群中的其他台可以正常进行 CONNECT, PUBLISH, SUBSCRIBE, 和 UNSUBSCRIBE 请求。
但是很神奇的是，我们的三台机器虽然是在不同的大区，一个国内，一个香港，一个硅谷，但是都是通过内网连接的。按照道理来说的话，应该不至于会有网络割裂的问题。
但是神奇的是，这几个配置开起来之后，后面如果有一台集群挂了之后，其他两台也能正常work了。













