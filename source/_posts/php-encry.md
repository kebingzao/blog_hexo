---
title: 浅谈之 - PHP 项目代码加密方案
date: 2022-09-20 16:57:29
tags: 
- security
- php
categories: php相关
---
## 前因
未来有个项目需要对客户进行远程部署交付，意味着我们的服务会部署在用户自己的服务器上，考虑到商业代码的安全性，有一些服务是用 PHP 开发的项目，PHP 这种解释性语言，其实是直接源文件显示的， 这样子其实就相当于把我们的源代码暴露出来。 这样子肯定是不行的。
                                                                        
所以我们要对 PHP 的代码进行加密，所以结合网上针对 PHP 加密的一些文章， 我这边再次加工自己总结了一份。

## PHP 的几种加密方式
按照加密的方式来看， PHP 可以包含以下几种方式:

### 1. 壳”加密”
这一类“加密”包括：
- 无扩展加密：`phpjiami`、`zhaoyuanma` 的免费版本等
- 有扩展的加密：`php-beast`、`php_screw`、`screw_plus`、`ZoeeyGuard`、`tonyenc` 等市面上几乎所有的开源PHP加密扩展。

<!--more-->
这种加密严格上，不算真正的加密，它们真的真的只能被称为“自解压压缩包”，像是 PHP 界的 `WinRAR`，或者是`UPX`、`ASPack`。 

这一类自解压压缩包的共同思路是：
- **加密**：直接加密整个PHP文件，不对原始PHP逻辑作出改动。无扩展的加密将给用户一个运行时环境（“壳”）和加密后的数据，有扩展的加密将直接获得加密后的数据，并要求用户在使用时安装对应的扩展。
- **解密**：壳或扩展先确认环境有没有被调试的风险，倘若没有，就直接在内存中解密出整个PHP文件，并使用eval或类似方式直接运行。

比如下面一个简单的实例:
```php
<?php
  $code = file_get_contents('待加密的PHP');
  $code = base64_encode(openssl_encrypt($code, 'aes-128-cbc', '密钥', false, 'IV'));
  echo "<?php eval(openssl_decrypt(base64_decode($code), 'aes-128-cbc', '密钥', false, 'IV'));";
```

对这一类“壳加密”来说，是有万能的“解密”方案的。不需要知道数据的加密算法到底是什么，因为真实代码在执行时总会被解密出来，各位只需要知道PHP到底执行了什么，从这儿拿出代码。

不管是`eval`、`assert`、`preg_replace('//e')`，还是这类PHP加密扩展，想要动态执行代码就必须经过 `zend_compile_string` 这一个函数。只需要编写一个 `dll/so`，给 `zend_compile_string` 挂上“钩子”，就能直接拿到完整的代码。

