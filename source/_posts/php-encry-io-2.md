---
title: 使用 ionCube 对 PHP 项目进行加密(2) - 加密指令详解
date: 2022-10-18 11:37:13
tags: 
- security
- php
categories: php相关
---
## 前言
之前我们已经有使用 ionCube 对 PHP 项目进行加密: {% post_link php-encry-io %}

之前拿来做测试的是 免费申请试用版本, 后面就直接买了一个正式版的 Cerberus 的，$399 一个 license 的，永久使用 (除非换机器)

![](1.png)

## 激活
本质上这个正式的 release 版本和申请试用的版本在功能上没啥差别 (都是 `Cerberus` 版本)，除了加密器的有效期 14 天以及加密文件的有效期 36 小时之外。

不过 release 版本在使用之前是需要先执行激活的，而且一旦激活之外，接下来就只能在这一台机器上才能加密了
<!--more-->
如果没有激活就使用的话，就会报这个问题:
```html
[root@localhost ioncube_encoder5_cerberus_12.0]# bin/ioncube_encoder56_12.0_64 test.php -o test-en.php --add-comment "Software written by Zach"

The Encoder is currently unlicensed.

Use the --activate option to acquire a new license.
```

激活也很简单， 加个 `--activate` 就行了，前提是激活的机器要可以联网
```html
[root@localhost ioncube_encoder5_cerberus_12.0]# ./ioncube_encoder.sh --activate
The license has been activated successfully.
```

这样子就激活好了，不到一分钟。 这个过程就是将这些加密器跟你的机器硬件码进行绑定，然后保存在 ionCube 服务器那边记录。

而且正式版本，确实在生成加密后文件的注释上，也没有那个 申请试用版本的 那个注释了，变成只有你自己制定的注释
```text
[root@localhost ioncube_encoder5_cerberus_12.0]# head test-en.php
<?php //002e9
// Software written by Zach
if(extension_loaded('ionCube Loader'))...
```

## 指令详情
他的加密指令其实非常的多，他的下载包里面有一个用户引导的 pdf 文件: `USER-GUIDE.pdf`, 这个 pdf 有 82 页，里面就是在讲这些加密指令的用法， 今天我们就来啃一啃， 里面有一些是用得到的，我会稍微讲一下， 不太用到的，就忽略

他有两种指令的执行方式:
1. 一种是统一使用 shell 脚本的方式来指定 php 版本:
```text
./ioncube_encoder.sh -81 hello.php -o hello_enc.php
```

2. 第二种就是直接指定对应的加密器，比如
```text
ioncube_encoder source.php -o target.php
```

接下来的示例统一用第二种方式来操作

### 1. 加密文件或者目录
有两种用法:
```text
ioncube_encoder [options] source -o target
```
或者
```text
ioncube_encoder [options] sources --into target
```
对于 `-o` 指令来说， `source` 和 `target` 都可以是单个文件或者是目录， 如果 source 是源文件，那么 target 就是加密文件，如果 source 是源目录的话， target 就是新建的加密后对应的目录

对于 `--into` 来说， `sources` 可以是单个文件或者是目录， 然后将加密生成的文件或者目录，放入已存在的 target 目录中(不存在就新建)。

而且使用 `--into` 可以传多个 source，比如:
```text
ioncube_encoder /project/file1.php /extra/file2.php --into /encoded-project
```

### 2. 添加注释
```text
--add-comment="Encoded by ionCube"
```
这个就会在加密后的 php 文件加上注释，比如:
```text
[root@localhost demo]# ../bin/ioncube_encoder74_12.0_64 test.php --into targetDir --add-comment="Encoded by Zachke"
[root@localhost demo]# head targetDir/test.php 
<?php //002e2
// Encoded by Zachke
if(extension_loaded('ionCube Loader'))
```
如果要支持多行，就多个该指令， 或者可以指定 文本文件也行:
```text
--add-comment "Copyright New FooBar Inc. 2021" --add-comment "All Rights Reserved"
--add-comments custom/comments.txt
```

