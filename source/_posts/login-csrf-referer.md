---
title: web 登录接口通过校验 referer 头部来防止 CSRF 攻击
date: 2020-04-11 14:59:43
tags: security
categories: web安全
---
## 前言
前段时间有位好心的白客有发了一封邮件过来，内容如下：
```text
Hi team, 
    
    this time i founded this vulnerability in your website.
    
    Vulnerability 9 : No csrf on login 
    
    Level : critical
    
    Login form is not protected against Cross Site Request Forgery.
    An attacker can craft html page containing POST information to have victim sign into an attacker's account, where the victim can add information assuming he/she is logged into the correct account, where in reality, the victim is signed into the attacker's account where the changes are visible to the attacker.
    
    The real issue here, is that when the victim runs the html Proof of Concept, the account is logged in to attacker's without any visible warnings, thus the victim is capable of theft of data and potentially vulnerable to account takeover.
    ...
```
简单的来说，就是虽然我们其他的业务接口都有 csrf token 的令牌校验，但是那是建立在用户登录后的情况下才下发的。 而恰恰登录接口是没有 csrf 校验，所以具备 CSRF 安全攻击的隐患，让我们赶紧修复。

## 修复问题
之前我们就非常重视 CSRF 的危害性 (具体看 {% post_link web-security %})，所以在用户登录的业务接口中，都会校验下发的 csrf token 令牌。如果校验不对，就认为非法请求。 但是恰恰登录接口是用来下发令牌的，所以这个接口就没有 csrf 令牌校验了。
<!--more-->
当然本身 CSRF 的攻击就有一定难度，所以之前的设想就是当用户登录后的时候，再被别人诱导点击发送请求，因为攻击者不清楚令牌，所以发送的请求，就算 cookie 校验正确，但是令牌校验出错，也会被识别为非法请求。 但是就忽略了登录接口的安全性， 也就是别人诱导点击(别的站点)的时候，也有可能会执行登录接口的触发，并执行登录操作 (可是用户名和密码黑客怎么知道？？ 好奇葩的攻击需求)

不管怎么说，这终究是一个安全隐患，还是得处理，所以就针对登录接口加上了 referer 头部的校验，如果 referer 头部不合法，那么也认为是非法请求：
```text
/**
 * Web 端登录.
 */
public function actionSignin()
{
    // 检查 referer
    $checkResult = CheckService::checkReferer();
    if ($checkResult === false) {
        Response::echoResult(['code' => -1, 'msg' => 'Bad request']);
    }
```
```text
    /**
     * 检查 referer,  默认检查官网 referer
     * @param string $regexp
     * @return bool
     */
    public static function checkReferer($regexp = "")
    {
        $referer  = urldecode(Yii::$app->request->referrer);
        $urlParts = parse_url($referer);
        if(!isset($urlParts['host'])) {
            return false;
        }

        /**
         * referer 需要满足的条件：
         *     1. host 为 *.example.com
         *     2. host 为 example.com
         */
        if (!$regexp) {
            $regexp  = "/.*\.example\.com$/";
        }
        $matches = array();
        preg_match($regexp, $urlParts['host'], $matches);
        if (!$matches && $urlParts['host'] != 'example.com') {
            return false;
        }

        return true;
    }
```

## 总结
关于 CSRF 攻击，还是得警惕。只要有安全隐患，那么就要去解决。