给出几篇文章作为参考：
- [PHPDecode 在线解密工具](http://blog.evalbug.com/2017/09/21/phpdecode_01/)
- [phpjiami 数种解密方法](https://www.leavesongs.com/PENETRATION/unobfuscated-phpjiami.html)
- [UnPHP - The Online PHP Decoder (在线 PHP 解密网站)](https://www.unphp.net/)
- [\[Web逆向\] PHP解密：zym加密 带乱码调试过程](https://www.52pojie.cn/thread-693641-1-1.html)

> 综上所述，不会考虑用这种方式来进行加密

### 2. 混淆加密
混淆加密，算加密， 基本原理是:
1. 移除代码内的变量，将其替换为乱码或 `l1O0` 组合成的变量名。因为只改变变量名，大部分情况下并不会对代码的逻辑产生影响。
2. 对PHP代码本身的明文字符串，像是变量名、函数名这些进行替换。
3. 一定程度上改变代码原始逻辑。

这一类在国内用的最多的是 [EnPHP](http://enphp.djunny.com/)，开源的有 [php-obfusactor](https://github.com/naneau/php-obfuscator)。当然，还有一种更强大的开源加密 [yakpro-po](https://github.com/pk-fr/yakpro-po)， 

这一类加密的开发门槛就相对高些了，需要熟悉对于抽象语法树（AST）的操作。

代码混淆对于一般的防破解来说强度是足够的，Google 在 Android 上即默认提供了 ProGuard 这一明文符号混淆工具，在PHP上同样，如果变量名、函数名等被混淆，确实可以增加破解难度，对应的工具是 `php-obfusactor`。不过，这对一般的逆向造不成什么影响，批量替换变量名就可以解决了。`EnPHP` 和 `yakpro-po` 相对会麻烦一些。

#### 2.1 EnPHP
- [EnPHP 官网](http://enphp.djunny.com/)
- [旧版本有开源](https://github.com/djunny/enphp)

特点:
- EnPHP 支持加密混淆 PHP 代码。
- EnPHP 可以对函数、参数、变量、常量以及内嵌 HTML 代码进行加密、混淆。
- 支持不同的加密强度、混淆方式
- 压缩后，均无法正常修改文件
- EnPHP 加密混淆后不添加额外代码，经过大量测试，性能损耗在 1% 以内，绝不会有额外开销。

这个是国内开发的，应该也是国内用的比较多的 php 混淆加密，旧版的有开源，新版的没有开源，但是在站点上，允许免费的进行加密和解密。

看了一下他们的官网，上面没有显示商业版本， 但是貌似可以联系他们的管理员来得到 商业版本

`EnPHP` 的特征是，将所有的函数名都提取到一个常量池，在一定程度上修改了变量名，不过不改变代码逻辑 (下面的截图是旧版的，新版有加上了 goto 混淆，会更复杂一点)。

![](1.png)

这种加密实现难度不高，只要熟悉对 `php-parser` 的操作即可写出来，不过随着混淆的程度越来越深， 破解的难度也会越来越大，而且变量，尤其是局部变量，是不可还原的。

网上有查到 EnPHP 破解，不过是第一版的 (还没有查到 新版 goto 版本的网上破解):
- [\[Web逆向\] PHP解密：EnPHP mzphp2加密 无法还原变量名的混淆加密](https://www.52pojie.cn/thread-883976-1-1.html)

**总结:**

EnPHP 是主打混淆 + 加密，随着混淆的程度越来越深(提供不同级别的混淆)， 破解的难度也会越来越大， 就算真的被解密了，也是不可能还原变量名的，就是不可能还原到源代码那种情况，会对可读性造成影响。

> 查看了一下，最近的一次更新是 `2020-05-21` , 也就是新版本上线的日期， 后面就再也没有更新，感觉好像也没有再维护

#### 2.2 php-obfuscator
- [php-obfuscator](https://github.com/naneau/php-obfuscator)

也是混淆 + 加密， 这个破解难度不高，而且 6 年多没有维护，下面的 `yakpro-po` 同样是开源，并且混淆算法更强， 这个不考虑

#### 2.3 yakpro-po
- [yakpro-po](https://github.com/pk-fr/yakpro-po)
- [yakpro-po demo test](https://www.php-obfuscator.com/?demo)

`yakpro-po` 的特征是大量的goto混淆, 类似下图

![](2.png)

这种混淆器的特点如下：
- 正常语句，将被混淆成 `labelxxx: one_line; goto nextlabel`;。直接将这三条语句视为一个混淆节点即可。
- `if / if else / if elseif else`，处理差别不大，直接还原即可。
- 嵌套型 if 相对比较麻烦，因为没有嵌套 if 的概念，一切 if 均在最外层。简单的处理方案是，如果跳到的节点有 if 语法，重新递归解析这个节点。

网上目前没有找到靠谱的开源的解混淆方案，而且这个混淆器是开源的. 意味着我们可以用这个混淆器直接在我们内部服务器对其进行加密。

他执行加密的 PHP 版本有要求 php 版本 7.0 上 (因为他依赖的 `PHP-Parser 4.x` 要求要 7.0 才能跑)。 但是他加密过后的 php 文件是可以在 5.2 和 7.3 之间版本上面跑的

查了一下 github 上面的最近一次提交记录 `2022-07-05`, 而最后一次 release 版本日期是 `2022-04-24`, 所以应该是还在维护

### 3. 无扩展虚拟机加密
目前市面上无扩展的虚拟机加密只有两款，且收费均不菲：
1. [Discuz应用中心开发的魔方加密](https://www.mfenc.com)
2. [Z-Blog团队开发的Z5加密](https://z5encrypt.com)

这两款加密的共同特点是：它们都实现了一个PHP语言的编译器，将PHP转换为它们的内部代码；用户将收到一个解释器，解释器的作用是根据内部代码来执行对应的指令。这就像写C语言一样，编译器负责把C语言写的代码转换为机器码，这种机器码CPU可以直接执行。

这种加密方式，在 Windows / Linux上已经很成熟了，代表作品是VMProtect。这种运行方式已经在理论上证明了反编译出源码是不可能的，相对来说也是无扩展加密中最安全的。安全的同时也需要付出一定的代价，它们的运行效率也是最低的。

尽管如此，它们也不是百分百安全。虽然不能反编译出源码，但是可以根据它们的执行逻辑转写出功能类似的代码。

#### 3.1 魔方加密
- [魔方加密](https://www.mfenc.com/)
- [更新日志](https://www.mfenc.com/document/changelog.md)

有找到破解的帖子连接， 但是点进去发现帖子被屏蔽了。

> 不过有查看了他们的更新日志，最新一次的更新是 `2017-04-18`， 所以应该是没有在维护了。 所以不考虑

#### 3.2 Z5 加密
- [Z-Blog团队开发的Z5加密](https://z5encrypt.com)

Z5加密的作者似乎在这之上改进了不少，笔者登陆其官网，发现其有如下功能：
- 增加垃圾代码、扁平化控制流、指令膨胀。
- 明文字符串加密、常量池。
- 虚拟机共享、反调试。

> Z5加密的破解极为麻烦，不过他的php 程序运行效率非常低，他自己官网也说了，不建议对整站的内容进行加密， 最好只对核心文件，比如授权文件进行加密。

> 而且考虑到的[最后一次更新](https://z5encrypt.com/docs/update-log/)是 `2020-04-06`, 感觉也没有再维护了。 所以不考虑


### 3. 近似加密
这其实不属于加密，而是利用 PHP 自身功能来达到类似加密的效果。PHP在5.5之后自带 OPcache，而5.5之前有 `Zend Optimizer`。而已经停止开发的 `Zend Guard`、老版本 `ionCube` 和部分配置下的 `Swoole Compiler`，即是基于这一些系统功能进行加密。

PHP通常在Zend引擎上运行，Zend引擎会先将PHP编译为OPcode，OPcache的原理就是缓存了这些OPcode避免下一次运行时仍然产生编译开销。当然，OPcache也是人类不可直接读的。按照PHP官网所说：
```text
OPcache 通过将 PHP 脚本预编译的字节码存储到共享内存中来提升 PHP 的性能， 存储预编译字节码的好处就是 省去了每次加载和解析 PHP 脚本的开销。

PHP 5.5.0 及后续版本中已经绑定了 OPcache 扩展。 对于 PHP 5.2，5.3 和 5.4 版本可以使用 » PECL 扩展中的 OPcache 库。
```

#### 3.1 Zend Guard
- [Zend Guard PHP App Protection](https://www.zend.com/products/zend-guard)
- [PHP 编码器、Zend Guard 和 PHP 7：概述](https://www.zend.com/blog/zend-guard-and-php-7)

Zend Guard 已经不再维护了， 最近一条 blog 是在 `2016-10-10`, 支持的 PHP 版本只到 `php 5.6`, PHP 7 及以上将不再支持。

```text
支持的 PHP 版本： 

5.3.x 到 5.6（使用 Zend Guard Loader 作为运行时 PHP 解码器）。
4.2.x 到 5.2.x（使用 Zend Optimizer 作为运行时 PHP 解码器）。
```

不过 Zend 的官网确实还在卖，也会持续提供服务，只不过他不再支持 PHP 7 及以上
```text
这是简单的部分。Zend Guard仍可在我们的商店购买。
如果您使用它来保护您的 PHP 5 应用程序，您可以继续使用它。我们会继续支持它。我们有一个团队可以回答问题并帮助您充分利用它。
```

而且有在 github 上发现针对 php5.6 的 zend 解密 (有可能商业付费版会更好， 但是不确定):
- [Zend-Decoder](https://github.com/Tools2/Zend-Decoder)

> 项目的 5.6 版本， 虽然 Zend Guard 还有支持，但是考虑到长远的升级版本迭代，后面项目可能会升级到 php 7 及以上， 那么这种加密就不适用了， 所以慎用

#### 3.2 ionCube
ionCube 低版本是这种加密， 高版本是更安全的 扩展加密。 因此对于低版本不考虑。

### 4. 扩展加密
这一块需要安装扩展，才能实现的加密


#### 4.1 Swoole Compiler
- [Swoole Compiler](https://business.swoole.com/compiler.html)
- [Swoole Compiler(文档)](https://www.kancloud.cn/swoole-inc/compiler/1788476)
- [faq](https://www.kancloud.cn/swoole-inc/compiler/1788479)

Swoole Compiler 是识沃科技推出的 PHP 代码加密和客户端授权解决方案，通过业内先进的代码加密技术，包括流程混淆、花指令、变量混淆、函数名混淆、虚拟机保护技术、扁平化代码、SCCP 优化等，将 PHP 程序源代码编译为二进制指令，来保护您的源代码。

这个也是非常安全的，自己做了一套解析器，然后将 PHP 编译成二进制代码。而且经过大量性能优化，加密后性能不仅没有任何损耗，反而会优化程序指令，提升性能。 目前外网上没有找到可靠的破解文章。

国内团队做的， 也是 Swoole 协程框架的团队， 技术上有保障， 提供 7*24 小时中文服务，无语言沟通障碍。

与`Zend Guard` 等传统的PHP加密器不同，Swoole Compiler 没有软件界面，离线版提供了 API ，可将 Swoole Compiler 集成到您的打包发布平台中，完全是可编程的。


#### 4.2 ionCube PHP Encoder 12
- [ionCube PHP Encoder 12](https://www.ioncube.com/php_encoder.php)
- [价格页](https://www.ioncube.com/php_encoder.php?page=pricing)
- [faq](https://www.ioncube.com/faq.php)

提供的功能算是比较多的:
- 使用编译的字节码保护 PHP 脚本以获得最佳性能和保护。
- 使用不存储但仅在需要时生成的可选加密密钥（动态密钥）。我们的独特功能大大增强了对将解密密钥存储在受保护文件中或根本不提供加密的替代方案的保护。
- 生成编码的 PHP 文件以在 PHP 8.1 和更早版本上运行。
- 使用 PHP 8.1 之前的 PHP 语言功能。
- 加密非 PHP 文件，例如 XML 和模板。
- 生成许可证文件以限制对编码文件的访问（Pro/Cerberus 版本）。
- 启用变量和函数、方法和类名称的单向转换（混淆）。
- 编码 PHP shell 脚本。
- 通过使用数字签名防止文件篡改。
- 防止其他人替换编码文件。
- 生成在给定日期或一段时间后过期的文件（Pro/Cerberus 版本）。
- 限制文件在 IP 地址和/或服务器名称的任意组合上运行（Pro/Cerberus 版本）。
- 限制文件在特定 MAC 地址上运行（Cerberus 版）。
- 与ionCube Package Foundry集成。
- 为自定义版权、许可证详细信息等向编码文件添加可读注释。
- 当文件过期或无权运行时，有自定义消息和自定义处理。

安全应该也是很安全的，网上也没有看到靠谱的破解文章， 其实就包含两个扩展:
1. 用于加密的编码器(PHP Encoder) , 这个要钱
2. 运行编码脚本的解码器 (ionCube Loader)， 这个不要钱，可以直接安装

然后这个编码器是要许可证 (license) 的，这个要购买的，一旦在这一台机器上激活了这个许可证，那么就只能在这一台机器上对 php 文件进行编码加密。

当然编码过后的加密文件可以拿到任何的平台进行运行，前提是允许的 php 环境要安装 ionCube Loader。

提供了三个版本的编码器，除了基础版本之外，其他的版本在编码的时候，还可以限制 编码后的 php 文件可以在特定机器上运行，可以绑定类似于 mac 地址，并且可以设置运行时间，不能超过某个时间点。

## 调研总结
可以从 `混淆加密` 和 `扩展加密` 来考虑， 从安全性考虑， 一定是扩展加密更安全(都是商业版本，要付费)， 不过在测试阶段，可以两者都测试一下
- `混淆加密` 可以使用 `EnPHP`(不开源，但是貌似免费，不过 2 年多没更新了，不知道是否还维护) 和 `yakpro-po` (完全开源) 看看效果
- `扩展加密` 就是 `Swoole Compiler` 和 `ionCube PHP Encoder 12` 考虑

当然，从绝对的安全和工作量来看， 应该还是要 扩展加密 更好， `混淆加密` 有两个问题:
1. 因为混淆加密 对程序不会有 100% 兼容, 有些奇特的写法或者用法 会导致报错， 所以需要开发人员去排查， 或者是加密的时候没问题，但是运行的时候却报错，这个就更坑了。 所以如果要做到完全兼容的话， 估计调整的代码不会少，这一部分的工作量不可控
2. 混淆加密是可以被破解的，虽然没办法 100% 还原源文件，代码的可读性会变差，但是如果仔细去读的话，也是可以知道程序逻辑的

混淆加密的好处就是免费的，有开源的程序可以直接用。 

后面有使用了一下`yakpro-po`开源的混淆加密: {% post_link php-encry-yp %}

还试了 `swoole compiler` 的扩展加密:  {% post_link php-encry-sc %}

---

参考资料:
- [PHP代码加密面面观](https://www.anquanke.com/post/id/176767)
- [PHPDecode 在线解密工具](http://blog.evalbug.com/2017/09/21/phpdecode_01/)









