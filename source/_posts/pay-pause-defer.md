---
title: paypal,google iap,stripe 循环订单暂停和延迟结算
date: 2020-04-18 14:03:40
tags: 
    - paypal
    - google iap
    - stripe
categories: 支付相关
---
## 前言
前段时间运营那边有个需求给开发， 之前运营那边有做了一个购买 2-3 年订单的优惠活动， 导致很多用户都买了这个活动。 但是后面发现这些用户中，有很多用户本来就是我们产品的 vip 用户，而且本身就有循环订购我们的产品。 也就是说，虽然我们在后台给他们加了 2-3 的 vip 的时限，但是因为这些用户本身就有循环订阅，导致 vip 还没有用完的时候， 下一个循环周期就扣费了。 这样子就会对用户的权益造成影响。 所以运营那边希望开发将这些用户找出来，并且将这些循环订单进行延迟结算，一直到用户本身的 vip 快到期的时候，才重新激活循环订阅，使其重新扣费。

我们现在的循环订阅的支付方式有 3 种， PayPal, Stripe, google iap。所以要针对这三种方式的循环支付进行额外的处理。

<!--more-->

## 针对 PayPal 循环的处理 -- 先暂停,时间到了再程序激活
查了一下 PayPal 官网的 api， 发现 PayPal 没有延迟结算的功能，所以只能先将用户的循环暂停，然后等到时间到了，再用脚本程序重新激活 (需要一个计划任务)，相关资料：

- http://paypal.github.io/PayPal-PHP-SDK/sample/
- http://paypal.github.io/PayPal-PHP-SDK/sample/doc/billing/ReactivateBillingAgreement.html

暂停订阅的相关代码如下:
```php
$contextData = $this->getPaypalConfigData($isOld);
try {
    $agreement = Agreement::get($recurringId, $contextData["context"]);
} catch (Exception $ex) {
    $this->logger->error($ex->getMessage());
    return null;
}
$this->logger->info("$accountId status: {$agreement->getState()}");
$this->logger->info("$accountId Retrieved an Agreement" . "Agreement" . $agreement->getId() . $agreement);
$state = $agreement->getState();
if (in_array($state, ["Cancelled", "Suspended"])) {
    $this->logger->info("$accountId recurring is cancel, status: $state");
    // 这时候说明这个循环早就取消了
}else{
    // 循环已经存在
    // 接下来就是暂停 PayPal 的循环操作
    if($state == 'Active'){
      $isOk = true;
      $agreementStateDescriptor = new AgreementStateDescriptor();
      $agreementStateDescriptor->setNote("Suspending the agreement");
      try {
          $agreement->suspend($agreementStateDescriptor, $contextData["context"]);
      } catch (Exception $ex) {
          $isOk = false;
          $this->logger->error("$accountId suspend paypal error:" . $ex->getMessage());
      }
      // 重新请求一下看是否请求成功
      if($isOk){
          try {
              $agreement = Agreement::get($recurringId, $contextData["context"]);
              if($agreement->getState() == 'Suspended'){
                  $this->logger->info("$accountId paypal suspend success");
              }
          } catch (Exception $ex) {
              $this->logger->error($ex->getMessage());
          }
      }
  }
}
```
然后后面再做一个计划任务，等用户的 vip 快过期了，再把他的订阅重新激活:
```php
$agreementStateDescriptor = new AgreementStateDescriptor();
$agreementStateDescriptor->setNote("Reactivating the agreement");

try {
    $suspendedAgreement->reActivate($agreementStateDescriptor, $apiContext);
    $agreement = Agreement::get($suspendedAgreement->getId(), $apiContext);
} catch (Exception $ex) {
    $this->logger->error("Reactivate the Agreement", "Agreement", $agreement->getId(), $suspendedAgreement, $ex);
    exit(1);
}
$this->logger->info("Reactivate the Agreement", "Agreement", $agreement->getId(), $suspendedAgreement, $agreement);
return $agreement;
```
这样就可以了，不过这样子会有一个问题: PayPal 因为暂停循环，也就是 suspend 操作，会导致 PayPal 平台会给用户发送 suspend 提醒邮件。 而且由于后面还要启用这个循环，导致可能再次扣款，所以需要运营要跟用户沟通一下。同时在个人中心页面，也会变成没有循环了。只有当激活之后，重新扣款的时候，才会重新变成有循环。

## 针对 Stripe 循环的处理 -- 暂停或者延长试用期
Stripe 会比 Paypal 灵活很多，他提供了两种方式:

