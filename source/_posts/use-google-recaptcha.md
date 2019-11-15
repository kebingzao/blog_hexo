---
title: 使用"Google reCaptcha"来防止站点接口被刷
date: 2019-11-15 15:17:33
tags: security
categories: web安全
---
## 前言
前段时间有收到一封来自国外安全公司的关于安全的预警邮件，大致的意思就是 他发现我们站点的 **忘记密码，发送验证邮件** 的功能并没有做防刷限制，导致他可以用**burp suite** 之类的web安全工具来批量刷接口，或者是自己写脚本来批量刷。 这样就会导致我们发送大量的垃圾邮件，这样子不仅会打扰我们的用户，而且还会导致我们发邮件的费用增加，更严重的可能会导致我们的 SES 账号被 AWS 封掉，具体之前有出现过：{% post_link aws-ses-block-again %}

## 解决
因为这个接口的特殊性(它不需要用户登录，所以也没有所谓的 cookie，session 验证，更没有 jwt token 之类的身份验证), 所以在排除掉 身份验证之后，只能通过其他的方式来处理：
<!--more-->

### 1.在网关那边加上防刷接口
首先要考虑的是加上接口防刷验证，比如 `nginx` 网关那边加上针对这个接口的防刷限制，比如一分钟之内，同一个 ip 访问超过 10 次的话，那么就直接返回 `403`。 同时将这个 ip 加入到黑名单， 24 小时之后解禁。

### 2. 加上验证码
原则上来说，ip 防刷机制是可以解决 90% 的刷接口的问题的，但是有时候会误杀，比如普通用户的多次输错邮箱之类的多次尝试，可能就会因为尝试太多，而被我们误杀。还有一种更普遍误杀的情况就是，很多时候，某一个区域(比如学校，公司)的出口 ip 其实是同一个，如果在这个区域内，同时有多个使用者一起使用这个功能的话，那么就会误杀了。 所以更友好的方式，就是加上`验证码`, 用户在提交请求的时候，要加上验证码，服务端那边才会通过请求。 这样就不会有误杀问题，而且也不怕被刷接口。但是验证码有以下问题：

- 验证码体验会比较不好，毕竟需要用户识别并输入
- 验证码机制如果不够强，也很容易被攻破，尤其是现在图片识别机器人和语音识别机器人其实很厉害了，差的验证码机制很容易就被破了，形同虚设

