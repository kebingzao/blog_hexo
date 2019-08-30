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
目前这种方式，是因为攻击者的 user-agent 是比较特殊的，所以我们可以这样去处理，但是一旦攻击者的 user-agent 是合法的，比如浏览器的合法 agent，那么我们将没办法处理。
所以后面可能要优化的方式就是：
- 攻击自动检测：对 nginx 日志进行自动分析和检测，出现对某个接口异常高请求量时及时报警。
- 攻击预防：DDOS 攻击可能从不同 IP 、不同 user agent 发起，所以单纯的 IP 黑名单或者 user agent 过滤很难解决所有问题。可以考虑使用云服务商提供的 DDOS 防御功能。

---

## 又攻击了
稳定了一天，然后第二天这个攻击者又开始攻击了，这一次更牛逼了，user-agent 都不一样了，而且都是合法的：
![1](7.png)
这些可难办了，而且我怀疑他之所以会换 user-agent， 是因为之前通过过滤 user-agent 的时候，返回的是 403，导致他知道被我们挡掉了。所以才换 user-agent，结果发现返回 200，那就是绕过我们的防御机制了。而且因为看起来都是真实ip，每一个ip的请求频率都不高，一两分钟才请求一次，属于正常的用户行为，我们也不可能去过滤掉这些 ip。所以临时方案只能再去根据攻击者的一些规则再去设置 nginx 过滤了， 这次的设置规则，要包含几点：
1. 针对该攻击者攻击的登录接口，做过滤，因为都是请求这一个登录接口
2. 针对攻击者请求的 referer 做过滤，通过 nginx log 来看， referer 都是同一个，而且还带有多语言路径， 即 **ru/signin**, 所以攻击者应该是俄罗斯的用户
3. 针对攻击者请求的 user-agent 做过滤，通过 nginx log 来看，都是属于手机端来请求，但是事实上我们的官网是 pc 端官网，很少有手机端请求的，所以通过这一点可以过滤掉大部分的正常用户，将苗头指向攻击者
4. 攻击者的目的就是用字典的方式去暴力破解密码，所以我们将返回值固定返回 200， 并且值是 **{"code":"-21","msg":"Account or password not correct."}**，让攻击者看返回值的时候，跟正常用户的登录失败的返回值一样，他就察觉不到其实我们已经把他挡住了，其实请求根本没有到 php 层，还以为真的密码错误呢。

所以最后设置的 nginx 的防御规则就是：同时要符合这三个条件，最后才固定返回 200 和对应的错误提示，如果 **default_type** 前面已经设置了，这边就不用单独设置
```text
	set $flag 0;
    if ($request_uri ~* "/user/signin") {
        set $flag "${flag}1";
    }
    if  ($http_referer ~* "/ru/signin") {
        set $flag "${flag}2";
    }
    if ( $http_user_agent ~* "Android|iPhone; iOS") {
        set $flag "${flag}3";
    }
    if ($flag = "0123") {
        default_type application/json;
        return 200 '{"code":"-2","msg":"Account or password not correct."}';
    }	
```
这样就可以将范围缩短的很少，但是不可否认的是，如果有普通用户在手机端登录官网，并且选择俄语界面来登录的话，这一类的用户也会被波及了。
用 curl 模拟浏览器请求：
```javascript
curl 'https://foo.com/user/signin?jtoken=null' -H 'Accept: application/json, text/javascript, */*; q=0.01' -H 'Referer: https://foo.com/ru/signin/' -H 'User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/76.0.3809.100 Safari/537.36' -H 'Sec-Fetch-Mode: cors' -H 'Content-Type: application/x-www-form-urlencoded; charset=UTF-8' --data 'mail=xxx%40bizz.com&pwd=sdfsdf&keep=0&device_type=&callback_url=&csrf_token=&jtoken=' --compressed

{"code":"-2","msg":"Account or password not correct."}
```
返回值也是对的。
所以通过这种方式，就可以将大部分的攻击都挡下来了。虽然还是会有某些情况下会有误杀的情况，但是至少服务器负载下来了，稳定了。
## 后续再优化
以上这种方式，只能是临时，不可能是长久的。 所以还是要优化我们的禁 ip 路由。目前我们的禁 ip 策略，规则比较简单，就是如果一分钟访问次数超过 100 次，就把这个 ip 禁掉。但是很明显不适合 DDOS 的这种情况。他们是有真实 ip 的 ip 池的，而且请求频率并不会太高。所以后面可以针对他们刷登录接口的这个做一下限制：比如如果同一个 ip 在 10 分钟之内对登录接口访问超过 20 次，那么说明有被刷接口的嫌疑，就把这个 ip 封掉，可以 24 小时之后再解禁之类的。
