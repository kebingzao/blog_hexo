---
title: 创建PayPal沙盒模式并测试支付流程
date: 2019-05-24 16:44:09
tags: paypal
categories: 支付相关
---
## 前前言
最近一直想要整理整个项目的支付相关的文档，所以就翻了一下以前的一些笔记，发现一些不涉及到业务流程相关的实践教程和踩过的坑都挺有用的，就想着干脆写到 blog 算了，顺便充实一下 blog，毕竟最近比较忙，都很久没有更新了，只能拿一些之前的存货出来充场面了 XD。
## 这才是前言
测试 PayPal 的时候，会在 PayPal 的沙盒模式里面支付，这时候就需要沙盒账号。PayPal 的沙盒账号只要有自己的账号就可以创建了，不需要特意到线上收款的那个 PayPal 账号那边创建， 而且是全网通用的。所以我只要自己创建一个个人的PayPal账号，那么就可以在上面创建其他的沙盒账号了。而且为了考虑测试沙盒环境下的跟踪，比如 webhook 的 resend，以及 api 的调用查看，我也需要单独创建一个个人账号的沙盒，以便测试环境可以更好的调试。
## 创建自己的PayPal账号
首先要选择创建的国家是 美国， 而且是个人账号。
<!--more-->
![1](1.png)
填信息，其他的信息都可以随便，但是邮箱要对的，因为要验证。
![1](2.png)
随便google 一个美国的街道填进去，比如：[这个](http://www.fakeaddressgenerator.com/World/us_address_generator)
![1](3.png)
![1](4.png)
这样就创建成功了，接下来就是验证邮箱了:
![1](5.png)
![1](6.png)
点击确认邮件，这样邮箱就校验成功了。这时候会出现验证手机的要求，当然我们的手机号是随便乱填的，选择 not now，跳过
![1](7.png)
![1](8.png)
这样就可以了。 这样就成功创建了自己的PayPal账号，记住账号名和密码。
## 沙盒环境创建app
账号创建了，接下来就是用这个账号创建一个沙盒环境，以供我们的测试环境可以用这个来测试。登录我的个人的PayPal的账号（如果这个账号是 us 国家的，可以在这个过程中省很多事，这也是为什么我会在创建的时候，选择 us）
### 创建一个app
要在这个账号里面创建一个app， 直接在 dashboard中就可以创建
![1](15.png)
但是这边要注意一点，这样一定要提供一个沙盒账号，而且这个账号一定要是 business 类型的。 所以我们接下来就去创建一个business 类型的沙盒账号：（如果刚开始创建的是企业类型的账号，那么这一步是可以省下来的，因为企业类型的那个账号，本身就可以直接用了，不需要再去创建了）
![1](16.png)
选择 accountType 为 business 才行
![1](17.png)
注意，这边创建成功之后，一定要将这个沙盒的 business 类型的账号跟你的当前的开发者账号绑定在一起。通过点击 **log in with PayPal** 登录这个沙盒账号，就可以实现绑定。
![1](18.png)
![1](19.png)
绑定成功，会有提示
![1](20.png)
这时候就绑定成了，这时候在点击 create app 就可以看到有 账号可以选择了。
![1](21.png)
点击创建， 这样就创建成了一个测试的app
![1](22.png)
### 配置 app
接下来就开始配置app了，它有两个模式，一个是沙盒模式 Sandbox，一个是生产环境 Live，配置都差不多，只不过 client Id 和 secret 不一样，配置的 webhook url 也要不一样。因为我们建这个账号主要是为了测试环境的沙盒模式更容易调试，所以我们只要配置沙盒模式就行了，生产环境用的账号不是这个账号，而是项目收款的那个账号。
![1](23.png)
接下来就添加 webhook 了， 填入 webhook url，这个要填测试地址的 webhook url，然后选择 all events
![1](24.png)
这样子 webhook 就配置成功了。就会得到一个 webhook ID, 那么接下来就开始测试了。

## 开发者页面创建沙盒测试账号
接下来就是在自己账号的开发者页面创建沙盒测试账号，这个沙盒账号跟我刚才创建的那个个人的账号是不一样的，我第一步创建的是开发者账号，而这次要创建的是在沙盒环境下，要进行付款而登录的那个账号，不能直接用开发者账号，而是要单独再去创建沙盒账号: [文档地址](https://developer.paypal.com/docs/classic/lifecycle/sb_create-accounts/)
![1](9.png)
选择美国地区，并且是个人账号。（这个已经帮你选好了，因为你之前创建的时候，就是这个）
注意，创建的时候，会进行邮箱验证，如果是之前就存在的沙盒账号，那么就会报错（前面说过，沙盒的付款账号是全 paypal 沙盒系统共用的，所以如果已经在其他的paypal账号那边创建过了，这边也不能重复）
![1](10.png)
这个才是正常的：
![1](11.png)
接下来要选择 No，也就是不要 Bank验证，不然后面会让你填信用卡
![1](12.png)
这样就创建成功了 （我连续创建两个）：
![1](13.png)
也就是一旦创建了这个沙盒账号，那么无论是在哪一个 paypal 账号的沙盒环境下，我这个账号都可以进行支付。
![1](14.png)

## 测试
既然沙盒配好了，对应的付款的沙盒账号也配好了，接下来我们就进行沙盒环境下的支付。当然首先要把测试环境的 client ID 和 secret 和 webhook ID 换成我们这个账号的。
然后就可以测试了，从 nginx log 上可以看到有 webhook 过来了。
![1](25.png)

## 备注
这个只能测试普通支付，没办法测试循环支付。 因为 plan 不存在， 所以如果要测试循环支付的话，就要预先先通过 API 先创建 plan， 然后再进行支付就可以了。
还有一点要注意的是，沙盒环境很不稳定，时灵时不灵的，尤其是在国内，所以后面就做了一层反向代理转发：{% post_link paypal-back %}



