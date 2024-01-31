---
title: php 开启 opcache 来降低负载并提高抗并发能力
date: 2023-11-21 17:58:34
tags: php
categories: php相关
---
## 前言
之前通过 {% post_link php-fpm-opt %} 有针对 php-fpm 的配置进行了调优，有起了一些效果，比如随着处理的进程加大，响应时间会快一点。 但是对于一些并发比较高的场景，还是会有 499 的情况，尤其是负载比较高的情况下，CPU 很容易跑满，php 子进程的响应速度也会越来越慢

当然最粗暴的就是加大服务器硬件配置，然后增大 `pm.max_children` 的值，硬抗，不过还有一种更合适的方式，就是开启 opcache 来降低服务器负载并提高抗并发能力。

## OpCode
opcache 其实就是 OpCode + cache，因此要先了解一下什么是 OpCode。

php 是一种解释型语言，它的执行可分为以下几个流程:
1. Scanning(Lexing) ,将PHP代码转换为语言片段(Tokens)
2. Parsing, 将Tokens转换成简单而有意义的表达式
3. Compilation, 将表达式编译成字节码 (OpCode)
4. Execution, 顺次执行 OpCode，每次一条，从而实现PHP脚本的功能。
<!--more-->

![1](1.png)

这样一来的话，对于同一个文件，反复请求，就要不断解析、编译和执行PHP脚本，消耗过多资源。为了解决这个问题，OpCode 缓存技术应运而生

## OPcache
当解释器完成对脚本代码的分析后，便将它们生成可以直接运行的中间代码，也称为操作码（Operate Code，opcode）。

Opcode 缓存就是将编译好的Opcode码缓存在内存中，下次再执行这个脚本时，直接会去找到缓存下的opcode码，直接执行最后一步，而不再需要中间的步骤了，其目地是避免重复编译，减少 CPU 和内存开销。

![1](2.png)

在 PHP 5.5 之前，不少扩展都实现了字节码的缓存，比如 Zend 公司开发的 Zend Optimizer Plus。

该扩展也随着 PHP 5.5 一起发布，并改名为 OPcache。也就是说，从 5.5 开始，OPcache 成了 PHP 的默认绑定扩展。该扩展通过将 PHP 脚本预编译的字节码存储到共享内存中，省去了每次加载和解析 PHP 脚本的开销。

