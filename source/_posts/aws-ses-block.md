---
title: 记一次AWS SES 邮件服务账号被禁的情况
date: 2019-05-29 14:08:33
tags: aws
categories: aws 相关
---
## 前言
今天有客服在反应后台那边没法回复邮件了。然后看了一下我们的邮件发送服务，发现了这个报错：
```html
, Call api time: 17.743922252s]
[Error] [queue.go 157] [2017-11-27 07:07:29] [MessageRejected: Sending suspended for this account. For more information, please check the inbox of the email address associated with your AWS account.
    status code: 400, request id: xxxx]
```
发现竟然是 AWS SES ([Simple Email Service AWS 的邮件发送服务](https://aws.amazon.com/ses/)) 那边的错误？ 导致我们的邮件发送全部失败了。
## 分析
整个邮件服务都有问题了。 上 AWS 后台看了一下，账号状态是 **shutdown** 的状态。
<!--more-->
![1](1.png)
后面才发现昨天有收到 AWS 的提醒邮件，说是我们的反弹率已经达到 18% (超过了正常情况的 5%，阀值是 10%)， 因此 AWS 就把我们这个账号的 SES 服务给停掉了。
## 反弹率 bounce
这边科普特一下什么是邮件的反弹率： 具体可以看这个：
- [硬反弹&软反弹 - 终极指南](http://www.emailcamel.com/node/104)
- [邮件服务商(ESP)邮件退信系列文章](http://www.emailcamel.com/node/285)

简单的来说，就是电子邮件无法传送到目标电子邮件地址的时候，就会被弹回来。如果这种情况很多的话，反弹率就会很高。
反弹分为**硬反弹**和**软反弹**。
### 硬反弹
硬反弹通常意味着什么？这意味着电子邮件地址不存在。
- 电子邮件已被删除 
也许高级管理人员离开了他的公司，他们删除了他的电子邮件地址。或者艺术总监也许删除了Hotmail地址，因为他收到太多的垃圾邮件了。电子邮件地址被删除，并永久放弃。这就是为什么电子邮件列表邮箱无效的越来越多的原因。
- 电子邮件永远不存在 -这通常是错字或人为错误的结果。但是，虚假的电子邮件并不总是偶然的。在极少数情况下，恶意竞争对手会以虚假的电子邮件和垃圾邮件陷阱加入你的电子邮件列表。使用双重选择加入您的电子邮件营销活动将减少此问题。

硬反弹的弊端很大。应立即从列表中删除它们。建议半个月进行一次邮件地址清洗工作。找出无效的邮件地址，并删除。

### 软反弹
与硬反弹不同，您仍然有机会与这些潜在客户进行沟通。很多时候软反弹是收件人的一个临时问题。
- 收件箱已满 
如果收件人的收件箱已满，则无法接收邮件。您可以多次向这些潜在客户发送消息，希望他们清理收件箱。
- 电子邮件服务器临时关闭
比如，他们正在服务器上进行维护。一旦ESP服务器重新开启，您可以再次向潜在客户发送电子邮件。
- 你的消息太大了 
如果你的消息太大，它会弹跳。例如，Gmail的发送限制为25 MB。请记住，图片和视频占用了大量的空间。

## 排查
很明显，我们的情况明显是属于硬反弹，而且从过往的 bounce 趋势图来看，基本上都是在 5% 以下，之所以一下子飙到那么高，肯定是有人刷某一个发送邮件的接口。
经过排查(排查发送邮件的表)，发现有个用户一直用我们客户端的一个**通过邮件发送文件**的这个功能来发送一堆的欺诈邮件
![1](2.png)
发了十几万封，难怪会有这种情况发生。
## 处理
### 停邮件队列服务
第一时间就先把邮件队列服务停掉了
### 限制频率
然后就是修改业务的这个接口，加上限制，每天只能发送十次，一次最多十个收件人(之前是没有限制的)
```html
// todo 超过10个邮件，直接返回错误，为了防刷
if _, ok := param["email"]; ok {
       mailTos := strings.Split(param["email"], ",")
       if len(mailTos) >= 10 {
              log.Errorf("mails too many(%v), accountId(%v)", len(mailTos), accountId)
              res.Code = -2
              res.Msg = "too many mails, must be less than 10"
              return utils.EchoDesResult(c.Controller, res)
       }
}
```
```html
key := fmt.Sprintf("mail_file_%v", param["accountId"])
if err := cache.Get(key, &sendNum); err == nil {
       if sendNum > 10 {
              res.Code = -1
              res.Msg = "sending frequent"
              return utils.EchoDesResult(c.Controller, res)
       }
}
```
然后更新到线上去，修复了这个bug
### 恢复服务
接下来就是考虑怎么使得我们的邮件服务重新恢复？？？ 从 AWS 发过来的邮件有说，就是要写申诉邮件，其实就是写一个 case 工单，让他们解禁，然后我们也发了。不过要等到 AWS 他们重新解禁，肯定要过上一段时间的，不能马上有。这个时间一般要 1-2 天，如果不催的话，甚至要更久，这个他们的流程就很多，要一层一层催。 
不过我们还是要有一个备用账号先顶着，后面就找公司的其他有用到 SES 的项目组，要了他们的账号先顶着。
换上这个备用账号的 accessKey 和 secretKey，但是刚开始测的时候，有报了一个错：
```html
AccessDenied: User `arn:aws:iam::xxx:user/ses-for-xxx' is not authorized to perform `ses:SendEmail' on resource `arn:aws:ses:us-east-1:xxx:identity/xxx.com'
```
原来是权限不对，因为我们的from mail是 no-reply@xxx.com 域名， 但是 xxx.com 这个域名并不在这个 SES 账号所允许的发送域名里面，所以后面就让运维加了一下： 在 SES 后台的 Domains 那边，把这个域名加上去，并通过校验：
![1](3.png)
这样就可以了。 所以下一步就是申诉成功，账号解禁之后。再替换回来。
## SES 的身份验证
这边讲一下 SES 发送的身份验证，SES 主要是通过验证发送者邮箱是否合法来判断这个邮件是否要发送的。他有两个验证的标准：
### Domains 验证
第一种验证就是判断发送者邮箱的那个域名是否已经是 SES 后台已经校验过的域名了。 比如上面的 no-reply@xxx.com 这个邮箱作为发送者邮箱，如果要正常发送，那么 xxx.com 这个域名一定要在 SES 管理后台的 Domains 里面，并且要验证通过：
![1](4.png)
如果域名已经合法了，前面的 title 就可以随便写了，我可以写 no-reply， 也可以写 biz 等等。
### Email Addresses 验证
当然如果有时候我们想要将这个发送者的邮箱换成我们自己的邮箱，比如 google 邮箱的话，google.com 这个域名肯定不是我们自己的域名，那么还有另一种方式可以做到，那就是添加发送者白名单。你可以在 SES 管理后台的 Email Addresses 添加一些邮件白名单，这些白名单不需要域名要在 Domains 里面，但是这些白名单里面的邮件也要进行验证过，当然只要这个邮箱是自己的，那么验证就很简单了。
![1](5.png)
然后 SES 服务在发送的时候，如果发现发送者邮件的域名不在 Domains 校验里面，这些就会去匹配邮件白名单，如果在邮件白名单里面的话，也可以正常发送，如果不在的话，就会报这个错误：
```html
 [MessageRejected: Email address is not verified. The following identities failed the check in region US-EAST-1: =?utf-8?B?QWlyRHJvaWQ=?= <xxx@xxx.com>
  status code: 400, request id: 4cdfbf99-81bb-xxx-xxx-xxx]
```
所以后面在切换 SES 备用账号的时候，一定要提前设置好权限的配置。












