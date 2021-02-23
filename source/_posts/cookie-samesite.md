---
title: chrome 默认 cookie 的 SameSite=Lax，导致 http 模式的站点的第三方 cookie 无法进行跨域传输
date: 2021-02-22 18:01:30
tags:  js
categories: 前端相关
---
## 前言
之前有用户遇到一个问题，就是在访问一个我们的站点的时候，https 模式下没有问题，但是用 http 模式下(因为某些特殊原因，要保留站点的 http 模式的访问)，就会出现接口报错的情况，导致用不了。

而且都是 chrome 才有， 其他浏览器都正常， 而且 chrome 的版本还是比较新的， 87 或者 88。 我自己本身 86 好像挺正常的。 为了验证一下，也将 chrome 版本也升级到 88， 结果还真重现了。

## SameSite 的问题
发现是登录接口的时候，原本要写进去的 cookie 虽然写了，但是再访问其他接口的时候，这些 cookie 根本不会带过去， 这样就导致没办法进行 cookie 校验了。 因为 cookie 在跨域的时候，并不会随着请求过去。
<!--more-->

![1](1.png) 

看样子是 chrome 浏览器将这些 cookie 的属性都加上了 `SameSite=Lax` 这个属性，导致在进行跨域请求的时候，这些 cookie 不会跟着传输

