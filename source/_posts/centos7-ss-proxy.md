---
title: CentOS7 配置 shadowsocks 全局代理
date: 2019-02-14 10:55:25
tags: Linux
categories: Linux 相关
---
## 前言
之前在使用 gvm 在安装 go 版本 的时候，就有出现因为没法翻墙而导致没办法下载资源的问题。因此 CentOS7 要设置一下代理才行, 因为之前有在腾讯云香港那边搭了一台中转服务器作为国内服务器翻墙的代理服务器。 所以这边直接用 shadowsocks 配置，然后将代理指向那台中转服务器即可。
<!--more-->

## 安装 shadowsocks 客户端
其实shadowsocks安装时是不分客户端还是服务器端的，只不过安装后有两个脚本一个是 sslocal 代表以客户端模式工作，一个是 ssserver 代表以服务器端模式工作。
```html
yum install epel-release -y
yum install python-pip
pip install --upgrade pip
pip install shadowsocks
```
```html
[root@VM_156_200_centos ~]# pip install shadowsocks
Collecting shadowsocks
  Downloading https://files.pythonhosted.org/packages/02/1e/e3a5135255d06813aca6631da31768d44f63692480af3a1621818008eb4a/shadowsocks-2.8.2.tar.gz
Installing collected packages: shadowsocks
  Running setup.py install for shadowsocks ... done
Successfully installed shadowsocks-2.8.2
```
先安装 Python 的pip，然后安装 shadowsocks。 接下来新建一个 **sslocal.json** 文件， 在 **/etc/shadowsocks/** 目录， 这个目录也是要新建的
```html
[root@VM_156_200_centos etc]# mkdir shadowsocks
[root@VM_156_200_centos etc]# cd shadowsocks/
[root@VM_156_200_centos shadowsocks]# touch sslocal.json
```
然后将 代理服务器的配置写进去：
```html
[root@VM_156_200_centos ~]# cat /etc/shadowsocks/sslocal.json
{
  "server":"119.xx.xx.79",
  "server_port":1927,
  "password":"xxxx",
  "method":"rc4-md5"
}
```
当然，这个配置还有更加详细的文档说明, 可供参考：
```html
{
    "server":"your_server_ip",
    #ss服务器IP
    
    "server_port":your_server_port,
    #端口
    
    "local_address": "127.0.0.1",
    #本地ip
    
    "local_port":1080,
    #本地端口
    
    "password":"your_server_passwd",
    #连接ss密码
    
    "timeout":300,
    #等待超时
    
    "method":"rc4-md5",
    #加密方式
    
    "fast_open": false,
    # true 或 false。如果你的服务器 Linux 内核在3.7+，可以开启 fast_open 以降低延迟。开启方法： echo 3 > /proc/sys/net/ipv4/tcp_fastopen 开启之后，将 fast_open 的配置设置为 true 即可
    
    "workers": 1
    # 工作线程数
}
```
## 启动 shadowsocks 客户端
接下来就将这个客户端启动起来：
```html
[root@VM_156_200_centos shadowsocks]# sslocal -c /etc/shadowsocks/sslocal.json start
INFO: loading config from /etc/shadowsocks/sslocal.json
2019-02-12 18:31:09 INFO     loading libcrypto from libcrypto.so.10
2019-02-12 18:31:09 INFO     starting local at 127.0.0.1:1080
```

使用的是 **sslocal** 这个命令，表示 shadowsocks 以客户端模式工作，这样就代表启动成功了。但是这样子是在前台运行的，如果要在后台运行，那么就要这样子：
```html
[root@VM_156_200_centos ~]# sslocal -c /etc/shadowsocks/sslocal.json -d start
INFO: loading config from /etc/shadowsocks/sslocal.json
2019-02-12 18:33:10 INFO     loading libcrypto from libcrypto.so.10
started
```
这样子 shadowsocks 客户端就在后台运行了

## 安装 Privoxy
安好了 shadowsocks 后，但它是 socks5 代理，我们在shell里执行的命令，发起的网络请求现在还不支持socks5代理，只支持http／https代理。为了我门需要安装 privoxy 代理，它能把电脑上所有http请求转发给 shadowsocks 。
访问官网 http://www.privoxy.org/ 获得 Privoxy 的最新源码: privoxy-3.0.28-stable-src.tar.gz , 执行 tar 进行解压：
```html
[root@VM_156_200_centos ~]# tar -vxf privoxy-3.0.28-stable-src.tar.gz
```
然后进入 privoxy-3.0.24-stable 进入。
```html
[root@VM_156_200_centos ~]# cd privoxy-3.0.28-stable/
```
安装前需要执行 **useradd privoxy** 创建一个用户 privoxy，
```html
[root@VM_156_200_centos privoxy-3.0.28-stable]# useradd privoxy
```

然后依次执行如下三条命令:
```html
autoheader && autoconf
./configure
make && make install
```
但是刚开始的时候报错：
```html
[root@VM_156_200_centos privoxy-3.0.28-stable]# autoheader && autoconf
-bash: autoheader: command not found
[root@VM_156_200_centos privoxy-3.0.28-stable]# ./configure
-bash: ./configure: No such file or directory
[root@VM_156_200_centos privoxy-3.0.28-stable]# make && make install
***
*** To build this program, you must run
*** autoheader && autoconf && ./configure and then run GNU make.
***
*** Shall I do this for you now? (y/n) y
/bin/sh: line 26: autoheader: command not found
make: *** [error] Error 1
```
看了一下报的错误，要先安装 **autoheader** 才行：
```html
[root@VM_156_200_centos privoxy-3.0.28-stable]# yum install autoconf
```
安装完成之后，再重新执行一下，就可以了：
```html
[root@VM_156_200_centos privoxy-3.0.28-stable]# autoheader && autoconf
。。。
autoheader: WARNING: More sophisticated templates can also be produced, see the
autoheader: WARNING: documentation.

[root@VM_156_200_centos privoxy-3.0.28-stable]# ./configure
checking build system type... x86_64-unknown-linux-gnu
checking host system type... x86_64-unknown-linux-gnu
checking for gcc... gcc
。。。
config.status: creating config.h

[root@VM_156_200_centos privoxy-3.0.28-stable]# make && make install
gcc -c -pipe -O2   -pthread -Wall -Ipcre  actions.c -o actions.o
。。。
rm -f config.base config.tmp
Privoxy 3.0.28 stable installation succeeded!
The Privoxy configuration files have been installed in /usr/local/etc/privoxy
```
这样就安装完了，而且配置文件就在 **/usr/local/etc/privoxy** 这个目录了,接下来进入到这个目录，修改配置文件，但是改之前，先备份一下：
```html
[root@VM_156_200_centos ~]# cd /usr/local/etc/privoxy/
[root@VM_156_200_centos privoxy]# cp config config.bak
```
先搜索关键字 **listen-address** 找到 **listen-address 127.0.0.1:8118** 这一句，保证这一句没有注释，8118就是将来http代理要输入的端口。然后搜索**forward-socks5t**,将 **forward-socks5t / 127.0.0.1:1080 .**此句的注释去掉.
```html
[root@VM_156_200_centos privoxy]# cat config | grep '127.0.0.1:8118'
      127.0.0.1:8118
listen-address  127.0.0.1:8118
[root@VM_156_200_centos privoxy]# cat config | grep '127.0.0.1:1080'
        forward-socks5t   /               127.0.0.1:1080 .
```
可以看到修改成功，并且把注释去掉了。 最后启动 privoxy：
```html
[root@VM_156_200_centos privoxy]# service privoxy start
Starting Privoxy, OK.
```

