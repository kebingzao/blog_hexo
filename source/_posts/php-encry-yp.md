---
title: 使用 yakpro-po 对 PHP 项目进行混淆加密
date: 2022-09-20 16:57:38
tags: 
- security
- php
categories: php相关
---
## 前言
之前有讨论 {% post_link php-encry %}, 所以本次我们尝试使用 `yakpro-po` 对我们的 PHP 项目进行混淆加密
- [YAK Pro - Php Obfuscator](https://github.com/pk-fr/yakpro-po)
- [PHP Parser](https://github.com/nikic/PHP-Parser/tree/4.x/)

## 环境需求
因为 `yakpro-po` 是基于 `PHP Parser 4.x` 的语法解析器做的。 而 `PHP Parser 4.x` 库的运行环境要求是要在 PHP 7.0 以上环境才能运行， 不过混淆后的代码可以在 PHP 5.2 到 7.3 之间都可以工作。

而我们的 PHP 程序是跑在 PHP 5.6 上面的。 所以可以满足混淆之后运行代码的环境。 而且为了更直观的表现出混淆加密和针对加密文件的运行可以在不同的服务器和 PHP 环境上，本次的测试用两台测试机器来测试，这样子会显得更直观
1. 一台服务器 -> `混淆加密服务器`，上面安装了 7.4 的 PHP 版本， 采用 docker-compose 安装的，参照: [LNMP一键安装程序](https://github.com/kebingzao/dnmp)， 这一台用来混淆加密 PHP 代码。
2. 一台服务器 -> `加密文件运行服务器`, 用来运行混淆过的 PHP 程序的代码，PHP 5.6 的环境， 看是否可以成功运行。

> 嫌麻烦的可以直接用一台，然后在各自的 docker 里面跑
<!--more-->
## 初始化并且简单测试 yakpro-po
因为 `混淆加密服务器` 已经用 docker 装载了 `php+nginx` 了，并且 localhost 的目录也挂载到宿主机这个目录了: `/root/dnmp/www/localhost` (php 运行目录)

因为 `yakpro-po` 需要用到 php 的 cli 程序，所以直接将库的内容拉下来放到  `/root/dnmp/www/localhost` 下面 (对应 php docker 容器的 `/www/localhost`)

按照教程操作(在宿主机操作):
```text
cd /root/dnmp/www/localhost
mkdir yp
cd yp
git clone https://github.com/pk-fr/yakpro-po.git
cd yakpro-po/
git clone https://github.com/nikic/PHP-Parser.git --branch 4.x
chmod a+x yakpro-po.php
```
到这一步，其实 `yakpro-po` 库就下载下来了。 接下来就是将 `yakpro-po` 外链到全局，因为要用到 php 的 cli 所以这一步要到 php 的容器里面处理:
```text
docker exec -it php /bin/sh
cd /usr/local/bin
ln -s /www/localhost/yp/yakpro-po/yakpro-po.php yakpro-po
```

这时在 php 容器内就可以执行 `yakpro-po` 的指令了:
```text
/usr/local/bin # yakpro-po --help
Info:   yakpro-po version = 2.0.14
...
```
接下来我们将 `/root/dnmp/www/localhost` 默认的 index.php， 重新拷贝一份成 test.php，然后试一下 `yakpro-po` 的混淆加密效果 (只要是执行加密操作的指令，都是只能在 php 容器下才行)

```text
/www/localhost # yakpro-po test.php
Info:   Using [/www/localhost/yp/yakpro-po/yakpro-po.cnf] Config File...
Info:   yakpro-po version = 2.0.14
Info:   Process Mode            = file
Info:   source_file             = [/www/localhost/test.php]
Info:   target_file             = [stdout]
Obfuscating /www/localhost/test.php
<?php
/*   __________________________________________________
    |  Obfuscated by YAK Pro - Php Obfuscator  2.0.14  |
    |              on 2022-09-13 18:20:35              |
    |    GitHub: https://github.com/pk-fr/yakpro-po    |
    |__________________________________________________|
*/
goto sHBds; mc8_3: function N21Ki() { goto L2r4Y; h8xPy: goto hKZls; goto G4HmV; GRff1: return $oppnQ["\x76\x65\x72\x73\x69\157\x6e"]; goto G4igu; G4igu: hKZls: goto IN9c9; eL3ew: return "\120\x44\117\x5f\115\131\123\121\x4c\40\xe6\x89\251\345\xb1\225\346\x9c\xaa\xe5\xae\x89\350\xa3\205\40\xc3\227"; goto h8xPy; G4HmV: mwf6G: goto M0ihh; M0ihh: try { goto gdwLS; q7ftK:
```
可以看到默认是直接输出到 std 中。 我们可以输出到文件中:
```text
/www/localhost # yakpro-po test.php  -o test_ob.php
Info:   Using [/www/localhost/yp/yakpro-po/yakpro-po.cnf] Config File...
Info:   yakpro-po version = 2.0.14
Info:   Process Mode            = file
Info:   source_file             = [/www/localhost/test.php]
Info:   target_file             = [test_ob.php]
Obfuscating /www/localhost/test.php
Info:   [variable         ] scrambled   :       10
Info:   [function_or_class] scrambled   :        9
Info:   [method           ] scrambled   :        1
Info:   [property         ] scrambled   :        1
Info:   [class_constant   ] scrambled   :        0
Info:   [constant         ] scrambled   :        0
Info:   [label            ] scrambled   :       66
```
test_ob.php 就是混淆加密后的文件。 这个也是可以正常执行成功的:
```text
[root@VM-16-223-centos localhost]# curl "http://localhost/test_ob.php"
<h1 style="text-align: center;">welcome  test kbz DNMP !!</h1><h2>版本信息</h2><ul><li>PHP版本：7.4.27</li><li>Nginx版本：nginx</li><li>MySQL服务器版本：8.0.28</li><li>Redis服务器版本：Redis 扩展未安装 ×</li><li>MongoDB服务器版本：MongoDB 扩展未安装 ×</li></ul><h2>已安装扩展</h2><ol><li>Core=7.4.27</li><li>date=7.4.27</li><li>libxml=7.4.27</li><li>openssl=7.4.27</li><li>pcre=7.4.27</li><li>sqlite3=7.4.27</li><li>zlib=7.4.27</li><li>ctype=7.4.27</li>
...
```
加密后的文件

![](1.png)

所以简单的单个文件的 加解密应该是没问题的。

### 去掉 commet 注释
正常默认的加密是有添加 commet 注释的，
```text
<?php
/*   __________________________________________________
    |  Obfuscated by YAK Pro - Php Obfuscator  2.0.14  |
    |              on 2022-09-14 15:22:41              |
    |    GitHub: https://github.com/pk-fr/yakpro-po    |
    |__________________________________________________|
*/
```
这个是可以去掉的，具体在 
```text
/root/dnmp/www/localhost/yp/yakpro-po/include/classes/config.php
```
将下面的这个方法改一下，直接返回 空字符串,这样子生成完的加密混淆文件，就不会带这个 注释头, 并且可以正常运行
```text
public function get_comment()
  {
      global $yakpro_po_version;
      $now = date('Y-m-d H:i:s');

      #return sprintf($this->comment,$yakpro_po_version,$now);
      return "";
  }
```

![](2.png)

## 对 php 程序进行加密混淆
我们的 php 程序其实分别用到了两种框架，一个是 yii2 框架, 一个是 lumen 框架。 这次简单先 拿 lumen 框架的这个项目试一下

lumen 框架项目直接上面的命令，不带任何选项，混淆后的代码根本跑不起来，因为默认混淆了类名、命令空间、变量等等，所以要手动添加选项来指定混淆选项。

### 1. 测试 lumen 框架项目加密
以用 lumen 框架的这个项目为例， 因为使用 composer 来管理第三方库， 所以 `vender` 会比较大，先排除 vender 目录掉，不加密 vender 目录，只加密逻辑代码试试。

先尝试用这种混淆方式试试:
```text
yakpro-po ucenter -o ucenter-ob \
--no-obfuscate-function-name  \
--no-obfuscate-class_constant-name \
--no-obfuscate-class-name \
--no-obfuscate-interface-name \
--no-obfuscate-trait-name \
--no-obfuscate-property-name \
--no-obfuscate-method-name \
--no-obfuscate-namespace-name \
--no-obfuscate-label-name
```
也就是我们不混淆以下:
```text
--no-obfuscate-function-name              不混淆函数名
--no-obfuscate-class_constant-name        不混淆类常量名
--no-obfuscate-class-name                 不混淆类名
--no-obfuscate-interface-name             不混淆接口名称
--no-obfuscate-trait-name                 不混淆特征名称
--no-obfuscate-property-name              不混淆属性名称
--no-obfuscate-method-name                不混淆方法名
--no-obfuscate-namespace-name             不混淆命名空间名称
--no-obfuscate-label-name                 不混淆标签名称
```
只混淆这些默认的配置:
```text
--strip-indentation 单行输出
--shuffle-statements 随机播放语句
--obfuscate-constant-name 混淆常量名
--obfuscate-string-literal 混淆字符串文字
--obfuscate-loop-statement 混淆循环语句
--obfuscate-if-statement 混淆 if 语句
--obfuscate-variable-name 混淆变量名
```
最后的执行结果如下:
```text
yakpro-po ucenter -o ucenter-ob --no-obfuscate-function-name --no-obfuscate-class_constant-name --no-obfuscate-class-name --no-obfuscate-interface-name --no-obfuscate-trait-name --no-obfuscate-property-name --no-obfuscate-method-name --no-obfuscate-namespace-name --no-obfuscate-label-name
Info:    Using [/www/localhost/yp/yakpro-po/yakpro-po.cnf] Config File...
Info:    yakpro-po version = 2.0.14
Info:    Process Mode        = directory
Info:    source_directory    = [/www/localhost/ucenter]
Info:    target_directory    = [ucenter-ob]
Obfuscating /www/localhost/ucenter/app/Events/Event.php
...
Info:    [variable         ] scrambled     :      316
Info:    [function_or_class] scrambled     :        0
Info:    [method           ] scrambled     :        0
Info:    [property         ] scrambled     :        0
Info:    [class_constant   ] scrambled     :        0
Info:    [constant         ] scrambled     :       13
Info:    [label            ] scrambled     :     2510
```
随便点一个 controller 看下, 可以看到代码有混淆加密了

![](3.png)

接下来将这一份混淆过的代码覆盖 `加密文件运行服务器` 的对应的代码文件，然后执行看看， 发现有报错了:
```text
[2022-09-19 05:54:28] lumen.ERROR: exception 'ErrorException' with message 'Use of undefined constant Xqy6R - assumed 'Xqy6R'' in /data/server/wwwroot/obfuscated/app/Models/Master/BusinessAccount.php:83
Stack trace:
#0 /data/server/wwwroot/obfuscated/app/Models/Master/BusinessAccount.php(83): Laravel\Lumen\Application->Laravel\Lumen\Concerns\{closure}(8, 'Use of undefine...', '/data/server/ww...', 83, Array)
```
看样子是有一个常量没有找到， 然后对比原文

![](4.png)

这个的定义是在 `php-common` 的 lib 那边定义的， 而混淆加密的时候， 是没有加密 vender 目录的。 导致这边的常量被混淆了之后，

然后又因为 vender 的代码没有被混淆， 所以就找不到。 所以这边有两种情况处理:
1. 混淆的时候，连同 vender 目录一起混淆
2. 混淆的时候，不混淆常量，就是加上  `--no-obfuscate-constant-name` 参数

#### 1.1 不混淆常量
先处理第二种会比较简单，就是不混淆常量， 指令就是:
```text
yakpro-po ucenter -o ucenter-ob --no-obfuscate-function-name --no-obfuscate-class_constant-name --no-obfuscate-class-name --no-obfuscate-interface-name --no-obfuscate-trait-name --no-obfuscate-property-name --no-obfuscate-method-name --no-obfuscate-namespace-name --no-obfuscate-label-name --no-obfuscate-constant-name  --no-strip-indentation 
```
注意，这边除了加上 `--no-obfuscate-constant-name` 参数指定不混淆常量之外， 还加上了 `--no-strip-indentation` 指令指定输出多行，这个是为了更直观的查看混淆后的文件内容

这样子就不会报错了，可以正常执行， 混淆过的代码是这样子的

![](5.png)

#### 1.2 混淆 vender 目录
试一下连同 vender 一起混淆的情况，将 vender 目录一起拷过去`混淆加密服务器`，然后执行同样的混淆指令，结果发现报错了

![](6.png)

结果报错了， 是一个第三方的库的报错??? 。 查了一下 `yakpro-po` 的 github 文档，还真发现了这个问题的 issue: [Hints for preparing your Software to be run obfuscated](https://github.com/pk-fr/yakpro-po#hints-for-preparing-your-software-to-be-run-obfuscated)
{% blockquote sedimentation-fault https://github.com/pk-fr/yakpro-po#hints-for-preparing-your-software-to-be-run-obfuscated %}
If you use the define function for defining constants, the only allowed form is when the
define function has exactly 2 arguments, and the first one is a litteral string!
You MUST disable constants obfuscation in the config file, if you use any other forms
of the define function!
There is no problem with the const MY_CONST = something; form!
{% endblockquote %}
也就是如果是使用 `define` 来定义常量的话， 必须要有两个参数，并且第一个是字符串。 我看了一下这个出问题的文件， 发现它调用的 define 语法都是只有一个参数，所以报错了。

要么就是要改成走 `const MY_CONST = something` 的方式来定义常量也可以。

这个就有点麻烦了， 因为涉及到第三方库， 我们总不可能去修改他的源代码。 所以应该也不能加 常量混淆的方式。

所以接下来试一下， 一样包含 vender 的，但是不走 常量 混淆，看看会不会报错
```text
yakpro-po ucenter -o ucenter-ob \
--no-obfuscate-function-name  \
--no-obfuscate-class_constant-name \
--no-obfuscate-class-name \
--no-obfuscate-interface-name \
--no-obfuscate-trait-name \
--no-obfuscate-property-name \
--no-obfuscate-method-name \
--no-obfuscate-namespace-name \
--no-obfuscate-label-name \
--no-obfuscate-constant-name  \
--no-strip-indentation
```
这次报错是没有报错了， 但是加密好像没有执行完就退出了 (没看到最后的 info 总结)，不知道是不是因为 vender 的第三方包太大了。

![](7.png)

重试了几次还是不行， 都是卡在这边。 后面查了一下，发现应该是 php 文件太多，导致的堆栈溢出了 ([Segmentation fault due to stack overflow in PHP's Garbage Collector (GC)](https://github.com/pk-fr/yakpro-po#known-issues))

{% blockquote sedimentation-fault https://github.com/pk-fr/yakpro-po#known-issues %}
sedimentation-fault reported on issue #75 that a segmentation fault could occure in php's garbage collector when obfuscating many big files in a project:

Trying to obfuscate ~5000 PHP files of ~1000 lines each, yakpro-po stopped after processing ~1600 files 
with a simple (and frustrating) Segmentation fault

Workaround:

There is a stack overflow in garbage collector. The solution is to increase limit for stack.
To see your current limit, type

ulimit -s

I had 8192 - for a task of this size obviously totally undersized...
Change this to something more appropriate, say

ulimit -s 102400

and retry - the segmentation fault is gone! :-)
{% endblockquote %}

然后按照它的要求将 docker 容器的 ulimit 设置了一个比较大的值:
```text
/www/localhost # ulimit -s
8192
/www/localhost # ulimit -s 102400
/www/localhost # ulimit -s
102400
```
但是发现这种方式好像也不行，而且退出容器，重新进去，又变成默认的 8192 了。 后面重新在 docker run 容器的时候， 直接设置 ulimit 的 stack 的值， 这个倒是有生效了。 而且宿主机也要设置
```text
 docker run  --name php7 -d  --ulimit stack=102400000:102400000  -v /root/dnmp/www:/www  dnmp_php
```
```text
[root@VM-16-223-centos localhost]# ulimit -s
8192
[root@VM-16-223-centos localhost]# ulimit -s 102400
[root@VM-16-223-centos localhost]# ulimit -s
102400
[root@VM-16-223-centos localhost]# docker exec -it php7 /bin/sh
/www # ulimit -s
100000
/www # cd localhost/
/www/localhost # yakpro-po ucenter -o ucenter-ob --no-obfuscate-function-name --no-obfuscate-class_constant-name --no-obfuscate-class-name --no-obfuscate-interface-name --no-obfuscate-trait-name --no-obfuscate-property-n
ame --no-obfuscate-method-name --no-obfuscate-namespace-name --no-obfuscate-label-name --no-obfuscate-constant-name  --no-strip-indentation
...
Obfuscating /www/localhost/bak-vender/vendor/symfony/http-kernel/DependencyInjection/ResettableServicePass.php

Fatal error: Allowed memory size of 134217728 bytes exhausted (tried to allocate 16384 bytes) in /www/localhost/yp/yakpro-po/include/functions.php on line 33
```
这时候不会卡住了，但是执行到后面，报了一个内存不够分配的情况，也就是 128M 内存不够用，也就是内存溢出了。 这个应该是 php 本身内存分配的问题， 默认的应该是 128M， 

直接修改 php.ini 的配置，改成 512M ,
```text
/usr/local/etc/php # cat php.ini | grep 'memory_limit'
memory_limit = 512M
```
最后是 vendor 目录也可以混淆了，最后输出的 info
```text
Info:	[variable         ] scrambled 	:    16722
Info:	[function_or_class] scrambled 	:        0
Info:	[method           ] scrambled 	:        0
Info:	[property         ] scrambled 	:        0
Info:	[class_constant   ] scrambled 	:        0
Info:	[constant         ] scrambled 	:        0
Info:	[label            ] scrambled 	:   131570
```
将这些加密后的 php 文件放到原先的目录上，也是可以 work 的。 (有一些第三方库如果还报错，可能就是其他语法的冲突了，可以选择忽略这个第三方库)

### 2. 测试 yii2 框架项目加密
接下来测试 yii2 的结果也是跟 lumen 差不多，如果只用上面的这几个混淆选项的话， 也没有任何问题，所以这边不再赘述。


## 总结
通过对 `yakpro-po` 混淆加密的实测， 发现如果要在我们的项目里面去实践，又不想改太多代码的话，尤其是涉及到第三方库的修改。那么只能设置以下的混淆选项
```text
--strip-indentation           单行输出
--shuffle-statements          随机播放语句
--obfuscate-string-literal    混淆字符串文字
--obfuscate-loop-statement    混淆循环语句
--obfuscate-if-statement      混淆 if 语句
--obfuscate-variable-name     混淆变量名
```
不能混淆以下的配置:
```text
--no-obfuscate-function-name           不混淆函数名
--no-obfuscate-class_constant-name     不混淆类常量名
--no-obfuscate-class-name              不混淆类名
--no-obfuscate-interface-name          不混淆接口名称
--no-obfuscate-trait-name              不混淆特征名称
--no-obfuscate-property-name           不混淆属性名称
--no-obfuscate-method-name             不混淆方法名
--no-obfuscate-namespace-name          不混淆命名空间名称
--no-obfuscate-label-name              不混淆标签名称
--no-obfuscate-constant-name           不混淆常量
```

因为测试的周期时间关系，我并没有将上述那些混淆出问题的配置一个一个去测试和联调，不过如果是复杂度和代码量都比较高的项目， 大概率会出问题。 尤其是一些依赖的第三方库，可能也会改到, 正常调整的话， 如果遇到不好改的，或者调整比较大的，可以配合他的 ignore 参数来忽略这些要加密的配置, 比如一些常量或者函数要调整，但是调用的地方太多了，不太好改，就可以通过这种配置将其忽略混淆:
```text
$conf->t_ignore_constants               = null;         // array where values are names to ignore.
$conf->t_ignore_variables               = null;         // array where values are names to ignore.
$conf->t_ignore_functions               = null;         // array where values are names to ignore.
$conf->t_ignore_class_constants         = null;         // array where values are names to ignore.
$conf->t_ignore_methods                 = null;         // array where values are names to ignore.
$conf->t_ignore_properties              = null;         // array where values are names to ignore.
$conf->t_ignore_classes                 = null;         // array where values are names to ignore.
$conf->t_ignore_interfaces              = null;         // array where values are names to ignore.
$conf->t_ignore_traits                  = null;         // array where values are names to ignore.
$conf->t_ignore_namespaces              = null;         // array where values are names to ignore.
$conf->t_ignore_labels                  = null;         // array where values are names to ignore.

$conf->t_ignore_constants_prefix        = null;         // array where values are prefix of names to ignore.
$conf->t_ignore_variables_prefix        = null;         // array where values are prefix of names to ignore.
$conf->t_ignore_functions_prefix        = null;         // array where values are prefix of names to ignore.

$conf->t_ignore_class_constants_prefix  = null;         // array where values are prefix of names to ignore.
$conf->t_ignore_properties_prefix       = null;         // array where values are prefix of names to ignore.
$conf->t_ignore_methods_prefix          = null;         // array where values are prefix of names to ignore.

$conf->t_ignore_classes_prefix          = null;         // array where values are prefix of names to ignore.
$conf->t_ignore_interfaces_prefix       = null;         // array where values are prefix of names to ignore.
$conf->t_ignore_traits_prefix           = null;         // array where values are prefix of names to ignore.
$conf->t_ignore_namespaces_prefix       = null;         // array where values are prefix of names to ignore.
$conf->t_ignore_labels_prefix           = null;         // array where values are prefix of names to ignore.
```

> 而且因为 php 是边执行边解释，所以还会导致混淆的时候，不会报错，但是后面执行的时候，突然间就报错，这个是很不可控的，如果要避免这种情况，就会有完整的单元测试来覆盖所有的接口和逻辑以保证混淆后的代码没有异常，这个成本也是非常高的

**如果只是基于上述可以 work 的配置的话，结果就是混淆的程度太低了，跟单文件混淆加密没啥区别， 解密难度其实不高。 而如果再加上上述一些混淆程度比较高的配置的话，又会很容易导致代码执行出问题。所以这个要看场景吧，如果是代码规范比较好的，代码量不会太多的是，其实可以用的。**

---

参考资料:
- [YAK Pro - Php Obfuscator](https://github.com/pk-fr/yakpro-po)
- [使用 yakpro-po 实现 Laravel 项目代码混淆加密](https://learnku.com/articles/65068)
- [Docker container does not inherit ulimit from host](https://stackoverflow.com/questions/36298913/docker-container-does-not-inherit-ulimit-from-host)
- [Docker LNMP环境搭建](https://www.awaimai.com/2120.html)
- [LNMP一键安装程序](https://github.com/kebingzao/dnmp)
- [破解由 YAK Pro - Php Obfuscator 混淆PHP代码的手段](https://github.com/demonxian3/crack-yakpro-php)



