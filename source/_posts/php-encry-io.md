---
title: 使用 ionCube 对 PHP 项目进行加密
date: 2022-09-29 17:31:23
tags: 
- security
- php
categories: php相关
---
## 前言
之前我们尝试使用 `yakpro-po` 和 `Swoole Compiler` 对我们的本地化部署的 PHP 项目进行混淆加密
- {% post_link php-encry-yp %}
- {% post_link php-encry-sc %}

接下来我们继续试用另一个扩展加密的方案 - ionCube

## ionCube
- [PHP Encoder 12](https://www.ioncube.com/php_encoder.php)

我们用的是 `PHP Encoder 12` 这个版本 (他的低版本有被破解的风险，这个版本是扩展加密版本，目前外网没有被破解的相关资讯，安全性应该靠谱)

简单的来说 ionCube 编码器将源代码编译为字节码，可以根据需要对编译后的代码进行混淆和加密，并具有以各种方式保护解密密钥的功能。

此外，`Pro` 和 `Cerberus` 版本中内置的 PHP 许可功能允许对 PHP 脚本进行许可，限制 PHP 代码可以在哪里运行以及它是否有到期时间。
<!--more-->
使用 ionCube 编码器，您可以:
* 使用编译的字节码保护 PHP 脚本以获得最佳性能和保护。
* 使用不存储但仅在需要时生成的可选加密密钥（动态密钥）。我们的独特功能大大增强了对将解密密钥存储在受保护文件中或根本不提供加密的替代方案的保护。
* 生成编码的 PHP 文件以在 PHP 8.1 和更早版本上运行。
* 使用 PHP 8.1 之前的 PHP 语言功能。
* 加密非 PHP 文件，例如 XML 和模板。
* 生成许可证文件以限制对编码文件的访问（Pro/Cerberus 版本）。
* 启用变量和函数、方法和类名称的单向转换（混淆）。
* 编码 PHP shell 脚本。
* 通过使用数字签名防止文件篡改。
* 防止其他人替换编码文件。
* 生成在给定日期或一段时间后过期的文件（Pro/Cerberus 版本）。
* 限制文件在 IP 地址和/或服务器名称的任意组合上运行（Pro/Cerberus 版本）。
* 限制文件在特定 MAC 地址上运行（Cerberus 版）。
* 与ionCube Package Foundry集成。
* 为自定义版权、许可证详细信息等向编码文件添加可读注释。
* 当文件过期或无权运行时，有自定义消息和自定义处理。

### 扩展加密的工作方式
扩展加密的通用方式一般就是包含加密器(encoder)和解密器(loader)两个程序, 通过加密器的一次加密，可以运行在各种有安装 loader 扩展的 php 环境，不限机器。 

不过会限制加密的机器，也就是所谓的我们要购买的加密器的 license， 当我们购买了这个license 之后，一旦在这台机器上激活这个 license，就只能在这台机器上才能使用加密器来加密。 但是使用这台机器加密过的 php 文件，可以跑在任何安装 ioncube loader 的所有的 php 环境。

> 所以如果要买，其实就是买加密器的 license

## 初体验
### 1. 申请试用
他这个是有申请试用的，有两周的试用版本: [申请免费试用](https://www.ioncube.com/encoder_eval_download.php)

![](1.png)

用我的 gmail 申请了一下，并且选择编码器平台为 `Linux`, 这时候就会发一封邮件到你的 mail 邮箱

![](2.png)

这时候可以去下载编码器了， 通过这个[下载 url](https://www.ioncube.com/eval_linux)  可以下载到一个 `.tar.gz` 的压缩包

看样子是直接给的是最高的 Cerberus 版本的试用版，这个编码器的有效期是 14 天， 加密后的文件会再 36 小时到期。

![](3.png)

里面包含了各个版本的 php 编码器

![](4.png)

接下来我们试一下能不能正常的加密和解密
### 2. 安装 loader
测试环境还是之前测 `yakpro-po` 和 `Swoole Compiler` 的那一台 php 7.4 的 docker 环境 ( [LNMP一键安装程序](https://github.com/kebingzao/dnmp))。

在进行加密之前，我们要先安装解密的 loader 组件。其实里面有包含安装 loader 的向导程序 (loader-wizard 目录，里面就一个 loader-wizard.php 这个文件)，只需要放到 localhost 下可以访问就行了

还是在之前测试的那一台机器，上面已经安装了 PHP 7.4，我们直接将这个 .tar.gz 的压缩包拷到 localhost 目录，并且进行解压

```text
tar -zxvf ioncube_encoder_evaluation.tar.gz
```
接下来直接访问 loader 向导 (这台 linux 要可以连外网，并且 nginx 的 80 要允许开放外网访问)
```text
http://43.xx.xx.234/ioncube_encoder_evaluation/loader-wizard/loader-wizard.php
```
可以看到有一个引导

![](5.png)

> 这边有一个重要警告，就是这个安装向导，在我们都安装好之后， 就可以不再需要了， 就可以删掉了

看起来安装过程有 6 步， 一步一步来，简单的概括，其实就是
1. 下载上面链接的 loader 扩展包
2. 将 loader 扩展包的对应的 php 版本的扩展文件放到当前 php 环境的 extension 目录
3. 下载 ioncube.ini 文件，并放到 php 的 `conf.d` 目录
4. 重启 php-fpm
5. 最后点击 `click here to test the Loader` 连接，让程序检查 loader 是否安装并且生效

所以我们先下载 loader 的扩展包

![](6.png)

这个其实就是各个 php 版本的 loader 的扩展， 可以全部放到扩展文件夹中，也可以只放一个我们的版本也就是 `ioncube_loader_lin_7.4.so`

因为接下来让我们下载的这个 `00-ioncube.ini` 其实就只启用了 7.4 环境的这个 loader，其他的 loader 好像也没有启用, 所以其实只需要将 `ioncube_loader_lin_7.4.so` 这个扩展程序放到 php 的扩展包目录就行了

![](7.png)

本着最小化原则，所以我只放这个 `ioncube_loader_lin_7.4.so` 扩展，其他扩展不放
```text
/usr/local/lib/php/extensions/no-debug-non-zts-20190902 # cp /www/localhost/ioncube/ioncube_loader_lin_7.4.so ./
/usr/local/lib/php/extensions/no-debug-non-zts-20190902 # chmod 755 ioncube_loader_lin_7.4.so
```
调整一下权限，跟其他扩展文件的权限一样

然后将第三步下载的这个 `00-ioncube.ini` 放到  `/usr/local/etc/php/conf.d`  中
```text
/usr/local/etc/php/conf.d # cp /www/localhost/00-ioncube.ini ./
```
接下来重启 php-fpm， 其实就是重启 docker 容器 ->  `docker restart php`

接下来点击第五步的这个 check 链接, 这时候就会显示安装成功了

![](8.png)

### 3. 使用编码器进行编码
loader 好了，接下来就是要开始对文件进行编码， 他之前有一个 readme.txt 文件， 使用编码的方式应该是在那里

![](9.png)

所以只需要执行对应版本的编码文件，也就是 12 版本的 7.4 版本的文件

![](10.png)

编码的方式有两种，一种就是用下载的那个编码的 shell 脚本 `ioncube_encoder.sh` 来进行编码, 比如执行 `-h` 可以看到指令
```text
[root@VM-16-223-centos ioncube_encoder_evaluation]# ./ioncube_encoder.sh -h

The following is a summary of command options for this script and its basic usage.

Usage: ioncube_encoder.sh [-C | -L | -O] [-4 | -5 | -53 | -54 | -55 | -56 | -71 | -72 | -74 | -81] [-x86 | -x86-64] <encoder options>

Encoder Version (optional):
-O : Use Obsolete Encoder (v10.2)
-L : Use Legacy Encoder (v11.0)
-C : Use Current Encoder (v12.0) - Default

PHP Languages:
-4 : Encode file as PHP 4
-5 : Encode file as PHP 5
-53 : Encode file as PHP 53
-54 : Encode file as PHP 54
-55 : Encode file as PHP 55
-56 : Encode file as PHP 56
-71 : Encode file as PHP 71
-72 : Encode file as PHP 72
-74 : Encode file as PHP 74
-81 : Encode file as PHP 81

Architecture (optional):
-x86 : Run the 32-bit Encoder
-x86-64 : Run the 64-bit Encoder

-h : Display this help and exit.
If -h is specified before a language has been selected, help will be displayed by this script.
if -h is specified after a language has been selected, help will be displayed by the Encoder.

If an Encoder version is not selected, the Current Encoder (12.0) will be selected.
If a PHP language is not selected, the script will exit.
If an architecture is not selected, the script will run the Encoder that matches your system architecture.

Once an unknown option is selected, the script will pass the remaining options to the Encoder.
You cannot select more than one Encoder version, PHP language or Architecture.
Script will exit should you try to run the 64-bit Encoder on a 32-bit system.

Usage examples:

Current 64-bit Encoder, encoded as PHP 8.1
  ./ioncube_encoder.sh -C -x86-64 -81 source_file.php -o target_file.php
```
也就是:
- `-C` 表示用的是编码器的 12 版本
- `-x86-64` 使用的是 64位的编码器
- `-81` 编码的文件是 php 8.1 的文件

至于后面的 `-o` 其实就是另一个程序的指令，也就是具体得到的那个编码程序的 shell 指令， 也就是下面我所说的第二种编码的 shell 方式， 因为看了一下 `ioncube_encoder.sh` 的代码，其实逻辑很简单，就是找到具体版本和环境的编码器文件，然后继续执行这个编码器的 shell 指令
```text
checkEncoderExists() {
    if [ -f $encoderPath ] ; then
        if [ -x $encoderPath ] ; then
            true
        else
            fail "The Encoder is not executable."
        fi
    else
        fail "The Encoder does not exist at the path: $encoderPath"
    fi
}
```
```text
checkEncoderExists

[ "$warning" != "" ] && echo "$warning"

exec $encoderPath "$@"
```
所以以下两个指令是等价的
```text
./ioncube_encoder.sh -C -x86-64 -81 source_file.php -o target_file.php
./bin/ioncube_encoder81_12.0_64 source_file.php -o target_file.php
```

所以我们可以简单的用命令加密一下我们的一个 php 文件, 因为是 7.4 版本 64 位的，所以我们可以很轻松的找到对应的编码器文件
```text
[root@VM-16-223-centos ioncube_encoder_evaluation]# ./bin/ioncube_encoder74_12.0_64  ../test.php -o ion_test2.php
```
将 `test.php` 加密成 `ion_test2.php`
> 上面的指令等价于 `./ioncube_encoder.sh -C -x86-64 -74 ../test.php -o ion_test2.php`

这时候打开看一下，可以发现已经被加密了

![](11.png)

因为同一台已经装了 loader， 所以我们可以直接执行
```text
[root@VM-16-223-centos ioncube_encoder_evaluation]# curl http://localhost/ioncube_encoder_evaluation/ion_test2.php
<h1 style="text-align: center;">welcome  test kbz DNMP !!</h1><h2>版本信息</h2><ul><li>PHP版本：7.4.27</li><li>Nginx版本
```
发现可以成功解密没有问题。

所以对单文件基本的加密和解密都没有问题。

### 4 其他
#### 4.1 36 小时解密文件失效
后面过了几天我试了一下，确实解密文件失效了， 执行的时候，什么都没有输出
```text
[root@VM-16-223-centos ~]# curl http://localhost/ioncube_encoder_evaluation/ion_test2.php
```
#### 4.2 把 loader 去掉再执行
在解密文件的有效期内，如果将 loader 去掉的话，就会报这个错误
```text
Loader for PHP needs to be installed
```

![](12.png)

#### 4.3 docker 容器内没办法执行加密指令
我还遇到了一种情况，就是 encode 的时候，在容器里面会报错:
```text
/www/localhost/ioncube_encoder_evaluation # ./ioncube_encoder.sh -C -x86-64 -74 ../test.php -o ion_test.php
./ioncube_encoder.sh: exec: line 445: /www/localhost/ioncube_encoder_evaluation/bin/ioncube_encoder74_12.0_64: not found
```
提示 加密器程序找不到， 但是其实是有的，因为 www 目录有映射到宿主机
```text
/www/localhost # stat /www/localhost/ioncube_encoder_evaluation/bin/ioncube_encoder74_12.0_64
  File: /www/localhost/ioncube_encoder_evaluation/bin/ioncube_encoder74_12.0_64
  Size: 1337344       Blocks: 2616       IO Block: 4096   regular file
Device: fd01h/64769d    Inode: 792587      Links: 1
```
在容器里面使用 stat 检测加密器是存在的，但是就是加密的时候，就会提示不存在。 同样的指令在宿主机就可以正常执行。 

所以我怀疑他这个加密过程是要绑定宿主机的某些参数的，在容器就会执行失败，报这个错误

#### 4.4 加密文件头部的通用注释，在正式版本是不会有的
用试用版本加密的文件头部都会有这一串, 并且不能用 `--add-comment` 指令的内容覆盖
```text
// IONCUBE ENCODER 12.0 EVALUATION
// THIS LICENSE MESSAGE IS ONLY ADDED BY THE EVALUATION ENCODER AND
// IS NOT PRESENT IN PRODUCTION ENCODED FILES
```
不过这一串在正式版本是不会有的，上面写的很清楚

#### 4.5 免费申请的加密包的过期时间是 14 天
他这个加密程序的过期时间是 14 天，我测试了一下，这个限制并没有依托外部文件，应该是直接内置到加密器的程序中的， 所以 14 天到了，这个加密器应该就不能用了. 这时候就会报这个错误:
```html
[root@VM-16-223-centos ioncube_encoder_evaluation]# bin/ioncube_encoder74_12.0_64 ../test.php -o ion_test3.php
Error: The Encoder evaluation period has expired.
```

不过可以重复下载，通过这个 [免费申请下载加密包](https://www.ioncube.com/eval_linux), 在浏览器的空窗口打开，就可以再下载一份新的，又有 14 天了。

但是他加密的文件，最多有效期是 36 小时，而且好像是不能通过 `--expire` 指令来延长， 只能缩短，不能延迟

#### 4.6 关于加密器的各种指令内容和文档
刚开始使用加密器加密的时候，一开始是通过 `-h` 查看使用文档，但是还是很不方便，去官网找了一圈，都没有找到对应的文档。

后面发现加密指令文档，竟然是放在之前下载的 加密包文件夹的 docs 目录下的 `USER-GUIDE.pdf`, 这里面有完整的各种加密的指令的文档说明

![](13.png)

这个 pdf 文档的用户手册有 82 页，其中有一大半是讲各种指令的各种用法，非常的丰厚，用途也非常的广泛。后面会挑一些比较有用的指令稍微讲解一下

## PHP 项目使用 ioncube 项目来加密
既然使用上没啥问题， 那么接下来就看能不能将这个加密应用到 php 项目了。 以我们的 Yii2 框架的项目为例。

因为这一台运行 Yii2 的服务是在另一台机器上的，并且环境是 php 5.6，所以我们得在这一台服务器上先安装 loader，这时候就会变成有两台服务器了，一台是之前测试的加密服务器 A，一台是我们的 php 项目运行的服务器 B, 所以流程就变成:
1. B 服务器安装 php 5.6 的 loader
2. B 服务器 php 的代码打包压缩成 code.zip，然后上传到 A
3. A 用 php 5.6 的加密程序加密 code.zip, 加密后的文件是 code-encode.zip，然后上传到 B
4. B 解压 code-encode.zip，并覆盖原先的代码
5. 运行 B 的对应 php 项目的代码，看是否正常

### 1. 安装 php 5.6 的 loader
本来我想简单一点，不需要再装 loader 的引导程序了，因为 php 7.4 的环境都装过了，结果发现 php 5.6 貌似没有 `conf.d` 目录放 ini 文件，为了避免少走弯路。 我还是将这个引导程序拷贝到 B 服务器 (其实就一个 php 程序)

然后直接在当前的目录直接起一个 php 服务 (不跟原先的 php 服务进程放一起，怕影响，所以直接单独起服务)
```text
root@jurchens:~/ioncube# php -S 0.0.0.0:9011 -t ./
PHP 5.6.40 Development Server started at Wed Sep 28 06:11:43 2022
Listening on http://0.0.0.0:9011
Document root is /root/ioncube
Press Ctrl-C to quit.
[Wed Sep 28 06:11:46 2022] 192.168.40.51:60091 [200]: /loader-wizard/loader-wizard.php
```
然后直接请求这个引导文件

![](14.png)

果然流程不太一样，除了将 5.6 的 loader 一样放到对应的 extension 目录之外， 原先放 conf.d 的 ini 文件，变成直接在 php.ini 写入。 所以我就将其写在 php.ini 的 `[PHP]` 块的最下面
```text
root@jurchens:/usr/local/php-5.6.40/etc# cat php.ini | grep -3 'zend_extension'
; Default: ""
;zend.script_encoding =

zend_extension = /usr/local/php-5.6.40/lib/php/extensions/no-debug-non-zts-20131226/ioncube_loader_lin_5.6.so

;;;;;;;;;;;;;;;;;
; Miscellaneous ;
```
然后重启 `php-fpm` 程序，并且将之前的单独起的 php 服务也重启了，这时候就可以看到 loader 安装成功了

![](15.png)

### 2. 加密 Yii2 项目
接下来就将 Yii2 的项目拿去 A 服务器加密，因为 vendor 目录很大(有使用 composer 做包管理)，文件很多，又是开源的库, 所以 vendor 并不需要一起加密 (加密的时候，因为文件太多太大，会很慢，没啥必要)， 所以打成压缩包的时候，就不要打进去了
```text
zip -r id-yii-origin.zip id-yii  -x "./id-yii/vendor/*"  -x "./id-yii/api/data/GeoLite2-City.mmdb"
```
生成压缩包的时候，忽略 vendor 目录和 ip 库文件(那个很大，也忽略)， 其他就全部都可以了。接下来就可以在 A 服务器进行加密了 (找到对应的 php 5.6 加密器)
```text
unzip -d id-yii-origin id-yii-origin.zip
./ioncube_encoder_evaluation/bin/ioncube_encoder56_12.0_64 id-yii-origin -o id-yii-origin-en --add-comment "Software written by Zach"
```
> 通过 `--add-comment` 可以添加自定义注释

> 而且默认只针对 .php 文件进行加密，其他类型的文件没有指定的话，不会加密只会原样返回

虽然打开加密后的文件
```text
[root@VM-16-223-centos id-yii]# head requirements.php
<?php //0037f
// IONCUBE ENCODER 12.0 EVALUATION
// THIS LICENSE MESSAGE IS ONLY ADDED BY THE EVALUATION ENCODER AND
// IS NOT PRESENT IN PRODUCTION ENCODED FILES

// Software written by Zach
if(extension_loaded('ionCube Loader')){die('The file '.__F
```
> 这时候这个文件在 A 服务器是没办法运行的，因为 A 装的 loader 是 7.4， 没办法解析 5.6 版本的加密文件， 不存在向下兼容的问题，版本不匹配，就是不行

接下来就将加密后的文件覆盖到 B 服务器的对应的项目文件。这样子对应的 php 文件都变成加密了。 试了几个接口， 挺正常的。

> 后面有遇到一个问题，就是覆盖的时候， 有可能日志文件被覆盖了，导致用户权限被改写了， 导致程序跑的时候日志写不进去，这时候要将日志文件的权限恢复成之前就行了

## 总结
试用了 ioncube 的 php 商业加密， 可以感觉到整个过程其实很顺畅，而且对程序的兼容性也好，哪怕是免费的申请试用，都有完整的用户手册和安装引导， 不亏是业内公认的头部 php 商业加密厂商。 

这家公司成立了 20 年了，专注于 php 的商业加密也有十几年，不管是使用体验上还是安全性都可以得到保障。

因为是按照 license 买的，哪怕是最高级的 Cerberus 版本，一个 license 是 $399, 因为 license 是终身， 只要不换加密的机器，就可以一直用， 所以应该也不贵
> 最高级的 Cerberus, 可以对加密的 php 文件做很多限制，包括并不限于，限制 php 的过期时间，限制 mac 地址，限制 ip 名单，添加许可证限制，添加允许linux的用户组限制

> 后面真的去采购了Cerberus 版本，具体: {% post_link php-encry-io-2 %}

---

参考资料:
- [PHP Encoder 12](https://www.ioncube.com/php_encoder.php)
- [产品价格对比页面](https://www.ioncube.com/php_encoder.php?page=pricing)
- [Frequently Asked Questions](https://www.ioncube.com/faq.php)







