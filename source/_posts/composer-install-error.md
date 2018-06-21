---
title: windows 使用composer install 报一些奇怪的错误
date: 2018-05-05 16:23:24
tags: php
categories: php相关
---
之前在windows下装composer依赖的时候，会报一些奇怪的错误，有些跟版本，或者平台还有关系

比如我在装 某一个项目 的composer的时候，就会报这个错误

{% blockquote %}
F:\airdroid_code\api-admin>composer install
Failed loading D:/wamp/bin/php/php5.4.3/zend_ext/php_xdebug-2.2.0-5.4-vc9.dll
Loading composer repositories with package information
Installing dependencies (including require-dev) from lock file
Your requirements could not be resolved to an installable set of packages.

  Problem 1
    - Installation request for mongodb/mongodb 1.1.2 -> satisfiable by mongodb/mongodb[1.1.2].
    - mongodb/mongodb 1.1.2 requires ext-mongodb ^1.2.0 -> the requested PHP extension mongodb is missing from your system.
  Problem 2
    - mongodb/mongodb 1.1.2 requires ext-mongodb ^1.2.0 -> the requested PHP extension mongodb is missing from your system.
    - jenssegers/mongodb v3.2.1 requires mongodb/mongodb ^1.0.0 -> satisfiable by mongodb/mongodb[1.1.2].
    - Installation request for jenssegers/mongodb v3.2.1 -> satisfiable by jenssegers/mongodb[v3.2.1].

  To enable extensions, verify that they are enabled in your .ini files:
    - D:\wamp\bin\php\php5.6.29\php.ini
  You can also run `php --ini` inside terminal to see which files are used by PHP in CLI mode.
{% endblockquote %}

明明我 php ini 文件有开启 mongo 的支持，但是却还是显示这个问题。

后面解决方法就是：加一个参数就可以了 <font color=red>ignore-platform-reqs</font>

{% codeblock %}
composer install --ignore-platform-reqs  
{% endcodeblock %}

这个命令来装，可以忽略掉一些奇怪的问题