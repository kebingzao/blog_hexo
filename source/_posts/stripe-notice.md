---
title: stripe 支付需要注意的点 -- 持续更新
date: 2019-07-19 14:43:32
tags: stripe
categories: 支付相关
---
## 前言
之前项目有用到了一些第三方支付，包括 paypal, google iap, stripe, apple iap, 还有国内的 alipay。其中每个支付类型都有一些坑，本章讲的是第三方支付 stripe 所遇到的一些问题。
尝试过很多种支付方式，发现 stripe 真的很友好，不管是对客户还是对商家，体验都是不错的， 而且 stripe 是直接用信用卡支付的， 不需要像 paypal 那样还需要拥有账号。
<!--more-->
## 无法加载 stripe 的 js sdk
之前在做手机端内嵌页支付的时候，有遇到过一个问题，就是一直没办法加载stripe的sdk：**https://js.stripe.com/v3/**，但是其他手机都没有问题。就这一台手机一直加载不出来，一直超时。
后面查了一下，发现是这台手机的时间设置为 2020 年 4 月，已经超过了 stripe 这个站点的 证书 时间，导致证书过期。因此就没法访问了。后面只要把手机时间调回来就行了。
## 关于 stripe 首次购买之后， webhook 延迟的问题
之前在做 rs 支付的时候，有遇到一个问题，就是 stripe 如果是首次购买的时候， webhook 会延迟过来，一般是一个小时之后才过来。 而且我发现 invoice.payment_succeeded 这个 webhook 是有过来的， 但是 charge 字段是空的，所以根本就没有扣款。
后面查了这一单的情况：
```html
{
    "id": "in_1EuYAEAP4l9EoBkAT02qEtbl",
    "object": "invoice",
    ...
    "billing_reason": "subscription_cycle",
    "charge": null,
    "closed": false,
    ...
    "next_payment_attempt": 1562739374,
    ...
    "status": "draft",
    ...
    "webhooks_delivered_at": 1562735786
}
```
发现它的状态不是 active， 说明这一次扣款是失败的， 而这个webhook 是马上过来的时候，所以应该还处于试用期。 所以扣款失败。然后尝试下次扣款是一个小时之后，那时候试用期就会过了。
后面查了一下代码，发现是这边的问题:
```php
$startTime = time()+20; // stripe创建循环如果有指定生效时间的话，那么就必须是未来的时间，不然会报错
$success = $service->sub($feeMode, $customer, $startTime, $fromType, $inOrderId, $recurringId);
```
原来是 startTime 设置了一个 未来的时间，这个本来是没错的， 但是问题就在于，如果这样做的话，就会导致第一次的扣款失败。 而下一次的尝试扣款就会在一个小时之后。就会出现我们现在的这种情况。所以如果是首次扣款的话，应该不要设置生效时间，哪怕是几秒之后。 直接设置为 null 就可以了。
```php
$startTime = null; // 马上扣款的话，就要设置为null
$success = $service->sub($feeMode, $customer, $startTime, $fromType, $inOrderId, $recurringId);
```
这样子，设置为 null 之后，就会马上扣款了。 不会再延迟一个小时了。
## 已经创建的plan是不能修改价钱
stripe 有更新 plan 的api ([文档](https://stripe.com/docs/api/plans/update))， 然而遗憾的是，已经创建的 plan 有些参数是不能修改的，比如 plan ID, 金额，币种，循环周期等。
```javascript
Updates the specified plan by setting the values of the parameters passed. Any parameters not provided are left unchanged. By design, you cannot change a plan’s ID, amount, currency, or billing cycle.
```
所以如果要改价钱的话，直接新建一个新的 plan。
## 创建循环订单的时候，可以自定义 metadata
stripe 允许我们在创建订单的时候，可以自定义一些信息在 metadata 里面。可以将一些我们产品的信息，比如这个支付用户的accountId，放在 metadata 中，这样子我们后面就可以通过这个循环订单的信息来知道是属于哪个用户的。
```php
\Stripe\Stripe::setApiKey("sk_test_4eC39Hqxxx");

\Stripe\Subscription::create([
  "customer" => "cus_FSmIlO8dkvOxiF",
  "items" => [
    [
      "plan" => "plan_FSm6isCPqQnI0Q",
    ],
  ],
  "metadata" => [
      "account_id" => '123456'
  ]
]);
```
这样子，如果支付成功，过来的支付成功的 webhook 里面就可以知道这个循环是那个用户创建并完成支付的了：
```php
{
	"id": "evt_1ExrRhAP4l9EoBkALxxxxx",
	"object": "event",
	"api_version": "2017-06-05",
	"created": 1563524884,
	"data": {
		"object": {
			"id": "in_1ExrRfAP4l9EoBkAZhPbVY1M",
            ...
			"lines": {
				"object": "list",
				"data": [{
					"id": "sub_FSn1Itcxxxx",
					"object": "line_item",
					"amount": 2499,
					"currency": "usd",
					"description": null,
					"discountable": true,
					"livemode": false,
					"metadata": {
						"account_id": "181544517"
					},
					"period": {
						"end": 1595147283,
						"start": 1563524883
					},
					"plan": {
						"id": "337",
						"object": "plan",
						"active": true,
                        ...
					},
                    ...
				}],
				"has_more": false,
				"total_count": 1,
				"url": "/v1/invoices/in_1ExrRfAP4l9EoBkAZhPbVY1M/lines"
			},
            ...
		}
	},
	"livemode": false,
	"pending_webhooks": 1,
	"request": {
		"id": "req_a0IZqunPgbESa3",
		"idempotency_key": null
	},
	"type": "invoice.payment_succeeded"
}
```
可以看到在 sub 的详细里面，其实是有带上 metadata 的，通过这个方式有解决了一个我们之前遇到的一个问题。
就是我们在调用 sub 创建循环并支付的时候，因为是同步请求，有时候 http 返回比较慢，反而支付成功的 webhook 却提前过来，这时候是在表里面找不到这个循环的。但是我们从metadata中可以得到accountId的值，因此就去查找这个用户是否存在，如果存在，所以就补上循环，并且正常进行升级流程了。
## 最低支付要超过0.5美金才行
之前在进行stripe订单支付的时候，如果 amount = 50 ， 也就是 0.5美金的时候。这时候就会支付不了，报这个错误：
```php
stripe charge errorAmount must convert to at least 400 cents. $0.50 USD converts to approximately $3.92 HKD.
```
也就是说，金额一定要超过 0.5美金。 后面重试了一把，将 amount 改成 51，也就是 0.51 美金，结果真的支付成功了。所以最低都要是 0.51 美金，才能stripe 付款。 具体可以看这个：[文档](https://stackoverflow.com/questions/35505886/stripe-checkout-is-not-working)
## 一次操作，进行多次付款的方法
stripe 一个 token(前端得到) 只能支付一次，如果一个token想要支付两次的话，那么就会报这个错误：
```php
You cannot use a Stripe token more than once: tok_1C5Q9nAP4l9EoBkACxxauZlk.
```
但是如果是用创建的 customer 来支付的话，就可以在程序里面同时支付多次。所以我们可以先通过这个token，直接创建一个customer，然后用这个customer来进行两次支付，一次是普通支付，一次是循环支付，这个是可以的。
## stripe 的优惠券机制
stripe 可以创建优惠券： [discount](https://stripe.com/docs/subscriptions/discounts), [coupons](
https://stripe.com/docs/api#coupons)。
这边需要知道的一点就是：
- 优惠券一旦创建成功，有 sub 或者客户使用的话，就算之前删除了，但是之前已经订阅过的订阅和用户还是照样生效。这一点跟 sub plan 一样，一旦创建成功，有用户订阅之后，就算删掉，已经存在的用户订阅还是会继续存在。
- 普通支付是不能用优惠券的。

具体使用，文档上已经写的很清楚了，这边不细说了
## 设置扣款失败的会在几天内尝试扣款
google 那边可以设置扣款失败之后，之后的重试天数，我们设置为 7 天之内，会再多次尝试扣款， stripe 也可以在后台测试：[文档](https://stripe.com/docs/subscriptions/lifecycle)，一般都是设置为 7 天之内。
## 试用期的最长时间
什么叫试用期， 就是如果某一个用户是高级账号，假设今天是 6月13日， 然后这个用户是7月1日高级账号到期，这时候今天他买了stripe循环支付。这时候创建订阅的时候，就可以设置为试用期就是 30-13 = 17 天 或者就是把试用期设置为 7月1日到期。也就是虽然用户创建了一个订阅，但是到7月1日之前，都是试用期，不扣钱，只有到了7月1号，才扣钱。
而stripe的试用期最长是 10年。
## 最长订阅周期
google 和 PayPal 的最长订阅周期都是只有一年，stripe 虽然可以在创建 plan 的时候，自定义周期：
```php
$plan = \Stripe\Plan::create(array(
  "name" => "Basic Plan",
  "id" => "basic-monthly",
  "interval" => "month",
  "interval_count" => "5",
  "currency" => "usd",
  "amount" => 0,));
```
以上表示循环周期是 5个月。 按照道理说， interval 设置为 year，然后 interval_count 设置为 100， 那不就是 100 年了。
但是后面查了一下 api 方法，发现其实最大只允许 1年的周期 （创建的时候）。 [文档](https://stripe.com/docs/api#create_plan)