## 配置/etc/profile
接下来就要在全局的配置文件里面添加代理设置, 执行vim /etc/profile,添加如下三句:
```html
export http_proxy=http://127.0.0.1:8118
export https_proxy=http://127.0.0.1:8118
export ftp_proxy=http://127.0.0.1:8118
```
```html
[root@VM_156_200_centos privoxy]# cat /etc/profile | grep proxy
# ss http proxy
export http_proxy=http://127.0.0.1:8118
export https_proxy=http://127.0.0.1:8118
export ftp_proxy=http://127.0.0.1:8118
```
第三句ftp的代理根据需要，不需要的话可以不添加, 然后执行 source 使得配置文件的修改生效：
```html
[root@VM_156_200_centos privoxy]# source /etc/profile
```
这样子全局代理就配置完了。
## 测试代理是否生效
我们可以直接这样弄：
```html
[root@VM_156_200_centos privoxy]# curl www.google.com
<!doctype html><html itemscope="" itemtype="http://schema.org/WebPage" lang="zh-HK"><head><meta content="text/html; charset=UTF-8" http-equiv="Content-Type"><meta content="/images/branding/googleg/1x/googleg_standard_color_128dp.png" itemprop="image"><title>Google</title><script nonce="90K5xN3lpHkZ8BSCabliSw=
```
这样就说明可以访问 google 的页面，一般来说就是翻墙成功了。当然更准确一点，就是判断当前代理出去的 ip 是不是就是香港那一台的中转服务器， 访问 cip.cc 就可以知道了
```html
[root@VM_156_200_centos privoxy]# curl  cip.cc
IP    : 119.xx.xx.79
地址    : 中国  香港  tencent.com

数据二    : 新加坡 | 腾讯云

数据三    : 中国香港香港 | 腾讯

URL    : http://www.cip.cc/119.xx.xx.79
```
可以看到出口ip，就是香港那一台的ip，说明有代理成功了。 如果是之前没有挂代理之前的话，那么就不会是香港的，而是国内广州的出口ip：
```html
[root@VM_156_200_centos ~]# curl  cip.cc
IP    : 119.xx.xx.28
地址    : 中国  广东  广州
运营商    : tencent.com

数据二    : 广东省广州市海珠区 | 深圳市腾讯计算机系统有限公司IDC机房(BGP)

数据三    : 中国广东省广州市 | 电信

URL    : http://www.cip.cc/119.xx.xx.28
```


