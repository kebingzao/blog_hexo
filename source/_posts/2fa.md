---
title: 如何在你的程序中启用基于 TOTP 的两步验证
date: 2021-04-01 11:33:22
tags: 
- 2fa
- security
- php
categories: web安全
---
## 前言
我们往往会在不同的网站上使用相同的密码，这样一旦一个网站账户的密码泄露，就会危及到其他使用相同密码的账户的安全。没错, 就是撞库行为, 往往你的站点的安全机制没有问题，但是架不住你的用户使用的密码跟其他被脱库的站点使用的密码一致(现在免费的社工库很多，随便写个简单的暴力碰撞程序(`记得挂上代理`)，就可以试出一堆可以正常使用的用户名和密码), 导致很容易通过撞库的行为从而知道这个用户在你站点的用户名和密码。 为了解决这个问题，一些网站在登录时要求除了输入账户密码之外，还需要输入另一个一次性密码。

而这个输入的一次性密码，其实就是我们所说的两步验证，这不是什么新奇的技术，国内都使用了N多年了，比如说银行用的动态令牌

![1](1.png)

再说一个大家比较有情怀的东西，就是早期网易的将军令(爷春回) 也是用的这个技术:

![1](2.png)

虽然不是什么新技术，不过早期因为手机的不普及，更多的是用硬件的方式来实现 (这玩意儿一旦没有电，就必须重新换一个，因为他没办法联网同步时间，不过这个耗电量极低，一个可以用好几年)。 但是随着手机的普及，以及大家的安全意识的提升。 越来越多的站点都用软件的方式来实现(比如 Evernote， Google， 以及我所在团队的站点)，而且成本很低。

<!--more-->

## 两步验证原理
两步验证，对应的英文是 `Two-factor Authentication（2FA）`,从名字可以看出，「两步」是 2FA 的重点，也就是 密码 + 一次性密码（One Time Password，OTP）

### OTP
2FA 中使用的是一次性密码（One Time Password，OTP），也被称作动态密码。一般 OTP 有两种策略：HOTP ( HMAC-based One Time Password) 和 TOTP ( Time-based One-time Password) 。目前被广泛使用的正是后者这种基于时间的动态密码生成策略。