### Stripe 延长试用期
stripe 可以通过通过延长试用期来 延长下一次扣款的周期：

- https://stripe.com/docs/billing/subscriptions/billing-cycle#api-trials

不过这种方式 最多只能添加 730 (两年)，不太符合我们的情况，因为我们还有 3年 的情况。

### Stripe 暂停订阅，并设置自动恢复的时间
不过他还有暂停订阅的情况，而且可以设置自动恢复循环的日期 (resume_at)：

- https://stripe.com/docs/billing/subscriptions/pausing  这个暂停循环的。

所以后面就采用暂停循环，然后设置重新恢复的时间。 代码如下:
```php
$update = Subscription::update($recurringId, [
    [
        'pause_collection' => [
            'behavior' => 'mark_uncollectible',
            'resumes_at' => $resumeAt
        ],
    ]
]);
$this->logger->info("$accountId stripe update" . json_encode($update));
```
如果把这个 update 对象打出来，可以看到:
```php
    ...
    "pause_collection": {
        "behavior": "mark_uncollectible",
        "resumes_at": 1621809047
    },
    ...
    "status": "active",
```
有这个暂停的消息，并且循环的状态还是 active 状态的。说明暂停循环，并不会改变当前循环的状态 (paypal 会改变)

## 针对 google iap 循环的处理 -- 延迟结算
google 循环暂停，程序没有 api，只能用户自己手动去暂停 (也可以恢复，但是也要用户自己手动恢复)， 所以只能延迟扣款:

- https://developer.android.com/google/play/billing/billing_subscriptions#Defer
- https://developers.google.com/android-publisher/api-ref/purchases/subscriptions/defer

而且这边要注意两个细节：
1. 延迟的周期是有范围的，最短是一天，最长是一年， 超过或者小于都会报错:
```php
{
  "error": {
    "code": 400,
    "message": "The desired expiry time for the subscription is not valid.",
    "errors": [
      {
        "message": "The desired expiry time for the subscription is not valid.",
        "domain": "androidpublisher",
        "reason": "subscriptionDeferInvalidTime"
      }
    ]
  }
}
```
2. defer 接口虽然会返回一个新的过期时间，但是这个过期时间跟真正延迟结算的过期时间是不一样 (试验了，大概会差两个小时)，所以延迟之后，还要请求接口得到正确的过期时间

代码如下:
```php
$ap = Factory::newApClient();
$purchase = $ap->purchases_subscriptions->get($packageName, $recurringItem->subscription_id, $recurringItem->token);
if ($purchase) {
    if($purchase->getAutoRenewing() == "true"){
        $postBody = new \Google_Service_AndroidPublisher_SubscriptionPurchasesDeferRequest();
        $info = new \Google_Service_AndroidPublisher_SubscriptionDeferralInfo();
        $info->setExpectedExpiryTimeMillis($originExpire);
        // 每次调用API的时间最多可以推迟一天，最长可以推迟一年。您可以在新的结算日期到来之前再次调用API，以进一步推迟结算。
        // 如果有超过一年的话，那么就要分两次，第一次先延迟一年，等那一年快到期了，再延期一年。
        if($desiredTime - $originExpire > 86400 * 365 * 1000){
            $desiredTime = $originExpire + 86400 * 365 * 1000;
        }
        $info->setDesiredExpiryTimeMillis($desiredTime);
        $postBody->setDeferralInfo($info);
        $result = $ap->purchases_subscriptions->defer($packageName, $recurringItem->subscription_id, $recurringItem->token, $postBody);
        $this->logger->info("$accountId google defer new expire:" . $result->getNewExpiryTimeMillis());
        // 接下来重新检查一下是否 defer 成功
        $newPurchase = $ap->purchases_subscriptions->get($packageName, $recurringItem->subscription_id, $recurringItem->token);
        if($newPurchase){
            // 这里会返回一个新的过期时间
            // 这时候发现一个问题，就是上面的过期时间也不能直接用，而是要重新请求新的过期时间
            $recurringItem->expire_time = $newPurchase->getExpiryTimeMillis();
            $recurringItem->last_expire_time = $newPurchase->getExpiryTimeMillis();
            $recurringItem->save();
            $this->logger->info("$accountId google new purchase" . json_encode($newPurchase));
        }
    }
}
```

google 延迟结算，也不会改变循环的状态，不过因为 google 的延迟结算一次最多延迟一年，所以对于有些要延迟1年以上的订单，后面还要做一个计划任务，到明年的这个时候，再进行一次的延迟结算。