后面查了一下资料 [Cookie Legacy SameSite Policies](https://www.chromium.org/administrators/policy-list-3/cookie-legacy-samesite-policies), 好像 chrome 打算在 80 版本的时候，就打算将未显式设置 SameSite 属性的第三方 cookie 都补上 `SameSite=Lax` 这个属性 (原先默认是 `SameSite=None`)， 不过好像因为疫情的原因，又推迟了。 现在好像又推进了流程。

而且经过我的测试发现，同样是chrome的最新版本 88， 但是有些浏览器就会这个机制， 但是有些浏览器就不会，还是原来的机制。 所以我怀疑这个功能应该是 A/B 测试的。

后面查了一下，确实是这样子:

{% blockquote SameSite Cookie 策略 https://www.cnbeta.com/articles/tech/984703.htm %}
谷歌宣布将从7月14日发布的Chrome 84稳定版开始，重新恢复SameSite cookie策略，并且会逐步部署到Chrome 80以及以上的版本中。
Chrome 80于今年2月份上线，谷歌就开始滚动推出SameSite更新，通过cookies的发送机制进行了一系列新的调整更好的维护用户隐私和安全。
今年4月份鉴于全球疫情的爆发，谷歌宣布暂时中止该更新，以便在COVID-19大流行期间保持重要网站的正常运行。
{% endblockquote %}

## SameSite 属性详情
我们先来看看这个属性的作用：

SameSite 属性可以让 Cookie 在跨站请求时不会被发送，从而可以阻止跨站请求伪造攻击（CSRF）。

属性值
SameSite 可以有下面三种值：

- `Strict` 仅允许一方请求携带 Cookie，即浏览器将只发送相同站点请求的 Cookie，即当前网页 URL 与请求目标 URL 完全一致。
- `Lax` 允许部分第三方请求携带 Cookie
- `None` 无论是否跨站都会发送 Cookie

之前默认是 None 的，Chrome80 后默认是 Lax。

### 跨域和跨站
那么问题来了， 什么是 `允许部分第三方请求携带` ? 有没有规则 ? 这时候就要搞清楚，对于 cookie 来说， 什么叫做第三方请求 ?

首先要理解的一点就是跨站和跨域是不同的。`同站(same-site)`/`跨站(cross-site)` 和 `第一方(first-party)`/`第三方(third-party)`是等价的。但是与浏览器同源策略（SOP）中的`同源(same-origin)`/`跨域(cross-origin)` 是完全不同的概念。

同源策略的同源是指两个 URL 的协议/主机名/端口一致。例如，`https://www.foo.com/pages/...`，它的协议是 `https`，主机名是 `www.foo.com`，端口是 443。

同源策略作为浏览器的安全基石，其`同源` 判断是比较严格的，相对而言，Cookie中的 `同站` 判断就比较宽松：只要两个 URL 的 `eTLD+1` 相同即可，不需要考虑协议和端口。其中，`eTLD` 表示有效顶级域名，注册于 Mozilla 维护的公共后缀列表（Public Suffix List）中，例如，`.com`、`.co.uk`、`.github.io` 等。`eTLD+1` 则表示，有效顶级域名+二级域名，例如 `taobao.com` 等。

举几个例子，`www.taobao.com` 和 `www.baidu.com` 是跨站，`www.a.taobao.com` 和 `www.b.taobao.com` 是同站，`a.github.io` 和 `b.github.io` 是跨站(注意是跨站)。

不过这边在实践中有个问题，就是虽然域名不一样，但是 `eTLD+1` 是一样的，比如一个 a.foo.com (前端站点) 和 b.foo.com (ajax 的服务端请求站点)，但是这两个域名设置的 cookie 的 domain 都是 `.foo.com`, 所以在 cookie 的判断中，这个是属于 同站的，但是又不符合 `Strict` 的严格要求，因为他要求两者的 url 要一致。  然后经过我的实践，发现在这种情况下的 `SameSite=Lax` 的策略，如果是在前端站点 https 的全站下访问， 那么 cookie 是可以携带的， 但是如果是 http 模式下的前端站点，即使你访问的服务端站点是 https 模式，那么还是会被限制。 (这个就是我上述前言说的情况)

## 修复措施
一般这种情况下有以下几种方式可以修复
1. 将前端站点改为 https 的方式，这样子设置 domain 为 `.foo.com` 的同站 cookie， 再进行 ajax 传输的时候， 也是可以传输的。
2. 不采用 cookie 进行校验的方式，而是将其换成 token 作为参数来校验
3. 将 cookie 的策略改成 `SameSite=None` 允许跨站 cookie 传输

其中第一点因为要保留 http 模式的业务请求，所以不考虑。 换成 token 的话，成本太高，要修改多个项目，也暂时不考虑。 所以我们考虑用第三点的方式来处理。

### 设置 `SameSite=None`
不过这边要注意两点:
#### 1. 要加 secure 属性
如果你想加 `SameSite=none` 属性，那么该 Cookie 就必须同时加上 `Secure` 属性，表示只有在 HTTPS 协议下该 Cookie 才会被发送。 这个倒是没问题，因为我们的前端站点虽然支持 http 协议，但是请求的服务端接口肯定都是 https 协议的， 所以是满足的。

#### 2. 需要 UA 检测，部分浏览器不能加 SameSite=none
IOS 12 的 Safari 以及老版本的一些 Chrome 会把 SameSite=none 识别成 SameSite=Strict，所以服务端必须在下发 Set-Cookie 响应头时进行 User-Agent 检测，对这些浏览器不下发 SameSite=none 属性。

这边具体可以看: [SameSite = None：已知的不兼容客户端](https://www.chromium.org/updates/same-site/incompatible-clients)

{% blockquote https://www.chromium.org/updates/same-site/incompatible-clients %}
已知某些用户代理与`SameSite = None'属性不兼容。
- Chrome的版本，从Chrome 51到Chrome 66（包括两端）。这些Chrome版本将拒绝带有“ SameSite = None”的Cookie。这也会影响Chromium衍生的浏览器的旧版本以及Android WebView 。根据当时的Cookie规范版本，此行为是正确的，但是在规范中添加了新的“ None”值后，此行为已在Chrome 67和更高版本中进行了更新。（在Chrome 51之前，SameSite属性被完全忽略，所有cookie都被视为“ SameSite = None”。）
- Android 12.1.3.2。之前的UC浏览器版本。较旧的版本将拒绝带有“ SameSite = None”的cookie。根据当时的cookie规范版本，此行为是正确的，但是在规范中添加了新的“ None”值后，此行为已在较新版本的UC浏览器中进行了更新。
- Safari和MacOS 10.14上的嵌入式浏览器以及iOS 12上的所有浏览器的版本。这些版本将错误地将标记为“ SameSite = None”的cookie视为标记为“ SameSite = Strict”的cookie。此错误已在较新版本的iOS和MacOS上修复。
{% endblockquote %}

而且他这边还给我们提供了一个示例代码，我转换为 php 代码就是:
```php
/**
 * Cookie 工具类, 不过这边不进行 cookie 的设置和获取， 这个有另外的 php 内置函数处理
 * 这边主要是针对 cookie 的一些属性设置的一些判断
 */
class Cookie
{
    // 判断是否支持 same site 为 none 的配置
    // 参考文档 https://www.chromium.org/updates/same-site/incompatible-clients
    public static function shouldSendSameSiteNone($userAgent){
        return !self::isSameSiteNoneIncompatible($userAgent);
    }
    // Classes of browsers known to be incompatible.
    private static function isSameSiteNoneIncompatible($userAgent){
        return self::hasWebKitSameSiteBug($userAgent) ||
            self::dropsUnrecognizedSameSiteCookies($userAgent);
    }
    private static function hasWebKitSameSiteBug($userAgent){
        return self::isIosVersion(12, $userAgent) ||
           (self::isMacosxVersion(10, 14, $userAgent) &&
            (self::isSafari($userAgent) || self::isMacEmbeddedBrowser($userAgent)));
    }

    private static function dropsUnrecognizedSameSiteCookies($userAgent){
        if(self::isUcBrowser($userAgent)) {
            return !self::isUcBrowserVersionAtLeast(12, 13, 2, $userAgent);
        }
        return self::isChromiumBased($userAgent) &&
            self::isChromiumVersionAtLeast(51, $userAgent) &&
           !self::isChromiumVersionAtLeast(67, $userAgent);
    }

    // Regex parsing of User-Agent string. (See note above!)

    private static function isIosVersion($major, $userAgent){
        $match = [];
        preg_match("/\(iP.+; CPU .*OS (\d+)[_\d]*.*\) AppleWebKit\//", $userAgent, $match);
        if (count($match) > 1){
            return $match[1] == $major;
        }
        // Extract digits from first capturing group.
        return false;
    }

    private static function isMacosxVersion($major, $minor, $userAgent){
        $match = [];
        preg_match("/\(Macintosh;.*Mac OS X (\d+)_(\d+)[_\d]*.*\) AppleWebKit\//", $userAgent, $match);
        if (count($match) > 2){
            return $match[1] == $major && $match[2] == $minor;
        }
        // Extract digits from first and second capturing groups.
        return false;
    }


    private static function isSafari($userAgent){
        return preg_match("/Version\/.* Safari\//", $userAgent) && !self::isChromiumBased($userAgent);
    }

    private static function isMacEmbeddedBrowser($userAgent){
        $regex = "/^Mozilla\/[\.\d]+ \(Macintosh;.*Mac OS X [_\d]+\) " . "AppleWebKit\/[\.\d]+ \(KHTML, like Gecko\)$/";
        return preg_match($regex, $userAgent);
    }

    private static function isChromiumBased($userAgent){
        return preg_match("/Chrom(e|ium)/", $userAgent);
    }

    private static function isChromiumVersionAtLeast($major, $userAgent){
        $match = [];
        preg_match("/Chrom[^ \/]+\/(\d+)[\.\d]* /", $userAgent, $match);
        if (count($match) > 1){
            return $match[1] > $major;
        }
        // Extract digits from first and second capturing groups.
        return false;
    }

    private static function isUcBrowser($userAgent){
        return preg_match("/UCBrowser\//", $userAgent);
    }

    private static function isUcBrowserVersionAtLeast($major, $minor, $build, $userAgent){
        $match = [];
        preg_match("/UCBrowser\/(\d+)\.(\d+)\.(\d+)[\.\d]* /", $userAgent, $match);
        if (count($match) > 3){
            $majorVersion = $match[1];
            $minorVersion = $match[2];
            $buildVersion = $match[3];
            if ($majorVersion != $major) {
                return $majorVersion > $major;
            }
            if($minorVersion != $minor) {
                return $minorVersion > $minor;
            }
            return $buildVersion >= $build;
        }
        // Extract digits from first and second capturing groups.
        return false;
    }
}
```
然后设置的代码就可以这样子写 (我刚好用的是 `Yii 2` 框架):
```php
// 不对 cookie 值进行加密
Yii::$app->request->enableCookieValidation = false;
// 接下来根据浏览器判断是否需要添加 samesite = none 的 cookie 属性, 因为有些浏览器不能识别 samesite=none 属性
// https://www.chromium.org/updates/same-site/incompatible-clients
$userAgent = Yii::$app->request->headers->get('user-agent');
if($userAgent && Cookie::shouldSendSameSiteNone($userAgent)){
    $secure = true;
    setcookie($name, $value, time() + $expire, '/;SameSite=None', $domain, $secure);
}else{
    $secure = false;
    setcookie($name, $value, time() + $expire, '/', $domain, $secure);
}
```
这样子就可以了， 在 http 的前端站点下

![1](2.png) 

而且可以看到在 浏览器的 控制台看到

![1](3.png) 

#### 3. 前端 cookie 读取问题
不过这边要注意一个问题，就是一旦这样子设置之后， 就意味着在 http 模式下的前端站点，就再也无法用 `document.cookie` 来获取这些 cookie， 因为一旦加了 secure 属性，那么在非 https 的站点是获取不到的(https 下是可以的，如果加了 httpOnly 属性，那么 https 也都不可以)。

所以如果你的站点需要读取或者判断服务端设置的 cookie 的话，那么这时候就要进行一些代码逻辑调整了。

#### 4. php session 的问题
我们知道 php session 是可以作为 cookie 存储的，但是在这种情况下， 作为 cookie 的 session 也是要设置为 `SameSite=None` 不然也是无法传输的。

以 `Yii 2` 框架为例, 那么就可以在 main.php 文件中设置

```text
'session' => array(
  'autoStart' => true,
  'cookieMode' => 'allow',
  // 设置 session cookie， 这边要设置 samesite=none 的值, 不然就会出现 session 设置了，但是 跨域请求的时候， 却传输不了的情况
  // 具体看 https://www.yiiframework.com/doc/guide/2.0/en/runtime-sessions-cookies
  'cookieParams' => [
    'httpOnly' => true,
    'secure'=> true,
    'path' => '/;SameSite=None'
  ]
)
```

这样子设置的 session cookie， 也可以带上这个属性了。

## chrome 无痕模式下无法使用的问题
注意，通过我们的上述设置，就可以在 chrome 的正常模式，在 http 的前端站点正常使用。 但是在 chrome 的无痕模式还是会出现 cookie 无法传输的情况:

![1](4.png) 

这时候其实就是被显示的 block 第三方的 cookie 了, 这时候有两种方式可以解禁。

### 1. 修改 default cookie 的配置
通过 `chrome://flags/#same-site-by-default-cookies` 将这个配置修改为 disable 就可以了

![1](5.png) 

不过要重启浏览器

### 2. 进入无痕模式的时候，将 阻止第三方 cookie 的按钮关掉
另一种更简单就是，在进入无痕模式的时候，将 阻止第三方 cookie 的按钮关掉

![1](6.png) 

这种也是可以的，而且比较轻量。


---
参考文档:
- [浏览器系列之 Cookie 和 SameSite 属性](https://github.com/mqyqingfeng/Blog/issues/157)
- [谷歌：7月14日发布的Chrome 84将恢复SameSite cookie更新策略](https://www.cnbeta.com/articles/tech/984703.htm)
- [SameSite=None: Known Incompatible Clients](https://www.chromium.org/updates/same-site/incompatible-clients)
- [Cookie Legacy SameSite Policies](https://www.chromium.org/administrators/policy-list-3/cookie-legacy-samesite-policies)
- [Yii 2.0 权威指南 - Sessions 和 Cookies](https://www.yiiframework.com/doc/guide/2.0/zh-cn/runtime-sessions-cookies)