### 3. 加密非 php 文件
默认是指定加密 `.php` 后缀结尾的文件, 如果是非 php 文件(或者文件夹)的话，也是可以加密的，使用 `--encode` 来指定， 比如我加密一个 test.txt 文件
```text
[root@localhost demo]# cat test.txt
hello ZachKe!!
[root@localhost demo]# ../bin/ioncube_encoder74_12.0_64 test.txt --into targetDir --add-comment="Encoded by Zachke" --encode test.txt
[root@localhost demo]# head targetDir/test.txt
<?php //002e2
// Encoded by Zachke
if(extension_loaded('ionCube Loader'))
```
这个有支持正则:
- `--encode "*.inc"`
- `--encode "config/*"`
- `--encode "doc[0-7]"` -> 包含 doc0, doc1, doc2, … doc7

### 4. 忽略某些文件或者文件夹
可以用 `--ignore` 来忽略某些文件夹，使其不加密。 (不加密，也就是不会生成对应的文件夹)
```text
[root@localhost demo]# cd originDir/ && ls
test2.php  test.php
[root@localhost demo]# ../bin/ioncube_encoder74_12.0_64 originDir -o enDir  --add-comment="Encoded by Zachke" --ignore test2.php
[root@localhost demo]# cd enDir/ && ls
test.php
```
> 这个也支持正则，跟上面一样，如果要忽略多个，就可以设置多个该指令

如果要忽略的文件夹中，刚好有一个文件不要忽略的话，可以增加 `--keep` 来指定要保留的文件, 下例就是忽略掉 tmp 目录中除了 README.txt 的所有文件
```text
--ignore project/tmp/ --keep README.txt
```

### 5. 加密模板文件
```text
ioncube_encoder --encrypt "*.tpl" /project --into /encoded-projects
```
或者加密 xml 文件也行

### 6. 不想加密但是要拷贝文件过去
上面使用 `--ignore` 虽然在加密的时候会忽略，但是在加密之后的目录中，也没有这个文件，所以如果要忽略加密，但是源文件也要在加密后的目录的话，要用 `--copy` 指令
```text
[root@localhost demo]# rm -rf enDir
[root@localhost demo]# ../bin/ioncube_encoder74_12.0_64 originDir -o enDir  --add-comment="Encoded by Zachke" --copy test2.php
[root@localhost demo]# cd enDir && ls
test2.php  test.php
[root@localhost enDir]# head test2.php
<?php
error_reporting(E_ALL);
```

### 7. 混淆加密
加密的时候，还可以选择是否要将代码进行混淆 `--obfuscate` (混淆之后，可能代码会有执行的问题)
```text
--obfuscate all
--obfuscate linenos,functions,classes
```

### 8. 设置加密文件的过期时间
```text
--expire-in <period>
--expire-on <yyyy-mm-dd>
```
前者是相对时间, 单位包括(s,m,h,d -> 秒，分，时，天):
```text
--expire-in 7d   -> 七天后过期
--expire-in 8h   -> 八小时后过期
```
后者是绝对时间:
```text
--expire-on 2023-07-27 -> 2023-07-27 过期
```

### 9. 限制加密后文件的运行 server
可以将加密后的文件限制为仅加载到具有特定 IP 地址和/或网站域名， 他的 server 取的是 php 的这个方法: `$_SERVER['SERVER_NAME']`
```text
--allowed-server www.foo.com  -> 只允许在 www.foo.com 上运行代码
--allowed-server www.foo.com,www.bar.com -> 只允许在 www.foo.com 和 www.bar.com 上运行代码
--allowed-server 192.168.1.4 -> 限制 ip
--allowed-server 192.168.1.4,192.168.1.20 -> 限制 ip
--allowed-server 192.168.1.20-192.168.1.25 -> 限制 ip
--allowed-server '{00:01:02:06:DA:5B}' -> 限制 mac 地址
--apply-file-user <user id/name> -> 指定 linux 运行加密文件的 user
--apply-file-group <group id/name> -> 指定 linux 运行机密文件的 group
```
当然可以结合起来进行更强的限制, 比如以下要 3 个都满足才能运行
```text
--allowed-server 'www.foo.com@192.168.1.1{00:02:08:02:e0:c8}'
```