## 关掉代理
那么我们要怎么关掉代理呢？
我们先把全局配置的代理注释掉，然后 source 生效。然后再关掉 privoxy 看看 （shadowsocks 不用关）：
```html
[root@VM_156_200_centos privoxy]# vim /etc/profile
[root@VM_156_200_centos privoxy]# cat /etc/profile | grep proxy
# ss http proxy
#export http_proxy=http://127.0.0.1:8118
#export https_proxy=http://127.0.0.1:8118
#export ftp_proxy=http://127.0.0.1:8118
[root@VM_156_200_centos privoxy]# source /etc/profile
[root@VM_156_200_centos privoxy]# service privoxy stop
Stopping Privoxy, OK.
[root@VM_156_200_centos privoxy]# curl  cip.cc
curl: (7) Failed connect to 127.0.0.1:8118; Connection refused
```
结果连 curl 都出问题了。 这种情况其实就是全局代理还在生效？？
这时候我执行了 **reboot** 指令，重新启动一下系统看看。
```html
[root@VM_156_200_centos ~]# cat /etc/profile | grep proxy
# ss http proxy
#export http_proxy=http://127.0.0.1:8118
#export https_proxy=http://127.0.0.1:8118
#export ftp_proxy=http://127.0.0.1:8118
[root@VM_156_200_centos ~]# curl  cip.cc
IP    : 119.xx.xx.28
地址    : 中国  广东  广州
运营商    : tencent.com

数据二    : 广东省广州市海珠区 | 深圳市腾讯计算机系统有限公司IDC机房(BGP)

数据三    : 中国广东省广州市 | 电信

URL    : http://www.cip.cc/119.xx.xx.28
```
这时候就生效了，就不代理了。 那我们再设置为全局代理。一样就是去掉全局配置的注释，然后启动 privoxy， 然后再启动 shadowsocks (如果没有开的话)。
```html
[root@VM_156_200_centos ~]# vim /etc/profile
[root@VM_156_200_centos ~]# cat /etc/profile | grep proxy
# ss http proxy
export http_proxy=http://127.0.0.1:8118
export https_proxy=http://127.0.0.1:8118
export ftp_proxy=http://127.0.0.1:8118
[root@VM_156_200_centos ~]# source /etc/profile
[root@VM_156_200_centos ~]# service privoxy start
Starting Privoxy, OK.
[root@VM_156_200_centos ~]# sslocal -c /etc/shadowsocks/sslocal.json -d start
INFO: loading config from /etc/shadowsocks/sslocal.json
2019-02-13 14:02:55 INFO     loading libcrypto from libcrypto.so.10
started
[root@VM_156_200_centos ~]# curl  cip.cc
IP    : 119.xx.xx.79
地址    : 中国  香港  tencent.com

数据二    : 新加坡 | 腾讯云

数据三    : 中国香港香港 | 腾讯

URL    : http://www.cip.cc/119.xx.xx.79
```
这时候就代理成功了。然后这时候我把它关掉，还是注释全局配置，privoxy 和 shadowsocks 都不要关
```html
[root@VM_156_200_centos ~]# vim /etc/profile
[root@VM_156_200_centos ~]# cat /etc/profile | grep proxy
# ss http proxy
#export http_proxy=http://127.0.0.1:8118
#export https_proxy=http://127.0.0.1:8118
#export ftp_proxy=http://127.0.0.1:8118
[root@VM_156_200_centos ~]# source /etc/profile
[root@VM_156_200_centos ~]# curl cip.cc
IP    : 119.xx.xx.79
地址    : 中国  香港  tencent.com

数据二    : 新加坡 | 腾讯云

数据三    : 中国香港香港 | 腾讯

URL    : http://www.cip.cc/119.xx.xx.79
```
这时候从这个 terminal 看还是 代理的情况???? 这么神奇？？ 但是我重开一个新的 terminal ，这时候就是没有代理的情况
```html
[root@VM_156_200_centos ~]# curl cip.cc
IP    : 119.xx.xx.28
地址    : 中国  广东  广州
运营商    : tencent.com

数据二    : 广东省广州市海珠区 | 深圳市腾讯计算机系统有限公司IDC机房(BGP)

数据三    : 中国广东省广州市 | 电信

URL    : http://www.cip.cc/119.xx.xx.28
```
这时候就正常了，这时候如果要重开代理的话，直接修改全局配置就行了 （privoxy 和 shadowsocks 其实都开着没事）
```html
[root@VM_156_200_centos ~]# vim /etc/profile
[root@VM_156_200_centos ~]# cat /etc/profile | grep proxy
# ss http proxy
export http_proxy=http://127.0.0.1:8118
export https_proxy=http://127.0.0.1:8118
export ftp_proxy=http://127.0.0.1:8118
[root@VM_156_200_centos ~]# source /etc/profile
[root@VM_156_200_centos ~]# curl cip.cc
IP    : 119.xx.xx.79
地址    : 中国  香港  tencent.com

数据二    : 新加坡 | 腾讯云

数据三    : 中国香港香港 | 腾讯

URL    : http://www.cip.cc/119.xx.xx.79
```
这样就开成功了。 所以如果要关掉代理也很简单，直接将配置文件的 proxy 注释掉，source 生效一下，然后重开一个新的 terminal 就行了， privoxy 和 shadowsocks 都不要关。

---

参考资料：  [linux 配置shadowsocks代理全局代理](https://www.jianshu.com/p/41378f4e14bc)