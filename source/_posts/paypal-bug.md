---
title: 记几次 paypal 支付中遇到的几个坑 -- 持续更新
date: 2019-05-27 19:46:24
tags: paypal
categories: 支付相关
---
## 前言
之前项目有用到了一些第三方支付，包括 paypal, google iap, stripe, apple iap, 还有国内的 alipay。其中每个支付类型都有一些坑，本章讲的是使用paypal 第三方支付的时候，遇到的一些问题，以及那时候是怎么去解决的。
## 交易记录查询不到
之前客服有反馈一个情况，就是突然间有一天，很多初次付款的 paypal 用户，都在反馈他们付款成功了，但是没有升级上来，后面查了一下 log， 发现在收到 paypal 的 **PAYMENT.SALE.COMPETED** 的 webhook 的时候，我们马上去查交易记录。但是只有查到一条循环创建的记录，没有查到交易成功的记录，所以我们程序就误认为付款其实还没有成功，就没有再继续下去了， 但是其实这时候交易是有成功的。
<!--more-->
![1](1.png)
因为当我过了不到一分钟再去人工查这个交易记录的接口的时候，就会发现已经变成两条了。 一条是循环建立的记录，另一条就是交易记录。
![1](2.png)
而且最开始的 webhook 的 data 里面，也可以看到 resource 里面也有新订单的信息了，但是那时候查的时候，transactions 交易记录里面就是只有一条？
**解决方法：**
在 COMPLETED 阶段的 webhook，虽然有可能交易记录还没有更新过来，但是这时候 webhook 里面已经有新的交易订单号了，因此遇到这种情况的话，其实应该去判断 webhook 里面的 resource 里面有没有当前最新的交易单号，如果有的话，那么其实就是合理的，就是应该升级了。
## 循环创建和交易完成的webhook顺序颠倒了
今天运营又反馈了一个问题， 就是有一个用户paypal 付款成功了，但是有没有升级上来。后面查了一下，竟然不是之前一直反馈的那个交易记录没有更新的问题。而是另一个奇葩的问题，就是 webhook 的循序颠倒了。
```html
[2019-01-06 21:09:23] Paypal.DEBUG: Event Type: PAYMENT.SALE.COMPLETED [] []
[2019-01-06 21:09:23] Paypal.DEBUG: recurringId: I-LL9G1xxxx [] []
[2019-01-06 21:09:23] Paypal.ERROR: recurring id not found in db..... I-LL9G1xxxx [] []
[2019-01-06 21:09:23] Paypal.ERROR: paypal webhook verify error: =>paypal complete check recurrring error:xxx
[2019-01-06 21:09:26] Paypal.DEBUG: received a new webhook:1546808966 [] []
[2019-01-06 21:09:26] Paypal.DEBUG: paypal webhook headers: {"content-type":["application\/json"],...
[2019-01-06 21:09:26] Paypal.DEBUG: paypal webhook content: "{\"id\":...
[2019-01-06 21:09:27] Paypal.DEBUG: Validate Received Webhook Event success, status:SUCCESS,...
[2019-01-06 21:09:27] Paypal.DEBUG: Event Type: BILLING.SUBSCRIPTION.CREATED [] []
[2019-01-06 21:09:27] Paypal.DEBUG: complete billing agreement webhook.... [] []
```
可以看到竟然是 **PAYMENT.SALE.COMPLETED** 这个 webhook 先过来，然后 **BILLING.SUBSCRIPTION.CREATED** 这个再过来。这样顺序就颠倒了，这样就会导致 completed 的时候，根本就找不到 循环订单的记录 ？？？
**解决方法：**
就是如果收到 completed 事件，但是在表里面找不到循环id的时候，这时候就要手动触发 subscription create 的事件，让它插入到订单循环表中，然后再继续进行 completed 的操作，这样就可以了。无论是首次支付 completed 先到的情况，还是往期循环，但是订单循环表里面没有记录，都可以用这种方式来处理。
然后如果接下来 subscription create 过来的时候，如果循环已经存在了，那么直接更新它的下一次付款时间就行了
## webhook 延迟很久才过来
今天有出现过一个paypal的情况，就是 有些用户反馈付款成功之后，好几个小时才升级成功。后面查了一下log，发现确实是这样子，以这个为例： I-HV93GMPJxxxx， 他在03:05 分的时候就付完款了，03:07 分的时候，就收到 sale.completed 的 webhook 了，但是这时候查询交易记录的时候，只有一条，就是循环建立那一条，根本没有支付成功的那一条。一直到 06:11 ，也就是过了3个小时之后，又过来了 sale.completed webhook。
但是从支付完之后，到升级上来，webhook 延迟了 3个小时，这样太坑爹了，严重影响用户体验。
**解决方法：**
跟第一种的方法一样，如果交易记录查不到也没有关系，只要 webhook 有带有效订单号，那么就可以直接插入了，这样就可以解决这个问题。
## 订阅设置的坑
之前有出现过一种情况，就是有一个 paypal 的循环用户，他的上个月，因为卡里面没有钱，所以上个月没有扣款，然后这个月卡里有钱了之后，paypal就再给他扣了一笔款（相当于这个月扣了两笔）。后面查了一下，发现在他的循环订单中，有一个配置：
![1](3.png)
就是这个配置 “将失败的付款添加到下一个结帐期” 为 yes。 导致 paypal 会将这个月失败的付款，添加到下个月的结账期。 导致下个月扣了两笔钱。
后面查了一下 paypal 的api ，发现确实有个api 是可以设置这个项： [这里](https://developer.paypal.com/docs/integration/direct/billing-plans-and-agreements/)
````html
/**
 * Allow auto billing for the outstanding amount of the agreement in the next cycle. Allowed values: `YES`, `NO`. Default is `NO`.
 *
 * @param string $auto_bill_amount
 * 
 * @return $this
 */
public function setAutoBillAmount($auto_bill_amount)
{
    $this->auto_bill_amount = $auto_bill_amount;
    return $this;
}
````
默认是 NO, 但是之前我们在创建 plan 的时候，这个值其实是设置为 yes 的
```html
$merchantPreferences->setReturnUrl($returnUrl)
    ->setCancelUrl($cancelUrl)
    ->setAutoBillAmount("yes")
    ->setInitialFailAmountAction("CONTINUE")
    ->setMaxFailAttempts("0")
    ->setSetupFee(new Currency(array('value' => $setupFee, 'currency' => 'USD')));
```
导致订阅了这个用户，就会附加上这个配置。 而且用户如果已经订阅了之后，还不能在 paypal 后台改: [这边](https://developer.paypal.com/docs/classic/paypal-payments-standard/integration-guide/manage_subscriptions/)。
那么能不能直接改这个 plan 的设置呢：
```html
    public function getPlan(){
        $plan = Plan::get($this->id, $this->apiContext);
        $merchantPreferences = $plan->getMerchantPreferences();
        $this->logger->info("detail:" . json_encode([
                'is_auto_bill_amount' => $merchantPreferences->getAutoBillAmount(),
                'return_url' => $merchantPreferences->getReturnUrl(),
                'create_time' => $plan->getCreateTime(),
                'des' => $plan->getDescription(),
                'id' => $plan->getId(),
            ]));
        return $plan;
    }
    // 将 auto_bill_amount 改成 NO
    public function changeAutoBillAmount(Plan $plan){
        $patchRequest = new PatchRequest();
        $patch = new Patch();
        $merchantPreferences = $plan->getMerchantPreferences();
        $merchantPreferences->setAutoBillAmount("NO");
        $patch->setOp('replace')
            ->setPath('/')
            ->setValue($merchantPreferences);
        $patchRequest->addPatch($patch);
        return $plan->update($patchRequest, $this->apiContext);
    }
```
但是我试了一下，虽然调用好像成功了，但是没有效果？？？
所以只能在新建的 plan 订单中，将他设置为 NO
```html
$merchantPreferences->setReturnUrl($returnUrl)
    ->setCancelUrl($cancelUrl)
    // 这边不要设置扣款失败，就自动累计到下一期
    ->setAutoBillAmount("NO")
    ->setInitialFailAmountAction("CONTINUE")
    // 设置最大扣款次数为 5次
    ->setMaxFailAttempts("5")
    ->setSetupFee(new Currency(array('value' => $setupFee, 'currency' => 'USD')));
```
## 通过 EC-token 请求循环的接口失败
之前有遇到一种情况，就是循环支付的return接口的时候，通过 EC-token 去得到交易记录的时候， 这个请求在很偶然的情况下会失败，而且没有任何的 response 的值，catch 不到东西。
```html
https://api.paypal.com/v1/payments/billing-agreements/EC-1TP65860SG56xxxxx/agreement-execute
```
这样就没办法在库里面通过这个EC-token，去得到对应是循环 Id，因此这一单循环就没有着落了。因为 token 只有在支付结束返回的 return 接口才会带， 这边会变得很麻烦，所以目前只增加错误日志 log。但是后面还是 catch 不到东西，就好像这个请求丢了似的。
**解决方法**:
就是将当下没有请求成功的token保存起来，然后在 ipn 或者 webhook 过来校验 recurringId 找不到的时候，再进行请求。保存 token ，我用的是 redis 的 有序集合 sorted set。 通过这个有序集合可以在进行筛选的时候，去掉已经过期的token， 假设过期时间是 10 分钟:
```html
// 存放 paypal token 的有序集合const 
PAYPAL_TOKEN_ZSET = 'paypalTokenZSet';
// 有效期设置为10分钟const
PAYPAL_TOKEN_EXPIRE = 600;

// =================这些是针对 paypal 的 token sorted set 的处理方式
public function addPpToken($token){
    // 分数就是当前的时间戳
    return $this->redis->zadd(self::PAYPAL_TOKEN_ZSET, [
        $token => time()
    ]);
}
// 移除所有有效期的token
public function delPpTokens($token){
    $this->redis->zrem(self::PAYPAL_TOKEN_ZSET, $token);
}
// 获取当前有效有效的token,并且移除掉其他过期的token
public function getAliveTokensAndDelExpireTokens(){
    $max = time() - self::PAYPAL_TOKEN_EXPIRE;
    $this->redis->zremrangebyscore(self::PAYPAL_TOKEN_ZSET, 0, $max);
    // 剩余的就是有效的
    return $this->redis->zrange(self::PAYPAL_TOKEN_ZSET, 0, -1);
}
```
主要就三个方法，将token入集合， 在集合里面删除token，获取当前有效的token，并去掉过期的token。接下来就是具体的应用了，有两个地方会用到，一个是 return 接口方法的时候：
```html
if ($token) {
    $agreement = new Agreement();
    $this->logger->info("start get agreement: $token ");
    $this->payResult->addPpToken($token);
    try {
        // ## Execute Agreement
        // Execute the agreement by passing in the token
        $agreement->execute($token, $this->apiContext);
    } catch (Exception $ex) {
        $this->logger->error($ex->getMessage());
        $this->returnFail($request);
    }
    $this->logger->info("start get recurringId: $token");
    // NOTE: PLEASE DO NOT USE RESULTPRINTER CLASS IN YOUR ORIGINAL CODE. FOR SAMPLE ONLY
    //ResultPrinter::printResult("Executed an Agreement", "Agreement", $agreement->getId(), $_GET['token'], $agreement);
    // ## Get Agreement
    // Make a get call to retrieve the executed agreement details
    try {
        $agreement = Agreement::get($agreement->getId(), $this->apiContext);
    } catch (Exception $e) {
        $this->logger->error($e->getMessage());
        $this->returnFail($request);
    }
    $this->logger->debug("Agreement Detail:" . $agreement->toJSON());
    // 到达这一步，说明token的使命已经结束了，可以去掉之前的缓存的
    $this->payResult->delPpTokens($token);
    // do something
    return $this->returnSuccess($request);
}
```
分别在请求之前添加，请求成功之后删除。还有一个就是当 ipn 或者 webhook 过来的时候，如果找不到的话，就进行再次请求：
```html
if ($order == null) {
    $this->logger->info("try get recurring again =>" . $orderId);
    // 我们认为这种情况，有可能是之前的获取 paypal 的请求出错了，因此我们将重新将之前获取失败的token重新再请求一次
    $tokens = $this->payResult->getAliveTokensAndDelExpireTokens();
    foreach ($tokens as $token) {
        $this->logger->debug("retry get recurring id: $token");
        $agreement = new Agreement();
        try {
            $agreement->execute($token, $this->apiContext);
            $agreement = Agreement::get($agreement->getId(), $this->apiContext);
        }  catch (Exception $ex) {
            $this->logger->error("$token retry get recurring fail:".$ex->getMessage());
            continue;
        }
        $originRecurringId = $agreement->getId();
        if($orderId === $originRecurringId){
            $this->logger->debug("success get token: $token, Agreement Detail:" . $agreement->toJSON());
            // 到达这一步，说明token的使命已经结束了，可以去掉之前的缓存的
            $this->payResult->delPpTokens($token);
            // do something
            $this->payResult->setPaySuccessful($order->account_id);
            break;
        }else{
            $this->logger->error("not match, skip: $token");
            continue;
        }
    }
}
```
这样子就大大的提高了成功率。事实上，这样做之后，就再也没有出现这种情况了。
## 支付成功，completed 没有过来，只过来 update 状态
最近经常有出现过 paypal 支付成功了，但是却没有升级上来的情况，后面发现原来是没有 **completed** 事件，而是改成 **update** 事件了。
![1](4.png)
**解决方法**:
通过观察我们发现， 当为 update 状态的时候，有一种情况，一种是如果用户没有支付完成的时候，这时候订单号其实是循环单号，而不是真正的订单号，只有当用户支付成功之后，这时候如果 update 状态过来，这时候的订单号才是真正的订单号。
所以我们这么改： 如果是 update 事件，并且当前的 order id 不是循环 id 的话，那么说明就是有效订单，那么就要升级上来， 这时候就不要再去等 completed 的 webhook 了，直接升级, 因为有可能 completed 的 webhook 会非常慢。
```php
//如果是Updated,orderid不等于recurringid,则认为是支付成功的。
if ($transaction->status == 'Updated') {
    if ($orderId == $this->recurringId) {
        return false;
    }
    ... do upgrade
}
```
## 关于 paypal 的循环支付的 return 接口返回两次的问题
之前有出现了一种情况，就是： 在 paypal  的  return  接口的时候， 有时候会发现 paypal 会调用两次。导致会出现两种情况：
- 第一次和第二次相隔的时间会比较长的时候，这时候在第二单的时候， 我的 EC-TOKEN 已经被用掉了，接下来已经找不到对应的订单了
- 第一次和第二次的时间差不多，第二次到的时候，第一次可能还没有执行完。但是这时候第二次就会在请求 agreement 的时候，就会报这个错，结果页面就直接跳到错误页面了

第二种情况就会出现这个 400 错误：
```javascript
{
	"code": 400,
	"msg": "Got Http response code 400 when accessing https:\/\/api.paypal.com\/v1\/payments\/billing-agreements\/EC-64M23731Jxxxxxx\/agreement-execute.",
	"url": "https:\/\/api.paypal.com\/v1\/payments\/billing-agreements\/EC-64M23731Jxxxxxx\/agreement-execute",
	"data": "{\"name\":\"SUBSCRIPTION_UNMAPPED_ERROR\",\"message\":\"A successful Billing Agreement has already been created for this token.\",\"information_link\":\"https:\/\/developer.paypal.com\/docs\/api\/payments.billing-agreements#errors\",\"debug_id\":\"6412fd645a4f3\"}"
}
```
其实处理方式很简单，先把第一次 return 过来的数据，先保存下来（保存到 redis），如果第二个过来的时候，找不到订单的话，直接通过 token 作为 key，从 redis 里面取出数据，然后继续执行接下来的流程。
## 支付成功，completed 没有过来，但是过来 update 状态的问题
最近经常有出现过 paypal 支付了，但是却没有升级上来的情况，后面发现原来是在检查交易记录的时候，发现没有 completed 事件，而是 改成 update 事件了。而且这个 update 事件的 transaction id，有可能是循环单号，也有可能是支付单号。
所以后面如果是收到 updated 的情况，如果当前的订单号，不是循环单号，而是真正的支付单号的话，那么就说明其实用户已经支付成功了，这时候其实就应该升级了。不应该再去等待 completed 过来，因为有可能 completed 过来会非常慢。
代码如下：
```javascript
//如果是Updated,orderid不等于recurringid,则认为是支付成功的。
$orderId = $transaction->getTransactionId();
if ($transaction->status == 'Updated') {
    if ($orderId == $this->recurringId) {
        $this->logger->info("[pay-skip]status=".$transaction->status."; order_id=".$orderId."; recurring_id=".$this->recurringId);
        return false;
    }
    $this->logger->info("[pay-success]status=".$transaction->status."; order_id=".$orderId."; recurring_id=".$this->recurringId);
    // do upgrade handle
}
```
## 因为浏览器插件无法付款的情况
之前客服有反馈了一个问题，就是有一个用户他没法用PayPal付款，点击PayPal后就显示我们产品的支付失败页面。
出现这个页面有两种页面：
- 一种就是有到PayPal的页面去付款，只不过付款失败了，recurringReturn 接口校验的时候，失败了，所以返回这个页面
- 另一种就是根本没有到PayPal的页面去，而是直接返回这个错误页面，这种情况就是浏览器插件屏蔽了PayPal的页面请求，导致直接返回这个页面。

后面查了一下，发现根本请求就没有到我们的服务器，所以是第二种情况，应该是浏览器插件屏蔽了PayPal的页面跳转，所以有让客服跟这个用户说下，让他屏蔽掉浏览器插件再试试