接下来简单示例，假设我生成的时候，指定允许的 ip 是 `192.168.1.1`
```text
[root@localhost demo]# ../bin/ioncube_encoder56_12.0_64 test.php -o test-en.php  --add-comment="Encoded by Zachke" --allowed-server '192.168.1.1'
```
这时候我运行加密文件的时候，就会报这个错误:
```text
[Fri Oct 21 03:41:10 2022] PHP Fatal error:  <br>The encoded file <b>/root/enDir/test-en.php</b> is not permissioned for this server in Unknown on line 0
[Fri Oct 21 03:41:10 2022] 127.0.0.1:50240 [500]: /test-en.php - <br>The encoded file <b>/root/enDir/test-en.php</b> is not permissioned for this server in Unknown on line 0
```
而我当前运行的局域网 ip 是 `192.168.40.72`, 所以我如果指定这个 ip 来生成的话:
```text
[root@localhost demo]# ../bin/ioncube_encoder56_12.0_64 test.php -o test-en-2.php  --add-comment="Encoded by Zachke" --allowed-server '192.168.40.72'
```
这时候就不会报这个错误:
```text
[root@localhost demo]# curl http://localhost:8011/test-en-2.php
<h1 style="text-align: center;">welcome  test kbz DNMP !!
```


### 10. 使用许可证 license 来加密
我们还可以结合我们自己生成的 license 来进行加密。

所以首先要先生成一份许可证文件，
```text
make_license --passphrase <mypass> [options] -o <output-path>
```
可以指定一些限制条件，比如过期时间，限制 server 等等 (就跟 php 加密的指令一样)，ionCube 给的加密包中，其中就有包含生成许可证文件的程序，有两个，一个是 `make_license`, 一个是 64 位的 `make_license_64`

生成许可证之后，就可以用这个指令进行加密了:
```text
--with-license <path>
```

#### 1. demo 1
接下来举个例子，我先生成一个许可证, 这边有两个参数，一个是生成的密钥，一个是可以设置许可证的过期时间为 6 天
```text
[root@localhost demo]# ../make_license_64 --passphrase abc123456 -o license_64.txt --expire-in 6d
[root@localhost demo]# cat license_64.txt 
------ LICENSE FILE DATA -------
TV4T0X720hezwwcBmG4bBrlg822dmi7R
5H6swAK1E3IuiYMoPNgsaQpyRkMIjrvH
hZtn/ObIdsGO1hjlXiI/prsKCzcFSbnu
q1VRd5oobL999Tt9Swk/2VV4q2fzByW4
hycYT6SJeZHyczUYyqe/Y+JsBIR6p41Z
K5JBIsi+CE+57A4FAX9JWwBUZtwCStVf
i1W0355Lr4oNsNysH5Iu3euv5TZCuoGn
ot8Fuea7HziaemAk6vHCuXuJtnV+gILu
NfNPCrRHyg9/BOk/JCqGpZyE
--------------------------------
```
有了许可证之后，我们就可以用这个许可证进行 php 代码的加密了:
```text
[root@localhost demo]# ../bin/ioncube_encoder56_12.0_64 originDir -o enDir  --add-comment="Encoded by Zachke" --with-license license_64.txt --passphrase abc123456 --license-check auto
```
这边注意两个细节， 一个是 `--with-license` 后面的许可证文件是相对路径， 一个是 `--passphrase` 是跟上面生成许可证的密钥一致。

这时候 enDir 里面的 php 文件就被加密了，不过因为 enDir 目录没有这个许可证，所以如果将 enDir 当做 php 的根目录启动的话:
```text
[root@localhost enDir] php -S 0.0.0.0:8011 -t ./
```
然后再访问里面的 php 文件，这时候就会出现报这个错:
```text
[root@localhost enDir]# curl http://localhost:8012/test_5.php
```
```text
[Thu Oct 20 10:46:10 2022] PHP Fatal error:  <br>The encoded file <b>/root/enDir/test_5.php</b> requires a license file <b>license_64.txt</b>. in Unknown on line 0
[Thu Oct 20 10:46:10 2022] 127.0.0.1:41998 [500]: /test_5.php - <br>The encoded file <b>/root/enDir/test_5.php</b> requires a license file <b>license_64.txt</b>. in Unknown on line 0
```
提示需要这个 `license_64.txt`, 然后接下来我们将这个 license 文件放到这个当前 php 启动的根目录下，再访问一下:
```text
[root@localhost enDir]# curl http://localhost:8012/test_5.php
<h1 style="text-align: center;">welcome  test kbz DNMP !!</h1
```

这时候就可以发现正常访问了，但是这个还不算 license 的真正用途，license 的用途主要用于，我只需要加密一次 php 程序之后，就可以通过 license 来控制当前 php 文件能不能运行。

比如说我交付一个服务到用户的服务器， 但是这个用户购买这个服务只购买了一年，一年到期了，如果他不续费的话， 那么 php 程序就不能再运行了(因为 license 过期了)， 但是他如果到期后续费的话， 那么我只需要再给他一个新的 license 文件，让他替换旧的 license 文件，这样子 php 程序又可以再跑一年。 

