---
title: 使用 swoole compiler 对 PHP 项目进行加密
date: 2022-09-22 18:12:47
tags: 
- security
- php
categories: php相关
---
## 前言
之前试了一下 `yakpro-po` 混淆加密的测试: {% post_link php-encry-yp %}

接下来我们试一下扩展加密 `Swoole Compiler` 的效果

## Swoole Compiler
`Swoole Compiler` 是swoole官方推出的PHP代码加密和客户端授权解决方案， 通过业内先进的代码加密技术（流程混淆，花指令，变量混淆，函数名混淆，虚拟机保护技术，扁平化代码，sccp优化等）将PHP程序源代码编译为二进制指令，来保护您的源代码，加密技术更先进、更安全。

与 `Zend Guard` 等传统的PHP加密器不同，`Swoole Compiler` 没有软件界面，它提供了API，可将 `Swoole Compiler` 集成到您的打包发布平台中，完全是可编程的。`Swoole Compiler` 相比其他传统的PHP加密器，安全强度更高。

`Swoole Compiler` 使用了特殊定制的ZendVM，与普通的PHP程序运行模式有较大差异。并具有如下特性：
1. 保护程序源码：避免 PHP 源代码泄漏，避免被编辑
2. 提升性能：使用`Swoole Compiler`底层内置了多个编译优化器，可优化 opcode，性能比源码执行有较大提高
3. 授权管理：内置了授权管理功能，可限制PHP程序运行的机器硬件和网络环境

> 还有一点，就是国人开发，响应时间快
<!--more-->
## 价格
官网上面有 4 种定价

![](1.png)

|-|按钮付费|基础版|高级版|旗舰版|
|---|---|---|---|---|
|金额| 3000/年 | 9800/永久 | 19800/永久 | 49800/永久|
|加密方式| 在线 | 离线 | 离线 | 离线 |
|限制解密端机器 | - | - | √ | √ |
|限制代码运行时间 | - | - | √ | √ |
|限制解密端域名 | - | - | √ | √ |
|定制特征码 | - | - | - | √ |

从加密方式来看， 如果要防止源代码泄露, 最好就不要用他们的在线的加密服务器来加密， 只能用私有的服务器来加密， 所以要买的话，只能是基础版起步

## 申请试用尝试一下
### 1. 线上申请
他们官网有提供试用功能，我就用我自己的账号申请了一下，有 10 次的试用机会

![](2.png)

因为用的是他们的在线加密版本，所以我在测试的时候，不能直接用我们的项目代码， 所以就简单的搞了一个 php 文件用来做一次。

因为这个是要跟本地运行程序的 php 环境要一致， 还是之前测试 `yakpro-po` 混淆的那个 docker 环境， php 容器的版本是 `7.4.27`

![](3.png)

所以直接用这个文件压缩成一个 zip 包，然后扔上去，选择 7.4 版本

![](4.png)

点击加密按钮之后，这时候就会下载一个加密后的解压包

![](5.png)

加压之后发现有两个文件，一个是加密后的 php 文件，一个是用于解密扩展的 license 文件

这时候打开，就会发现都是乱码

![](6.png)

所以接下来我们就要在这台服务器上面，看看能不能将这个加密后的 php 文件跑起来
### 2. 下载扩展
既然是扩展加密， 那第一件事，我们肯定是要安装扩展的，也就是安装 `swoole_loader.so` 扩展

首先我们在容器内找到 php 存放扩展文件夹的位置
```text
/www # php -i | grep extension
extension_dir => /usr/local/lib/php/extensions/no-debug-non-zts-20190902 => /usr/local/lib/php/extensions/no-debug-non-zts-20190902
mbstring extension makes use of "streamable kanji code filter and converter", which is distributed under the GNU Lesser General Public License version 2.1.
sqlite3.extension_dir => no value => no value
```
所以扩展的位置就是在 `/usr/local/lib/php/extensions/no-debug-non-zts-20190902`
```text
/usr/local/lib/php/extensions/no-debug-non-zts-20190902 # ls
gd.so         mysqli.so     opcache.so    pdo_mysql.so  sodium.so
```
接下来就是下载 `swoole_loader.so` 扩展然后放进去， 这个官方是有安装向导的: [使用Loader-Helper安装向导](https://www.kancloud.cn/swoole-inc/compiler/1788478)

进入下载页面 `https://compiler.swoole.com/encryptor/download/`, 然后选择对应的 linux 版本， 我们的 php 是 linux 非线程安全版本， 所以下载这个

![](8.png)

下载下来，就将这个文件放到这个扩展目录下
```text
/usr/local/lib/php/extensions/no-debug-non-zts-20190902 # ls -l
total 4312
...
-rwxr-xr-x    1 root     root        165316 Sep 21 15:59 swoole_loader.so
```

### 3. 修改 php.ini
接下来就是修改 php 容器的 `php.ini`, 将扩展加进去, 这边注意一个细节，容器里面的 `php.ini` 是只读文件，是没办法修改的
```text
/usr/local/etc/php # vi /usr/local/etc/php/php.ini
```
修改就会报 `resource busy`, 因为这个文件是有做宿主机挂载映射的，所以我们其实是要修改宿主机对应的 ini 文件的。

通过 `docker inspect php` 可以查看这个配置文件的宿主机挂载路径, 也就是 `/root/dnmp/services/php/php.ini`

![](7.png)

所以我们去宿主机将这个文件修改一下，在 extension 那边补上:
```text
extension=swoole_load.so
```
并且在文件最后面补上指定的 license 的绝对路径 (这边的绝对路径是容器内的绝对路径):
```text
[swoole_loader]                                                                           
swoole_loader.license_files=/www/localhost/sc/swoole-compiler.license
```
然后要重启这个容器来使得配置生效
```text
[root@VM-16-223-centos localhost]# docker restart php
[root@VM-16-223-centos ~]# docker exec -it php /bin/sh
/www # cat  /usr/local/etc/php/php.ini | grep  'swoo'
extension=swoole_loader.so
[swoole_loader]                                                                           
swoole_loader.license_files=/www/localhost/sc/swoole-compiler.license
```
这时候就可以通过 `php -m` 来查看这个扩展是否有生效了
```text
/www # php -m | grep swoo
swoole_loader
```
更直观的就是访问 `phpinfo()`

![](9.png)

### 4. 执行解密后的 php 文件
接下来就执行一下加密后的 php 文件，是否可以正常解析

![](10.png)

可以看到原先是加密的 php 文件可以正常解析了。 

## 总结
从目前的试用来看，应该是没啥问题，如果还要进一步测试，而且考虑到源代码不上传，那么就要至少购买基础版才能继续了

---

参考资料:
- [常见问题](https://www.kancloud.cn/swoole-inc/compiler/1788479)
- [Swoole Compiler 文档](https://www.kancloud.cn/swoole-inc/compiler/1788476)
- [Swoole Compiler 加密 Drupal 产生的一些问题](https://segmentfault.com/a/1190000019314415)
- [Swoole Compiler 加密PHP源代码（简版）](https://blog.csdn.net/qq_16149125/article/details/126075633)