### HOTP 的原理
> 本节的原理来自 [Google账户两步验证的工作原理](https://blog.seetee.me/post/2011/google-two-step-verification/)

虽然实践中更多的是使用 TOTP 来进行两步校验， 但是它是基于 HOTP 来实现的， 所以我们先了解一下 HTOP 的工作原理。

1. 客户端和服务器事先协商好一个密钥K，用于一次性密码的生成过程，此密钥不被任何第三方所知道。此外，客户端和服务器各有一个计数器C，并且事先将计数值同步。

2. 进行验证时，客户端对密钥和计数器的组合(K,C)使用HMAC（Hash-based Message Authentication Code）算法计算一次性密码，公式如下：
```text
HOTP(K,C) = Truncate(HMAC-SHA-1(K,C))
```
上面采用了 `HMAC-SHA-1`，当然也可以使用`HMAC-MD5`等。HMAC 算法得出的值位数比较多，不方便用户输入，因此需要截断（Truncate）成为一组不太长十进制数（例如6位）。计算完成之后客户端计数器C计数值加1。

3. 用户将这一组十进制数输入并且提交之后，服务器端同样的计算，并且与用户提交的数值比较，如果相同，则验证通过，服务器端将计数值C增加1。如果不相同，则验证失败。

这里的一个比较有趣的问题是，如果验证失败或者客户端不小心多进行了一次生成密码操作，那么服务器和客户端之间的计数器C将不再同步，因此需要有一个重新同步（Resynchronization）的机制。这里不作具体介绍，详情可以参看 [RFC 4226](http://tools.ietf.org/html/rfc4226)。

### TOTP 的原理
> 本节的原理来自 [Google账户两步验证的工作原理](https://blog.seetee.me/post/2011/google-two-step-verification/)

介绍完了HOTP，Time-based One-time Password（TOTP）也就容易理解了。TOTP 将 HOTP 中的计数器C用当前时间T来替代，于是就得到了随着时间变化的一次性密码。非常有趣吧！

虽然原理很简单，但是用时间来替代计数器会有一些特殊的问题，这些问题也很有意思，我们选取几个进行一下探讨。

首先，时间T的值怎么选取？ 因为时间每时每刻都在变化，如果选择一个变化太快的T（例如从某一时间点开始的秒数），那么用户来不及输入密码。如果选择一个变化太慢的T（例如从某一时间点开始的小时数），那么第三方攻击者就有充足的时间去尝试所有可能的一次性密码（试想6位数字的一次性密码仅仅有10^6种组合），降低了密码的安全性。除此之外，变化太慢的T还会导致另一个问题。如果用户需要在短时间内两次登录账户，由于密码是一次性的不可重用(这个其实可以配置)， 用户必须等到下一个一次性密码被生成时才能登录，这意味着最多需要等待59分59秒！这显然不可接受。综合以上考虑，Google选择了30秒作为时间片，T的数值为从 Unix epoch（1970年1月1日 00:00:00）来经历的30秒的个数。

第二个问题是，由于网络延时，用户输入延迟等因素，可能当服务器端接收到一次性密码时，T的数值已经改变，这样就会导致服务器计算的一次性密码值与用户输入的不同，验证失败。解决这个问题个一个方法是，服务器计算当前时间片以及前面的n个时间片内的一次性密码值(这个其实也可以配置)，只要其中有一个与用户输入的密码相同，则验证通过。当然，n不能太大，否则会降低安全性。

### TOTP 的算法和特点
综上所述， TOTP 的算法大体是这样子:
1. 客户端和服务器事先协商好一个SECRET，用于一次性密码的生成过程，此密钥不被任何第三方所知道。此外，客户端和服务器都采用时间做计数器。
2. 客户端对密钥和计数器的组合(SECRET,time/30) 使用HMAC（Hash-based Message Authentication Code）算法计算一次性密码，公式如下：`HMAC-SHA-1(SECRET, time/30)`
3. 各种算法加特效后成6位数字

基于 TOTP 的密码有如下特点
1. 无需记忆，不会产生 password 这样的泄漏问题
2. 动态生成,每30s生成一个，安全性大大提高
3. 对网络无要求，离线下仍可正常使用
4. 成本低，无需购买硬件和软件 (只需要安装一个客户端就行了，这种市面上很多)

## 两步验证流程实现
算法和原理搞懂了，接下来讲讲流程:
1. 服务端随机生成一个类似于`DPI45HKISEXU6HG7` 的密钥，并且把这个密钥保存在数据库中。
2. 在页面上显示一个二维码，内容是一个URI地址（`otpauth://totp/厂商:账号?secret=密钥&issuer=厂商`）
3. 客户端扫描二维码，把密钥`DPI45HKISEXU6HG7`保存在客户端。
4. 客户端每30秒使用密钥`DPI45HKISEXU6HG7` 和时间戳通过 TOTP 算法 生成一个6位数字的一次性密码
5. 用户输入这个一次性密码，然后服务端进行验证

### 客户端的选择
要实现上述两步验证的流程，我们需要一个客户端 (Google Authenticator compatible app)，这个客户端我们不需要直接去开发，市面上很多:
- [Authy for iOS, Android, Chrome, OS X](https://authy.com/)
- [Google Authenticator for iOS](https://itunes.apple.com/us/app/google-authenticator/id388497605?mt=8)
- [Google Authenticator for Android](https://play.google.com/store/apps/details?id=com.google.android.apps.authenticator2)
- [Google Authenticator (port) on Windows Store](https://www.microsoft.com/en-us/store/p/google-authenticator/9wzdncrdnkrf)
- [1Password for iOS, Android, OS X, Windows](https://1password.com)

比如我装的是 `Google Authenticator for iOS`, 打开是这样子的 (因为我已经有使用了)

![1](3.png)

上面有两条记录，说明我在两个站点上，有开启了两步验证，然后这边就会根据 TOTP 算法，每隔 30s 就会刷一次 code 。

如果要扫码的话，直接选择右下角的 "+" 按钮，然后选择 `扫描二维码` 即可

### 流程的效果图
有了流程图之后，配合效果图，就很好理解了。

1. 首先当用户选择两步验证的时候，就要提示他要立即绑定 谷歌身份验证器 

![1](4.png)

2. 接下来前端就请求服务端得到 OTP 的绑定内容(`otpauth://totp/厂商:账号?secret=密钥&issuer=厂商`)，以二维码的方式显示

![1](5.png)

3. 谷歌身份验证器扫码之后，就会绑定这个站点的两步验证信息(其实就是保存密钥，厂商，账号等信息)，同时生成 code

4. 最后再输入刚才生成的 6 位数的 code，然后前端传到服务端去验证，如果验证通过之后，那么就绑定成功了。

5. 这样子用户下次登录的时候，服务端检测到用户有开启两步验证，那么就弹出框让其输入 OTP 码， 然后验证，如果验证通过，才进入真正的产品界面

![1](6.png)

这样子，一个简单的两步验证的流程就完成了。 客户端不需要我们自己处理， 前端只需要请求接口就行了， 主要是服务端的逻辑， 要负责下发 OTP 绑定内容给客户端扫码绑定，又要负责校验。 所以我们接下来讲一下服务端要怎么处理。

### 服务端的处理方式
我们用的是 PHP 来处理，而且用的是一个第三方包 [antonioribeiro/google2fa](https://github.com/antonioribeiro/google2fa), 这个包的功能非常全面，我们要的功能都有，而且不仅仅是 TOTP 验证，他也提供 HOTP 验证。

首先用 composer 安装一下这个包:
```text
composer require pragmarx/google2fa
```

#### 1. 生成 secret key
首先要先生成 secret key， 然后保存到这个用户所在的信息表中 (不能跟已存在的重复，如果重复，就重新生成一个)
```text
use PragmaRX\Google2FA\Google2FA;
    
$google2fa = new Google2FA();
    
return $google2fa->generateSecretKey();
```

这个密钥默认是 16 位，这个其实够用了，如果需要安全性更高的，那么也可以设置为 32 位
```text
$secretKey = $google2fa->generateSecretKey(32); // defaults to 16 bytes
```

#### 2. 生成二维码的 OTP 绑定内容
接下来就要生成 OTP 的绑定内容， 用来做二维码的内容显示:
```text
otpauth://totp/厂商:账号?secret=密钥&issuer=厂商
```
这个也有 api:
```text
$qrCodeUrl = $google2fa->getQRCodeUrl(
    $companyName,
    $companyEmail,
    $secretKey
);
```
这样子就生成了， 不过要注意一点的是，这个只是扫描二维码之后的内容。 只是一个字符串而已，并不是二维码本身。 如果前端有自己的生成二维码的程序的话，那么只需要将这个返回给前端就行了。 如果前端没有的话，那么服务端也可以直接生成一张包含这个内容的二维码图片，然后传给前端， 前端直接显示这种图片即可。

这边需要注意一个细节，就是这个包在新版的时候，已经将生成二维码图片的功能去掉了，如果你要生成二维码的话，那么就要结合其他的第三方包，比如 [Bacon/QRCode](https://github.com/Bacon/BaconQrCode) 这个包来生成:
```text
use PragmaRX\Google2FA\Google2FA;
use BaconQrCode\Renderer\ImageRenderer;
use BaconQrCode\Renderer\Image\ImagickImageBackEnd;
use BaconQrCode\Renderer\RendererStyle\RendererStyle;
use BaconQrCode\Writer;

$google2fa = app(Google2FA::class);

$g2faUrl = $google2fa->getQRCodeUrl(
    'pragmarx',
    'google2fa@pragmarx.com',
    $google2fa->generateSecretKey()
);

$writer = new Writer(
    new ImageRenderer(
        new RendererStyle(400),
        new ImagickImageBackEnd()
    )
);

$qrcode_image = base64_encode($writer->writeString($g2faUrl));
```

不过这个包的旧版本，比如 `4.0.0` 版，还是有提供这个生成二维码图片地址的功能的:
```text
$google2fa->->setAllowInsecureCallToGoogleApis(true);
$google2fa->->getQRCodeGoogleUrl($company, $userEmail, $secret, $size);
```
看了一下他的源码，他是直接封装了 google chart 的 api 来处理的:
```text
    public function getQRCodeGoogleUrl($company, $holder, $secret, $size = 200)
    {
        if (!$this->allowInsecureCallToGoogleApis) {
            throw new InsecureCallException("It's not secure to send secret keys to Google Apis, you have to explicitly allow it by calling \$google2fa->setAllowInsecureCallToGoogleApis(true).");
        }

        return Url::generateGoogleQRCodeUrl(
            'https://chart.googleapis.com/',
            'chart',
            'chs='.$size.'x'.$size.'&chld=M|0&cht=qr&chl=',
            $this->getQRCodeUrl($company, $holder, $secret)
        );
    }
```
这个在国内是访问不了的。 可能也是考虑到其他原因吧， 后面新版本就把这个东西去掉了。

#### 3. OTP 验证
当 谷歌身份验证器扫码之后， 为了让服务端知道已经有扫码绑定了， 这时候就需要第一次将 OTP 输入，然后到服务端进行校验，如果校验成功，那么就说明流程成功。 所以这边会涉及到 验证码的校验, 这边也是有直接的 api 可以处理
```text
$secret = $request->input('secret');

$valid = $google2fa->verifyKey($user->google2fa_secret, $secret);
```
其中第一个参数就是我们生成存放到数据库的密钥， 第二个参数才是前端传过来的 一次性验证码。


### 几个注意事项
现在我们已经将整个两步验证的流程都串起来了。 不过有几个注意事项要记得处理:

#### 1. OTP 不能复用
生成的 OTP 应该是不能复用的，也就是用户在登陆或者危险操作时输入完一次 OTP 后，手机端 OTP 仍然未刷新时，在进行登陆或者危险操作时输入刚才的 OTP 是无效的，必须等待手机上 OTP 的刷新。

这个包是有 api 支援的, 换成 `verifyKeyNewer` 这个 api 就行了
```text
$secret = $request->input('secret');

$timestamp = $google2fa->verifyKeyNewer($user->google2fa_secret, $secret, $user->google2fa_ts);

if ($timestamp !== false) {
    $user->update(['google2fa_ts' => $timestamp]);
    // successful
} else {
    // failed
}
```

#### 2. 设置足够时间的窗口期
既然可以离线使用，那么怎么保证时间的差异性，我们服务端会兼容服务器时间的前后30s。从而有效的避免细微时间上差异而导致的验证失败，同时也避免了用户刚输入完 OTP 后还未做提交操作时 OTP 刷新而引起验证失败。

这个包的 api 也是有的，他可以设置验证时间的窗口期，默认是延长 1 个窗口期， 也就是这个 OTP 的有效时间是 60s:
```text
 protected $window = 1; // Keys will be valid for 60 seconds
```
当然我们也可以显示的设置:
```text
$window = 8; // 8 keys (respectively 4 minutes) past and future

$valid = $google2fa->verifyKey($user->google2fa_secret, $secret, $window);
```

#### 3. 防止暴力破解 OTP
在遇到使用遍历所有6位数数字进行暴力破解 OTP 时，我们要对错误次数进行限制，超过一定的错误次数，那么就要进行限制。

#### 4. 要重新登录
在开启两步认证后，其他所有登陆的客户端都会因为开启两步认证而过期，必须重新登陆。

#### 5. 客户端绑定的时候，允许直接输入密钥进行绑定
有时候客户会扫不了二维码，这时候我们要提供 密钥，允许直接在客户端输入账号和密钥进行绑定， 这个在 google 身份验证器的时候，也是可以操作的

![1](7.png)

#### 6. 预留备用 code 行为
因为有时候会出现当前手机上没有安装 谷歌身份验证器， 但是又需要登录我们的站点。 因此我们站点如果有开启两步验证之后，同时也会预留一组备用码。 当身份验证器不在身边的时候，也可以通过输入备用码进行登录， 不过这个备用码也是一次性的，一旦用完之后， 只能在我们产品的后台重新刷新生成。

![1](8.png)

## 总结
以上就是在项目实践中实现两步验证的流程了。 总的来说不难，不过需要注意，因为 TOTP 是需要客户端和服务端的时间同步的，如果客户端直接将时间调快的话，那么验证就会出错的。


---
参考文档:
- [How does Google Authenticator work?](https://security.stackexchange.com/questions/35157/how-does-google-authenticator-work)
- [Google账户两步验证的工作原理](https://blog.seetee.me/post/2011/google-two-step-verification/)
- [Google Two-Factor Authentication for PHP](https://github.com/antonioribeiro/google2fa)
- [Coding 两步认证技术介绍](https://blog.coding.net/blog/two-factor-authentication)
- [Linux 之 利用Google Authenticator实现用户双因素认证](https://www.cnblogs.com/hanyifeng/p/5516053.html)
- [谷歌验证 (Google Authenticator) 的实现原理是什么？](https://www.zhihu.com/question/20462696)