所以 license 的价值就是在于可以自由控制加密后的 php 文件什么时候可以运行，什么时候不能运行，在 php 代码只加密一次的情况下， 可以用 license 来灵活控制， 而不需要频繁加密 php 程序。 下面给个 demo:

#### 2. demo 2
1. 首先先生成一个很快过期的 license， 比如 6 分钟过期:

```text
[root@localhost demo]# ../make_license_64 --passphrase abc123456 -o license_zach.txt --expire-in 6m
[root@localhost demo]# cat license_zach.txt 
------ LICENSE FILE DATA -------
...
```

这时候得到了第一个 license -> `license_zach.txt`

2. 接下来用这个 license 对 php 进行加密

```text
[root@localhost demo]# ../bin/ioncube_encoder56_12.0_64 originDir -o enDir  --add-comment="Encoded by Zachke" --with-license license_zach.txt --passphrase abc123456 --license-check auto
```

这时候就得到加密后的 php 目录

3. 这时候将这个目录作为 php 的启动根目录:

```text
[root@localhost enDir]# php -S 0.0.0.0:8011 -t ./
```

然后运行里面的 php 加密文件，发现会报错:

```text
[Fri Oct 21 03:10:39 2022] 127.0.0.1:52654 [500]: /test.php - <br>The encoded file <b>/root/enDir/test.php</b> requires a license file <b>license_zach.txt</b>. in Unknown on line 0
```

因为根目录下找不到对应的license 文件 -> `license_zach.txt`

4. 所以接下来我们将这个 license 文件放到了这个 php 的启动根目录下，这时候运行 php 文件就可以了:

```text
[root@localhost enDir]# curl http://localhost:8011/test.php
<h1 style="text-align: center;">welcome  test kbz DNMP !!
```

5. 然后接下来等了 5 分钟后， 差不多原先的许可证过期了，再访问一下这个 php 文件，就会报这个错误:

```text
[Fri Oct 21 03:17:56 2022] PHP Fatal error:  <br>The encoded file <b>/root/enDir/test.php</b> requires a license file.<br>The license file <b>/root/enDir/license_zach.txt</b> has expired. in Unknown on line 0
[Fri Oct 21 03:17:56 2022] 127.0.0.1:36428 [500]: /test.php - <br>The encoded file <b>/root/enDir/test.php</b> requires a license file.<br>The license file <b>/root/enDir/license_zach.txt</b> has expired. in Unknown on line 0
```

提示许可证已经过期了，无法运行

6. 接下来再执行一下生成许可证的程序，注意密码 和 生成的文件名要一致:

```text
[root@localhost demo]# ../make_license_64 --passphrase abc123456 -o license_zach.txt --expire-in 1h
```

7. 然后将这个重新续费的许可证文件，直接覆盖原先的许可证文件就行了， 这时候重新运行 php 文件，就可以成功了。

#### 3. 关于 passphrase 指令
在之前的实践中，发现 passphrase 这个指令除了可以针对输入的密码当做密钥之外，还可以将某一个文件作为密钥，比如:
```text
[root@localhost demo]# cat zach.key 
abc123456
[root@localhost demo]# ../make_license_64 --passphrase zach.key  -o license_zach.txt --expire-in 1h
[root@localhost demo]# ../bin/ioncube_encoder56_12.0_64 originDir -o enDir  --add-comment="Encoded by Zachke" --with-license license_zach.txt --passphrase zach.key --license-check auto
```
这样子也是可以的。

### 11. loader 没有装的运行提示和操作
```text
--message-if-no-loader <text>
--action-if-no-loader <php code>
```
当 loader 没有安装的时候，我们可以自定义提示语和执行对应的 php 代码，比如
```text
--message-if-no-loader "'No Loader is installed. Please contact support.'"
```

### 12. 自定义事件
```text
--loader-event "<event>=<message>"
```
允许针对某一些事件，去自定义一些信息，包括并不限于加密文件过期，加密报错， license 文件过期或者没找到 等等
```text
--loader-event "expired-file=This software has expired."
```

### 13. 设置元属性/水印
也可以理解为代码水印， 他是针对文件的，不会针对代码，比如设置当前的加密的版本:
```text
--properties "version=1,company='Foo Technologies'"
```















