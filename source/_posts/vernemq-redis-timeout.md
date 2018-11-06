---
title: VerneMQ redis 连接超时问题
date: 2018-11-06 11:58:21
tags: 
    - mqtt
    - vernemq
categories: webrtc相关
---
## 前言
最近vernemq集群中，香港和硅谷那两台有时候会重启，原因就是监控程序 cron 一直没办法 pub 和 sub， 然后调用 监控程序的 web 来重启的。
后面查看了一下香港那一台的log，发现有这么一个错误：
```html
2018-10-11 06:45:08.947 [error] <0.3648.0>@vmq_diversity_script:call_function:64 can't call into Lua sandbox for function auth_on_register due to {timeout,{gen_server,call,[<0.613.0>,{call_function,auth_on_register,[{addr,<<"220.130.153.xx">>},{port,1145},{mountpoint,<<>>},{client_id,<<"39f82305f5230abcc433ef5be342a">>},{username,<<"c177fc757d6be8534a57fa68fb910">>},{password,<<"xxx">>},{clean_session,false}]}]}}
```
看意思好像就是在执行lua脚本的 auth_on_register 这个方法的时候，连接redis超时了？？同时官网也有一个对应的issue： [Timeout error for call into Lua sandbox]( https://github.com/erlio/vernemq/issues/798)。
可能是 redis 的 ping 超时了导致，后面试了一下，每一个集群的 mqtt 服务 分别 ping redis 服务的时间：
除了广州那一台的ping是 18ms， 香港和硅谷那两台分别是 137 ms， 所以差距还是很明显的。所以才会有 调用 lua redis 脚本 timeout 的错误，因为校验的时候，要去连接 redis， 一旦网络差一点，就马上会超时了。
而现在之所以香港 和 硅谷 连接 redis 会超时是因为我们的测试服的 redis 是在泉州的实体机上。而 cn， hk，us 都是在腾讯云上， 其中 cn 在广州， hk 在香港， us 在硅谷。 所以在没有对等网络的连接下， cn 因为线路近，所以时间还行。但是 hk 和 us 那就坑爹了，分分钟就ping 超时。
## 解决
所以后面的解决措施就是： 在 cn 那一台的腾讯云机器上， 部署一台新的 redis ， 用来做 vernemq 的 校验 redis。 这样，由于在对等网络下，全部走内网的情况， 这三台服务器应该都挺快的。
后面部署完之后，广州那一台直接是同一台机子，所以速度就不用说了，0.01ms， 香港那一台走内网也很快，只要 9ms。不过美区那个有点坑，走内网还要 163 ms， 这个后面得再观察一下。
可以这样先试试看，后面如果美区会比较延迟的话，再对 redis 做主从，把一个从库架在美区， 因为本身 vernemq 对 redis 也只会读，不会写。