所以先排除自己做的可能性(无论是人力还是时间成本，性价比都很低)，考虑了一下市面上成熟的验证码产品，最后选择了 [Google reCaptcha](https://www.google.com/recaptcha/intro/v3.html), 接下来我们来了解一下。

## Google reCaptcha
国外很多网站都用这个这个服务，开发文档：[recaptcha intro](https://developers.google.com/recaptcha/intro)
### 1 Google reCaptcha 介绍
简单的来说，`google reCaptcha` 目前有3个版本：
#### reCaptcha v1
图形验证码，需要用户输入图片里的文字 (目前已经是 shutdown 状态，无法创建该类型 reCaptcha)  

![](rev1.jpg) 


####  reCaptcha v2  
如果我们选择使用 `reCaptcha v2`, 我们可以为我们的站点创建以下 `1`、`2`、`3` 这3种验证类型中的其中一种。当 google 认为你可能是一个 robot 的时候，会出现 `4`.的情况，要求你进行进一步验证。

##### 1 checkbox 
需要用户勾选 `I'm not robot` 复选框  

![](rev2.jpg) 

##### 2 invisible 

用户无需做任何操作。前端需要使用 google 提供的 js api, 把相关事件绑定到 “提交” 按钮（或其他事件）上。相当于原本的 `"i'm not robot"` 复选框勾选事件可以替换成自定义的 `onSubmit` 或者其他事件。

![](rev3.jpg) 

invisible 下的 `image challenge`  : 

![](revinvisible.jpg) 

这时候google 那边会有一个蒙层，上面用一个 `iframe` 嵌入进入，不会影响到我们原来的 ui 元素

##### 3 reCAPTCHA Android  

安卓设备中的 reCaptcha 验证，这块还没研究。后面如果有应用，再补上

##### 4 进一步验证：image challenge 

`v2 reCaptcha` 在以下几种情况，会要求进行进一步验证(v3不会)：

  - 过于频繁：一定时间内同一个 ip 频繁重试这个 `i'm not a robot` （在 v2 invisible 下，频繁触发对应事件）。
  - `recaptcha js` 初始化之后很久才去进行 recaptcha 验证。
  - 在浏览器种勾选 `阻止第三方 cookie`
  - 一些异常鼠标或者滚动条行为, 比如没有监测到鼠标的 `movement` 就点击了勾选复选框（不知道怎么重现）

![](revchanllenge.png)  

参考：

  - [ReCaptcha always asking for extra verification?
](https://stackoverflow.com/questions/30524505/recaptcha-always-asking-for-extra-verification)
  - [Way to skip reCAPTCHA images challenge
](https://stackoverflow.com/questions/45754869/way-to-skip-recaptcha-images-challenge) 
  - [google-one-click-recaptcha](https://www.wired.com/2014/12/google-one-click-recaptcha/)


#### reCaptcha v3  
无需用户进行任何操作。根据用户的各种行为以及在浏览器上的各种事件，评出一个 0.0 ~ 1.0 之间分数(score)，待服务端请求 api 去验证的时候把 score 交给服务端，至此 reCaptcha 的工作就完成了。服务端根据得到的 score 结合具体场景，自行处理后面的业务。

**v3 不会出现 image challenge**。


![](rev3.jpg) 


### 2 reCaptcha 接入须知 
#### 接入建议 
建议使用 `v2` ，因为 `v3` 存在已知并且未 close 的问题：

  - https://github.com/google/recaptcha/issues/235
  - https://github.com/google/recaptcha/issues/248

有用户反馈突然发生所有的 score 都变成 0.1 分的情况(包括真实用户测试)。我在自测的时候出现了这种情况：在 chrome 下验证总是得到 0.9 分，但是在 safari 下得总是得到 0.1 分。 

#### 浏览器支持情况  
基本上现代的浏览器都支持

详情见  https://support.google.com/recaptcha#6223828 。

#### 国内用户访问问题 

由于 `reCaptcha` 接入需要前端载入 google.com 域名下的 js 文件，所以国内用户可能无法正常加载 reCaptcha。 但是 google 提供了`Globally accessible` 的域名： www.recaptcha.net ，只要把对应的资源 url 中的 www.google.com 换成 www.recaptcha.net 就可以使用 `reCapatcha` 的全部功能。

参考：[reCaptcha faq](https://developers.google.com/recaptcha/docs/faq) 

#### 安全设置问题

![](revsetting.jpg) 

安全偏好设置有 3 个可选选项，不同选项不知道具体有哪些区别，以及对用户的使用会有哪些影响。目前暂时没有找到相关资料，自测起来好像也没有什么区别。
不过后面在多次测试过程中，在接入 `google recaptcha` 的时候，发现选用介于 `用户使用最方便` 和 `最安全` 中间的这个等级，会导致 `image challenge` 频繁出现(基本每次都出现)。 所以我们选用 `用户使用最方便` 这个等级。

#### 已知问题 

  - [IE 下可能需要用户手动设置处理无法显示 reCaptcha 问题](https://support.google.com/recaptcha)

### 3 开发流程 

#### 创建 reCaptcha 

见 [recaptcha admin](https://www.google.com/recaptcha/admin/create) ， 注意，创建时最好勾选 `验证 reCAPTCHA 解决方案来源`。
标签随便填一个好认的，然后域名填写你要验证的前端站点域名，最后创建成功之后会生成：

1. site key (用于前端嵌入页面)  
2. secret key (用于服务端调用 `recaptcha api` 验证)

![](1.png) 

![](2.png) 

所以虽然有填写要验证的域名，但是其实并不需要去验证这个域名是不是你持有的，因为有成套的 `key` 会验证。如果前后端的 `key` 不匹配，那么就会验证失败。

#### 前端接入 

前端根据文档接入 reCaptcha 之后，在提交请求的时候会带上一个 `g-recaptcha-response` 参数。见 [reCaptcha 接入文档](https://developers.google.com/recaptcha/docs/display)。

```html
<html>
  <head>
    <title>reCAPTCHA demo: Simple page</title>
    <script src="https://www.google.com/recaptcha/api.js" async defer></script>
  </head>
  <body>
    <form action="?" method="POST">
      <div class="g-recaptcha" data-sitekey="your_site_key"></div>
      <br/>
      <input type="submit" value="Submit">
    </form>
  </body>
</html>
```
当然这个是最简单的实例，很多时候我们并不会在 html 的页面加载，而是另一种异步加载，比如真实使用中，我们的代码如下：
```html
    <form id="form_resend">
        <div id='recaptcha' class="g-recaptcha" data-sitekey="" data-callback="grecaptchaCallback"
            data-size="invisible"></div>
        <label for="email">请输入您的邮箱，我们将会发送一封重置密码的邮件到该邮箱</label>
        <input id="email" type="email" name="mail" placeholder="邮箱" autocomplete="off" />
        <button id="submit" class="button-primary" name="submit" value="submit"
            type="submit">发送邮件</button>
    </form>
```
然后初始化的时候，在 js 代码里面进行加载：
```javascript
$('#recaptcha').attr('data-sitekey', "xxxxxxxxxxxxxxxxxxxx")
window.grecaptchaCallback = function(token){
    // 发送给服务端
    self.server.sendResetpwMail({
        mail: $("#mail").val().trim(),
        g_token:token
    }).done(function (resp) {
        // success
    }).fail(function () {
        // fail
    }).always(function () {
        // 重置状态
        grecaptcha.reset();
        // do something tip
    });
};
var script = document.createElement("script");
script.type = "text/javascript";
script.src = "https://www.recaptcha.net/recaptcha/api.js";
script.async = 'async';
script.defer = 'defer';
document.body.appendChild(script);
```
这样子逻辑就好了，接下来就是绑定 `form` 的 `submit` 事件，然后触发 `grecaptcha.execute()` 即可，这时候如果验证通过的话，就会触发`grecaptchaCallback` 这个全局对象，然后把 `token` 传给前端，然后前端再连同 `token` 和 `key`，一起传给服务端进行校验。
```javascript
grecaptchaExecute:function(e){
    e.preventDefault();
    if (!this.validEmail()) return false;
    grecaptcha.reset();
    grecaptcha.execute();
}
```

#### 服务端验证 
服务端接受前端请求中的 `g-recaptcha-response` 参数，然后调用 reCaptcha 的验证接口，根据验证接口返回值判断验证是否通过。

如果在创建 `reCaptcha` 时没有勾选 `验证 reCAPTCHA 解决方案来源`，任何人都可以从你站点的 html 表单或者 js 里拿到 site key ，来嵌入到自己的网站(hack.com) 然后自己提交，并得到正确的 `g-recaptcha-response`。然后用得到的 response token 去你的接口进行伪造提交。如果服务端这时候去调用 reCaptcha 验证接口，返回的结果是通过的。解决方式为服务端不仅要验证“是否通过”，还要验证 “来源”。

如果在创建 `reCaptcha` 时勾选了 `验证 reCAPTCHA 解决方案来源`，攻击者仍然可以拿到我们网站 site key 并得到正确 `g-recaptcha-response`，然后提交到我们的服务端。只是服务端在调用 reCaptcha 接口时会直接判断为不通过，返回 `incorrect-captcha-sol`。

见 [服务端验证](https://developers.google.com/recaptcha/docs/verify)。 

具体的代码为:
```php
            // 来自官网请求，则进行 google recaptcha token 验证。 若校验失败，则直接结束请求。
            $gtoken = Request::getParam('g_token');
            $isGTokenValid = Request::checkGoogleRecaptcha($gtoken);
            if (! $isGTokenValid) {
                LogService::logWarning("mail : $mail, gtoken invalid : $gtoken");
                Yii::$app->end();
            }
```
然后进行 token 的校验:
```php

/**
 * 发送一个 post 请求并返回结果
 * @param $url
 * @param array $formData 表单格式
 * @param int $timeout
 * @return bool|string
 */
public static function httpPost($url, $formData = [], $timeout = 5)
{
    LogService::logInfo("Guzzle Request Post Url : $url");
    $client = new Client(['timeout' => $timeout]);

    try {
        $response = $client->request('POST', $url, [
            'form_params' => $formData,
        ]);
    } catch (\Exception $e) {
        LogService::logError("Guzzle Exception : $url , " . $e->getMessage());
       return false;
    }

    return $response->getBody()->getContents();
}

 /**
 * 校验 google recaptcha token (该方法目前仅支持 v2 recaptcha)
 * @param $token
 * @return bool
 */
public static function checkGoogleRecaptcha($token = '')
{
    $formData = [
        'secret' => yiicfg('googleRecaptchaSecretKey'),
        'response' => $token,
    ];

    $resp = Request::httpPost(yiicfg('googleRecaptchaApiUrl'), $formData);

    // 如果请求 recaptcha api 异常，则本次判校验为通过
    if ($resp === false || is_string($resp) === false) {
        return true;
    }

    $jsonData = json_decode($resp, true);
    if (isset($jsonData['success']) && $jsonData['success'] == false ) {
        $errcode = isset($jsonData['error-codes']) ? $jsonData['error-codes'] : [];
        LogService::logWarning("gtoken invalid : $token, error : " . json_encode($errcode));
        return false;
    }

    return true;
}
```

这样子服务端的校验就完成了。

## 总结
其实对于大部分的正常用户来说，基本上就是不需要强制的图片验证码，所以使用体验几乎跟之前没有差别，只有刷接口或者是频繁操作页面的使用者，这时候 google 会判断是否需要显示图片验证码。

最后感谢新隆同学，这里面的很多细节调研都是他去处理和整理的，所以才有这篇文章的产出。








