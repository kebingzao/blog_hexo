---
title: 记一次接口受到的基于http请求的DDOS攻击
date: 2019-08-28 19:26:55
tags:
- nginx
- security
categories: web安全
---
## 前因
前几天的一个晚上从03:00开始，某一台业务服务器出现负载持续偏高（不会像往常一样自动下降），接口开始访问超时。开启IP黑名单后也没有降低负载。
![1](1.png)
通过 **htop** 指令，可以看到负载达到了 10 多了:
<!--more-->
![1](2.png)
## 解决
重启该业务服务 和 php-fpm 都没有用。 除非重启实例， 但是重启实例，如果是被人刷接口的话，也解决不了问题。
通过看 nginx 的log， 发现有一个登录接口一直在重复请求：
![1](3.png)
虽然这些请求的 IP 地址不同，但 user agent 都是 tradingView/2.7.1。由此，确定是受到使用大量 http 请求作为手段的 DDOS 攻击。
由于请求有固定的 user agent，所以在 nginx 中根据 user agent 过滤请求，防止请求到达服务即可。
```javascript
if ( $http_user_agent ~* "tradingView") {
                return 403;
        }
```
等了一会儿，果然返回  403 了。
![1](4.png)
而且自己通过 chrome 模拟 user-agent 测试的时候，也会返回 403：
![1](5.png)
过了一会儿，负载降下来了：
![1](6.png)
## 改进措施
攻击自动检测：对 nginx 日志进行自动分析和检测，出现对某个接口异常高请求量时及时报警。
攻击预防：DDOS 攻击可能从不同 IP 、不同 user agent 发起，所以单纯的 IP 黑名单或者 user agent 过滤很难解决所有问题。可以考虑使用云服务商提供的 DDOS 防御功能。