## OPcache 参数说明
一样直接看官方文档: [OPcache 配置](https://www.php.net/manual/zh/opcache.configuration.php)

这边挑几个平时配置用的比较多的说明一下:
### 1. opcache.enable
- **默认值**: 1
- **说明**: 启用操作码缓存。如果禁用此选项，则不会优化和缓存代码。 在运行期使用 ini_set() 函数只能禁用 opcache.enable 设置，不可以启用此设置。 如果在脚本中尝试启用此设置项会产生警告。新的 php 版本是默认开启的，但是旧的比如 `5.6.x` 默认是关闭的

### 2. opcache.enable_cli
- **默认值**: 0
- **说明**: 仅针对 CLI 版本的 PHP 启用操作码缓存。 通常被用来测试和调试

### 3. opcache.memory_consumption
- **默认值**: 128
- **说明**: OPcache 的共享内存大小，以兆字节为单位。最小允许值为 "8"。如果设置的值小于最小值，则强制设置为允许的最小值。

### 4. opcache.interned_strings_buffer
- **默认值**: 8
- **说明**: 存储预留字符串的内存大小。PHP 解释器在背后会找到相同字符串的多个实例，把这个字符串保存在内存中，如果再次使用相同的字符串，PHP 解释器会使用指针。这么做能节省内存。默认情况下，PHP 驻留的字符串会隔离在各个 PHP 进程中。这个设置能让 PHP-FPM 进程池中的所有进程把驻留字符串存储到共享的缓冲区中，以便在 PHP-FPM 进程池中的多个进程之间引用驻留字符串。

### 5. opcache.max_accelerated_files
- **默认值**: 10000
- **说明**: OPcache 哈希表中可存储的脚本文件数量上限。真实的取值是在质数集合{ 223, 463, 983, 1979, 3907, 7963, 16229, 32531, 65407, 130987, 262237, 524521, 1048793 } 中找到的第一个大于等于设置值的质数。设置值取值范围最小值是 200，最大值是 1000000。超出范围的值将限制为允许的值。

### 6. opcache.validate_timestamps
- **默认值**: 1
- **说明**: 如果启用，那么 OPcache 会每隔 `opcache.revalidate_freq` 设定的秒数 检查脚本是否更新。 如果禁用此选项，你必须使用 `opcache_reset()` 或者 `opcache_invalidate()` 函数来手动重置 OPcache，也可以 通过重启 Web 服务器来使文件系统更改生效。 

### 7. opcache.revalidate_freq
- **默认值**: 2
- **说明**: 检查脚本时间戳是否有更新的周期，以秒为单位。 设置为 0 会导致针对每个请求， OPcache 都会检查脚本更新。如果 `opcache.validate_timestamps` 配置指令设置为禁用，那么此设置项将会被忽略

### 8. opcache.fast_shutdown
- **默认值**: 0
- **说明**: 如果启用，则会使用快速停止续发事件。 所谓快速停止续发事件是指依赖 Zend 引擎的内存管理模块 一次释放全部请求变量的内存，而不是依次释放每一个已分配的内存块。从 PHP 7.2.0 开始, 默认执行


## OPcache 相关配套函数
PHP 官方提供了与 OPcache 相关的函数，具体文档: [OPcache 函数](https://www.php.net/manual/zh/book.opcache.php)

|函数名|说明|
|---|---|
| [opcache_compile_file](https://www.php.net/manual/zh/function.opcache-compile-file.php) | 该函数可以用于在不用运行某个 PHP 脚本的情况下，编译该 PHP 脚本并将其添加到字节码缓存中去。 该函数可用于在 Web 服务器重启之后初始化缓存，以供后续请求调用。|
| [opcache_get_configuration](https://www.php.net/manual/zh/function.opcache-get-configuration.php) | 该函数将返回缓存实例的配置信息。|
| [opcache_get_status](https://www.php.net/manual/zh/function.opcache-get-status.php) | 该函数将返回内存中缓存实例的状态信息。不会返回有关文件缓存的任何信息。|
| [opcache_invalidate](https://www.php.net/manual/zh/function.opcache-invalidate.php) | 废除脚本缓存|
| [opcache_is_script_cached](https://www.php.net/manual/zh/function.opcache-is-script-cached.php) | 该函数用于检测 PHP 脚本是否已经被缓存|
| [opcache_reset](https://www.php.net/manual/zh/function.opcache-reset.php) | 该函数将重置整个字节码缓存。在调用 opcache_reset() 之后，所有的脚本将会重新载入并且在下次命中的时候重新解析。此函数仅重置内存中的缓存，不会重置文件缓存。|

利用这些函数，我们可以手动清空 OPcache，或者也可以将其封装成脚本，在代码更新后自动执行脚本.

或者说利用这些函数来创建可视化操作面板, 比如后面会说到的这个项目: [Opcache Status](https://github.com/rlerdorf/opcache-status)

## 本地开启 OPcache
因为我的本地环境是 windows 环境，采用了[phpstudy](https://www.xp.cn/) 这个集成环境来安装 php，版本是 `5.6.9`:
```text
>php -v
PHP 5.6.9 (cli) (built: May 13 2015 19:29:00)
Copyright (c) 1997-2015 The PHP Group
Zend Engine v2.6.0, Copyright (c) 1998-2015 Zend Technologies
```
查看当前启用的扩展是否有包含 OPcache:
```text
>php -m | grep OPcache
```
发现并没有安装，但是我们可以在 phpstudy 面板中，针对当前安装的这个 php 版本进行配置，并开启 OPcache，默认是没有开启

![1](3.png)

![1](4.png)

这时候就可以看到开启了
```text
>php -m | grep OPcache
Zend OPcache
Zend OPcache
```
可以在 `php.ini` 看到这一节配置
```text
[opcache]
zend_extension=php_opcache
opcache.enable=1
opcache.enable_cli=1
opcache.memory_consumption=128
opcache.interned_strings_buffer=8
opcache.max_accelerated_files=10000
opcache.max_wasted_percentage=5
opcache.revalidate_freq=60
opcache.use_cwd=1
opcache.validate_timestamps=1
opcache.save_comments=1
opcache.enable_file_override=Off
opcache.fast_shutdown=1
opcache.mmap_base=0x20000000
```
参数就不再说了， 后面具体开启的时候，会调参数。 这样子其实 OPcache 就开起来了，可以直接执行 php 指令来查看:
```text
$ php -r "var_dump(opcache_get_status());"
array(8) {
  ["opcache_enabled"]=>
  bool(true)
  ["cache_full"]=>
  bool(false)
```

## 服务器开起来
线上跟本地差不多，一样查看 php 的版本和是否开启这个模块，如果使用 `php -m | grep OPcache`, 发现还是没有的话，可以查看是否有编译这个模块:
```text
[kbz@VM-16-9-centos ~]$ php -i | grep configure | grep opcache
Configure Command =>  './configure'  '--prefix=/usr/local/php-5.6.34' '--with-config-file-path=/usr/local/php-5.6.34/etc' '--with-fpm-user=www' '--with-fpm-group=www' '--disable-debug' '--disable-rpath' '--enable-fpm' '--enable-inline-optimization' '--enable-shared' '--enable-opcache' '--enable-pcntl' '--enable-shmop' '--enable-sysvmsg' '--enable-sysvsem' '--enable-sysvshm' '--enable-mbstring' '--enable-sockets' '--enable-soap' '--enable-zip' '--enable-calendar' '--enable-bcmath' '--enable-exif' '--enable-intl' '--enable-mysqlnd' '--with-libdir=lib64' '--with-mysql=mysqlnd' '--with-mysqli=mysqlnd' '--with-pdo-mysql=mysqlnd' '--with-openssl' '--with-zlib' '--with-bz2' '--with-curl' '--with-gd' '--with-jpeg-dir' '--with-zlib-dir' '--with-png-dir' '--enable-gd-native-ttf' '--with-freetype-dir' '--with-gettext' '--with-iconv' '--with-mcrypt' '--with-mhash' '--with-ldap' '--with-readline'
```
这种情况，就有编译了。正常情况下，是可以直接找到这个 so 文件的:
```text
[kbz@VM-16-9-centos ~]$ locate opcache
...
/usr/local/php-5.6.34/lib/php/extensions/no-debug-non-zts-20131226/opcache.so
```

> 如果没有安装的话，就得重新安装 php，并且携带参数 `--enable-opcache`

所以接下来一样在 php.ini 中将 OPcache 开起来就行了, 在 `[opcache]` 这一栏下
```text
[opcache]

zend_extension=opcache.so
opcache.enable = 1
opcache.memory_consumption=1024
opcache.interned_strings_buffer=32
opcache.max_accelerated_files=80000
opcache.revalidate_freq=30
opcache.fast_shutdown=1
opcache.enable_cli=1
```
注意，这边有几个参数是有调整的，比如将缓存的内存设置为 1024M，将存储预留字符串的内存大小设置为 32M，将 可存储的脚本文件数量上限设置为 80000，最后将缓存刷新的时间设置为 30s。最后重启一下 php-fpm 即可

## 安装状态面板
既然 opcache 已经开启了，那么接下来我们怎么有效的查看它的使用情况呢，有一个开源的库，可以简单的显示它的面板情况: [Opcache Status](https://github.com/rlerdorf/opcache-status)

我们只需要将其下载到服务器的某一个目录下
```text
cd /data/wwwroot/
git clone https://github.com/rlerdorf/opcache-status.git --depth=1
cd opcache-status
```
然后在 nginx 那边增加一个路由，然后转发解析到这个目录下的 `opcache.php` 就行了，注意最好只能内网才能访问
```text
location ^~ /opcache  {
    root    /data/wwwroot/opcache-status/;
    try_files $uri /opcache.php;
    fastcgi_pass 127.0.0.1:9000;
    include    fastcgi.conf;
    allow 127.0.0.1;
    allow xxx;
    allow xxx;
    deny all;
}
```
就可以直接走域名访问了

![1](5.png)

如果是本地测试，就不需要走 nginx 了，直接 php 起端口就行了:
```text
php -S localhost:8003 opcache.php
```
就可以在浏览器访问 `localhost:8003` 查看了

而且线上优化之后，其实对负载的优化是非常明显的，cpu 和 内存都降了一节

![1](6.png)

## OPcache 缓存刷新/重置的几种方式
1. php 执行 `opcache_reset()` 方法，重置整个 Opcode 缓存,所有的PHP脚本将会被重新解析再预编译为 Opcode
2. php 执行 `opcache_invalidate()`方法, 清除指定脚本缓存,可以传递两个参数,一个是刷新文件路径,一个是force字段, 如果 force 没有设置或者传入的是 FALSE，那么只有当脚本的修改时间 比对应Opcode的时间更新时，脚本的缓存才会失效
3. 重启 php-fpm `service php-fpm restart`，不过这种情况请求会中断， 或者重载 `service php-fpm reload`，重载相对于重启则平顺很多,不会导致用户请求直接中断,相对来说风险低很多,但是 php-fpm 收到reload信号，便会向所有子进程发送 SIGGUIT 信号，同时注册一个定时器，在规定的时间之内子进程没有退出，接着在发送 SIGTERM 信号，结束子进程。如果在一秒之内子进程还是没结束 直接发送SIGKILL 强制杀死，这时候 nginx 就会返回 502 错误码
4. 配置 `opcache.validate_timestamps=1` 和 `opcache.revalidate_freq=xx`, 以秒为单位,那么 OPcache 会每隔 `opcache.revalidate_freq` 设定的秒数 检查脚本是否更新, 如果发现内存中的时间晚于文件更新时间，将会更新缓存
5. OPcache 达到最大内存消耗，也就是 `opcache.memory_consumption` 这个配置，Opcache 就会尝试重新启动缓存
6. OPcache 达到最大加速文件，也就是 `opcache.max_accelerated_files` 这个配置，Opcache 就会尝试重新启动缓存
7. OPcache 浪费的内存超过了配置的 `opcache.max_wasted_percentage`，Opcache 就会尝试重新启动缓存, 手动清空 wasted memory 只能重启 PHP-FPM 或使用 PHP 函数 opcache_reset()

> 需要注意的是，当 PHP 以 PHP-FPM 的方式运行的时候，OPcache 的缓存是无法通过 php 命令进行清除的，只能通过 http 或 cgi 到 php-fpm 进程的方式来清除缓存

> 如果 Opcache 不设置过期时间的话，应尽量避免流量高峰期发布代码，因为这时候清理缓存，可能会导致重复新建缓存。

## OPcache 对性能的影响
1. 如果性能瓶颈不在于CPU和内存，而在于I/O操作，比如数据库查询带来的磁盘I/O开销，那么 OPcache 的性能提升是非常有限的。
2. 刚启动或缓存被清空时，如果服务器流量大可能会导致 [thundering herd problem](https://en.wikipedia.org/wiki/Thundering_herd_problem) 或缓存猛击，许多请求同时生成相同的缓存条目


## 后续再优化
后续还有几个可以再优化的地方:
### 1. 生产环境不设置缓存定时刷新时间
将刷缓存的时间(`opcache.validate_timestamps`)设置为 0, 表示永不刷新，然后在每次 Jenkins 构建 php 程序的时候，调用一个脚本来手动刷新缓存

```text
#!/bin/bash
WEBDIR=/var/www/html/
RANDOM_NAME=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c 13)
echo "<?php opcache_reset(); ?>" > ${WEBDIR}${RANDOM_NAME}.php
curl http://localhost/${RANDOM_NAME}.php
rm ${WEBDIR}${RANDOM_NAME}.php
```

> 上面的脚本意思就是，先生成一个随机的字符串 `RANDOM_NAME`,长度为13，包含大写字母、小写字母和数字,接下来创建一个新的PHP文件，文件名为 `${RANDOM_NAME}.php`，并且这个文件在 `${WEBDIR}` 目录下。这个PHP文件的内容是`<?php opcache_reset(); ?>`, 接下来使用 curl 命令访问这个新创建的PHP文件。由于这个文件的内容是 `opcache_reset()`，所以这会导致服务器上的`OPcache`被重置, 最后，删除这个临时创建的PHP文件

总的来说，这个脚本的目的是在Web服务器上重置 OPcache ，而不需要在服务器的PHP代码中硬编码这个功能, 很适合在 Jenkins 构建结束的时候直接执行这个脚本来刷新 OPcache，因为如果不刷新的话，就有可能会在 Jenkins 构建更新程序 php 文件的时候，出现新旧 php 缓存不一致，导致当前请求报 500 的情况。

或者直接走 php 的 CLI 指令，不过这种方式如果是在 php-fpm 下是没有效果的，只能用上面那种 http 请求 php 文件的方式。
```text
#!/bin/bash
php -r "opcache_reset();"
```

如果要确定执行刷新是否有效果的话，可以查看状态面板的 `last_restart_time`, 如果刷新成功的话，`last_restart_time` 就是刷新时间

![1](7.png)

### 2. 增加 hugepage 配置
启用 `opcache.huge_code_pages` 将 PHP 代码（文本段）拷贝到 HUGE PAGES 中。这应该会提高性能，但是需要适当的 OS 配置。自 PHP 7.0.0 起可在 Linux 上使用，自 PHP 7.4.0 上可在 FreeBSD 上使用。
```text
opcache.huge_code_pages=1
```

同时需要在 `/etc/sysctl.conf` 设置系统分配的大页面（huge pages）的数量，比如下面就分配了512个大页面
```text
[kbz@VM-16-137-centos ~]$ cat /etc/sysctl.conf | grep huge
vm.nr_hugepages=512
```

### 3. [配置 preload 用于缓存预热](https://www.php.net/manual/zh/opcache.preloading.php)
从 PHP 7.4.0 起，PHP 可以被配置为在引擎启动时将一些脚本预加载进 opcache 中。在那些文件中的任何函数、类、 接口或者 trait （但不包括常量）将在接下来的所有请求中变为全局可用，不再需要显示地包含它们。这牺牲了基准的 内存占用率但换来了方便和性能（因为这些代码将始终可用）。它还需要通过重启 PHP 进程来清除预加载的脚本， 这意味着这个功能仅在生产环境中有实用价值，而非开发环境。

需要注意的是，性能和内存之间的最佳平衡因应用而异。 “预加载一切” 可能是最简单的策略，但不一定是最好的策略。 此外，只有当不同的请求间有持久化的进程时，预加载才有用。这意味着，虽然在启用了 opcache 的命令行脚本中可以使用预加载， 但这通常是没有意义的。

```text
opcache.preload=preload.php
```

> 注意: 如果一台服务器有多个 php 项目时，可能会引起未知错误，开启需谨慎

---
参考资料:
- [opcache 官方文档](https://www.php.net/manual/zh/opcache.configuration.php)
- [opcache的配置与使用](https://www.phpnanshen.com/article/158.html)
- [PHP Opcache 注意事项以及调优](https://learnku.com/php/t/34638)
- [【PHP】细说Opcache](https://www.wuwenjia.com/archives/83)






