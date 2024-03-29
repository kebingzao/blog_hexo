---
title: CentOS 7 安装 Nginx
date: 2020-09-15 15:51:56
tags: nginx
categories: nginx相关
---
## 前言
首先安装 nginx 的教程，网上到处都是，其实真没有为这个写 blog 的必要，当然之所以还是要写的原因无非是当初自己在安装的时候，虽然照着教程安装，但是还是有遇到了一些神奇的情况，当然后面都解决了，不过还是记一下比较好，后面自己要安装的时候，省的再去找网上的文章，自己看自己的 blog 它不香吗 ↖(^ω^)↗

安装 nginx 有两种:
1. 直接系统安装，本文主要讲这种
2. 使用容器安装，比如 docker

## 使用 docker 安装
这种方式很简单，直接执行一条指令就可以了:
```text
docker run -d -p 80:80 --name webserver nginx
```

这样子就可以了，其中 `-p` 是端口映射， `-d` 就是以守护态运行
<!--more-->
## 直接系统安装
Nginx 是 C语言 开发，建议在 Linux 上运行，当然，也可以安装 Windows 版本，本文则使用 CentOS 7 作为安装环境。
这边有涉及到几个依赖安装，按照下面顺序来
### 1. gcc 安装
安装 nginx 需要先将官网下载的源码进行编译，编译依赖 gcc 环境，如果没有 gcc 环境，则需要安装：

```text
yum install gcc-c++
```

### 2. PCRE pcre-devel 安装
PCRE(Perl Compatible Regular Expressions) 是一个Perl库，包括 perl 兼容的正则表达式库。nginx 的 http 模块使用 pcre 来解析正则表达式，所以需要在 linux 上安装 pcre 库，pcre-devel 是使用 pcre 开发的一个二次开发库。nginx也需要此库。

```text
yum install -y pcre pcre-devel
```

### 3. zlib 安装
zlib 库提供了很多种压缩和解压缩的方式， nginx 使用 zlib 对 http 包的内容进行 gzip ，所以需要在 Centos 上安装 zlib 库。

```text
yum install -y zlib zlib-devel
```

### 4. OpenSSL 安装
OpenSSL 是一个强大的安全套接字层密码库，囊括主要的密码算法、常用的密钥和证书封装管理功能及 SSL 协议，并提供丰富的应用程序供测试或其它目的使用。nginx 不仅支持 http 协议，还支持 https（即在ssl协议上传输http），所以需要在 CentOS 安装 OpenSSL 库。

```text
yum install -y openssl openssl-devel
```

### 5. 官网下载
直接下载 `.tar.gz` 安装包，地址：https://nginx.org/en/download.html

使用 wget 命令下载（推荐）。确保系统已经安装了wget，如果没有安装，执行 yum install wget 安装。

```text
wget -c https://nginx.org/download/nginx-1.18.0.tar.gz
```

### 6. 解压
```text
tar -zxvf nginx-1.18.0.tar.gz

cd nginx-1.18.0
```

### 7. 配置
直接使用默认配置就够了, 当然也可以自定义配置，不过我觉得没啥必要
```text
./configure
```

### 8. 编译安装
```text
make

make install
```

### 9. 启动，停止 nginx
先得到 nginx 所在的目录，然后启动，最后查看 80 端口是否有开启
```text
[root@VM_156_200_centos ~]# whereis nginx
nginx: /usr/local/nginx
[root@VM_156_200_centos nginx-1.18.0]# cd /usr/local/nginx/sbin/
[root@VM_156_200_centos sbin]# ./nginx
[root@VM_156_200_centos sbin]# netstat -anlp | grep 80
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      920/nginx: master p
```
可以看到 80 端口开起来了， curl 一下, 发现已经启动了
```text
[root@VM_156_200_centos ~]# curl 127.0.0.1
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```
还有其他几种操作方式，包括关闭和重载:
```text
./nginx -s stop
./nginx -s quit
./nginx -s reload
```
也可以通过查看进程，判断 nginx 的进程是否存在:
```text
[root@VM_156_200_centos ~]# ps aux|grep nginx
root       920  0.0  0.0  20560   612 ?        Ss   17:19   0:00 nginx: master process ./nginx
nobody     921  0.0  0.0  23080  1632 ?        S    17:19   0:00 nginx: worker process
root      7701  0.0  0.0 112708   980 pts/1    R+   17:35   0:00 grep --color=auto nginx
root     29977  0.0  0.0    192     4 ?        S    Jul09   0:00 s6-supervise nginx
root     29981  0.0  0.3 279284  6392 ?        Ss   Jul09   0:00 nginx: master process nginx -c /config/nginx/nginx.conf
33       30078  0.0  0.3 280300  6444 ?        S    Jul09   3:52 nginx: worker process
...
```

### 10. 重启 nginx 和 重载
1. 先停止再启动（推荐）：
对 nginx 进行重启相当于先停止再启动，即先执行停止命令再执行启动命令。如下：
```text
./nginx -s quit
./nginx
```

2. 重新加载配置文件：
当 nginx 的配置文件 nginx.conf 修改后，要想让配置生效需要重启 nginx，使用`-s reload` 不用先停止 nginx 再启动 nginx 即可将配置信息在 nginx 中生效，如下：
```text
./nginx -s reload
```

### 13. 添加到系统服务并且设置开机自启动
经过编译安装以及解决问题，Nginx 已经运行正常，但是此时 Nginx 并没有添加进系统服务。接下来会将 Nginx 添加进系统服务和设置开机自启动。

首先查看是否有添加:
```text
[root@VM_156_200_centos sbin]# systemctl status nginx
Unit nginx.service could not be found.
```
没有找到相关的服务，下一步就是添加系统服务。 接下来就是在 `/usr/lib/systemd/system` 目录中添加 `nginx.service` 这个文件:
```text
[root@VM_156_200_centos system]# touch nginx.service
[root@VM_156_200_centos system]# vi nginx.service
[root@VM_156_200_centos system]# cat nginx.service
[Unit]
Description=nginx - high performance web server
Documentation=http://nginx.org/en/docs/
After=network.target remote-fs.target nss-lookup.target
  
[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStartPre=/usr/local/nginx/sbin/nginx -t -c /usr/local/nginx/conf/nginx.conf
ExecStart=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true
  
[Install]
WantedBy=multi-user.target
```
简单分析一下，上面的文件是啥意思:
1. **Unit** 
  1. Description : 服务的简单描述
  2. Documentation : 服务文档
  3. After= : 依赖，仅当依赖的服务启动之后再启动自定义的服务单元

2. **Service**
  1. Type : 启动类型，一般守护进程都是用 forking
  2. PIDFile : pid文件路径
  3. ExecStartPre : 启动前要做什么，上文中是测试配置文件 －t 
  4. ExecStart : 启动
  5. ExecReload : 重载
  6. ExecStop : 停止
  7. PrivateTmp : True表示给服务分配独立的临时空间
  
3. **Install**
  1. WantedBy : 服务安装的用户模式, 如果是 `multi-user.target` 表明当系统以多用户方式（默认的运行级别）启动时，这个服务需要被自动运行。当然还需要 `systemctl enable` 激活这个服务以后自动运行才会生效。
  
接下来将其设置为开机自启动:
```text
systemctl enable nginx.service
```
这边需要注意一点的是，因为我们是 CentOS 7, 所以我们可以直接用 `systemctl enable` 来进行自启动设置。如果是 CentOS 6 的版本，那么就要将启动项写入 `rc.local` 才行。

接下来查看状态并启动一下:
```text
[root@VM_156_200_centos system]# sudo systemctl status nginx
● nginx.service - nginx - high performance web server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: inactive (dead)
     Docs: http://nginx.org/en/docs/
[root@VM_156_200_centos system]# sudo systemctl start nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```
发现启动失败了???, 然后用原始的指令试一下，发现也不行, 而且也没法关闭了
```text
[root@VM_156_200_centos sbin]# ./nginx -s stop
nginx: [error] open() "/usr/local/nginx/logs/nginx.pid" failed (2: No such file or directory)
```
应该是进程的问题， 只能将 nginx 进程重新杀掉，再重启了, 找到进程所在的 PID， 然后杀掉
```text
[root@VM_156_200_centos logs]# ps aux | grep nginx
root      8464  0.0  0.0  20696  1384 ?        Ss   17:37   0:00 nginx: master process ./nginx
...
```
```text
[root@VM_156_200_centos logs]# kill -QUIT 8464
```
```text
[root@VM_156_200_centos logs]# ps aux | grep nginx
root     26044  0.0  0.0 112708   976 pts/1    R+   18:19   0:00 grep --color=auto nginx
root     29977  0.0  0.0    192     4 ?        S    Jul09   0:00 s6-supervise nginx
root     29981  0.0  0.3 279284  6392 ?        Ss   Jul09   0:00 nginx: master process nginx -c /config/nginx/nginx.conf
...
```
```text
[root@VM_156_200_centos logs]# kill -QUIT 29981
```
注意，这边有可能要杀好几次，才能杀干净，所以可以通过这个查看
```text
[root@VM_156_200_centos ~]# fuser -n tcp 80
```
最后再重启 nginx:
```text
[root@VM_156_200_centos sbin]# ./nginx
```
这样子就可以了，而且也可以正常关闭了:
```text
[root@VM_156_200_centos sbin]# ./nginx -s quit
[root@VM_156_200_centos sbin]# curl 127.0.0.1
curl: (7) Failed connect to 127.0.0.1:80; Connection refused
```
接下来继续尝试添加到系统服务, 注意，如果 nginx.service 配置有变化，要先执行这个
```text
systemctl daemon-reload
```
```text
[root@VM_156_200_centos system]# sudo systemctl enable nginx.service
[root@VM_156_200_centos system]# sudo systemctl start nginx.service
```
这样子就成功了， 可以查看日志:
```text
[root@VM_156_200_centos system]#  sudo journalctl -f -u nginx.service
-- Logs begin at Tue 2020-06-30 09:16:54 CST. --
Sep 10 18:07:14 VM_156_200_centos systemd[1]: nginx.service: control process exited, code=exited status=1
```
最后可以重启或者重载配置文件，都可以:
```text
[root@VM_156_200_centos system]# sudo systemctl restart nginx.service
[root@VM_156_200_centos system]# sudo systemctl reload nginx.service
```
这样子 nginx 就安装好了，接下来就是修改配置文件并测试一些其他的请求。

### 14. 修改配置文件并测试
这边简单一点， 默认的配置文件是 `/usr/local/nginx/conf/nginx.conf`, 在这个文件的下面添加一行:
```text
location = /hello {
    return 200 'hello zach';
}
```
然后重启并刷新一下配置:
```text
[root@VM_156_200_centos conf]# sudo systemctl reload nginx.service
[root@VM_156_200_centos conf]# curl 127.0.0.1/hello
hello zach
```
可以看到已经生效了。

## 重载问题
不过在 CentOS 7 下，如果用 `systemctl reload nginx.service` 来重载的话，这时候会有问题，因为它并不会校验 nginx.conf 文件。 也就是说如果 nginx.conf 文件的配置没有问题，那么没有差， 但是一旦有问题， 用这个指令的话，就会重载失败， 更坑的是， 他并没有报失败的错，其表现跟正常成功一样。

这个坑可以解决的，具体可以看 {% post_link centos7-systemctl-reload-nginx %}

## 配置 tls 的问题
虽然 nginx 装完了， 但是我发现在配置 https 的时候, 运行会被报错:
```text
 # HTTPS server
    server {
        listen       443 ssl;
        server_name  localhost;

        ssl_certificate      ssl/server.crt;
        ssl_certificate_key  ssl/server.key;

        ssl_session_cache    shared:SSL:1m;
        ssl_session_timeout  5m;

        ssl_ciphers  HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers  on;

        location / {
            root   html;
            index  index.html index.htm;
        }
    }
```
这时候 reload 就会报错:
```text
[root@VM-0-13-centos conf]# sudo systemctl reload nginx.service
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@VM-0-13-centos conf]# systemctl status nginx.service
● nginx.service - nginx - high performance web server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: active (running) (Result: exit-code) since Thu 2021-10-21 16:30:39 CST; 43min ago
     Docs: http://nginx.org/en/docs/
  Process: 31476 ExecReload=/usr/local/nginx/sbin/nginx -s reload (code=exited, status=1/FAILURE)
Main PID: 13905 (nginx)
    Tasks: 2
   Memory: 1.0M
   CGroup: /system.slice/nginx.service
           ├─13905 nginx: master process /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
           └─16998 nginx: worker process

Oct 21 16:30:39 VM-0-13-centos nginx[13900]: nginx: configuration file /usr/local/nginx/conf/nginx.conf test is successful
Oct 21 16:30:39 VM-0-13-centos systemd[1]: Started nginx - high performance web server.
Oct 21 16:33:24 VM-0-13-centos systemd[1]: Reloading nginx - high performance web server.
Oct 21 16:33:24 VM-0-13-centos systemd[1]: Reloaded nginx - high performance web server.
Oct 21 16:38:02 VM-0-13-centos systemd[1]: Reloading nginx - high performance web server.
Oct 21 16:38:02 VM-0-13-centos systemd[1]: Reloaded nginx - high performance web server.
Oct 21 17:14:20 VM-0-13-centos systemd[1]: Reloading nginx - high performance web server.
Oct 21 17:14:20 VM-0-13-centos nginx[31476]: nginx: [emerg] the "ssl" parameter requires ngx_http_ssl_module in /usr/local/nginx/conf/nginx.conf:42
Oct 21 17:14:20 VM-0-13-centos systemd[1]: nginx.service: control process exited, code=exited status=1
Oct 21 17:14:20 VM-0-13-centos systemd[1]: Reload failed for nginx - high performance web server.
```
原来如果要配置 ssl 模块的时候， 是要开启 `ngx_http_ssl_module` 模块的，我看了一下当前的配置:

```text
[root@VM-0-13-centos conf]# /usr/local/nginx/sbin/nginx -V
nginx version: nginx/1.18.0
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC)
configure arguments:
```

发现是没有额外配置这个模块的， 因为这个模块不是系统默认的内建模块， 参见上面安装的 `./configure` 步骤，并没有加这个模块。 这时候有两种方式可以添加
1. 一种是直接卸载后再重装， 然后在 `./configure` 步骤，换成这个 `./configure --with-http_ssl_module` 这样子就会将这个模块编译进去
2. 还有一种就是不需要卸载， 直接在原来的 nginx 补上这个模块

### 现有的 nginx 程序添加 http_ssl_module 模块
主要有几个步骤:
#### 1. 首先切换到源码包目录
```text
[root@VM-0-13-centos ~]# cd nginx-1.18.0/
```

#### 2. configure 配置加上这个模块
```text
[root@VM-0-13-centos nginx-1.18.0]# ./configure --prefix=/usr/local/nginx  --with-http_ssl_module
checking for OS
+ Linux 3.10.0-1160.31.1.el7.x86_64 x86_64
checking for C compiler ... found
```
这样子配置就完成了， 就可以 make 编译了

#### 3. make 编译
```text
[root@VM-0-13-centos nginx-1.18.0]# make
make -f objs/Makefile
make[1]: Entering directory `/root/nginx-1.18.0'
cc -c -pipe  -O -W -Wall -Wpointer-arith -Wno-unused-parameter -Werror -g  -I src/core -I src/event -I src/event/modules -I src/os/unix -I objs \
    -o objs/src/core/nginx.o \
```

这里接下来不要进行 `make install`，否则就是覆盖安装,原来的 nginx 还有一堆的配置文件，不能被覆盖。我们应该只覆盖编译出来的 nginx 可执行程序：

#### 4. 停掉原来的 nginx 并且覆盖原来的 nginx 可执行程序
```text
[root@VM-0-13-centos nginx-1.18.0]# sudo systemctl stop nginx.service
[root@VM-0-13-centos nginx-1.18.0]# curl 127.0.0.1
curl: (7) Failed connect to 127.0.0.1:80; Connection refused
```
覆盖之前，先把原来的先 cp 一份出来
```text
[root@VM-0-13-centos nginx-1.18.0]# cp /usr/local/nginx/sbin/nginx /usr/local/nginx/sbin/nginx.bak
```
最后将新编译的 nginx 可执行程序覆盖原来的程序:
```text
[root@VM-0-13-centos nginx-1.18.0]# cp ./objs/nginx /usr/local/nginx/sbin/
```

#### 5. 重启 nginx
这样子就好了， 然后重新启动 nginx
```text
[root@VM-0-13-centos nginx-1.18.0]# sudo systemctl start  nginx.service
```
查看模块是否有开启
```text
[root@VM-0-13-centos nginx-1.18.0]# /usr/local/nginx/sbin/nginx -V
nginx version: nginx/1.18.0
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC)
built with OpenSSL 1.0.2k-fips  26 Jan 2017
TLS SNI support enabled
configure arguments: --prefix=/usr/local/nginx --with-http_ssl_module
```
测试一下 https 有没有问题:
```text
[root@VM-0-13-centos nginx-1.18.0]# curl https://127.0.0.1
curl: (60) Peer's certificate issuer has been marked as not trusted by the user.
More details here: http://curl.haxx.se/docs/sslcerts.html

curl performs SSL certificate verification by default, using a "bundle"
of Certificate Authority (CA) public keys (CA certs). If the default
bundle file isn't adequate, you can specify an alternate file
using the --cacert option.
If this HTTPS server uses a certificate signed by a CA represented in
the bundle, the certificate verification probably failed due to a
problem with the certificate (it might be expired, or the name might
not match the domain name in the URL).
If you'd like to turn off curl's verification of the certificate, use
the -k (or --insecure) option.
```
报了一个证书的安全问题, 因为证书是自制的， 这个是正常的，就好像 浏览器上访问的不安全提示一下。 我们加 `--insecure` 参数就可以忽略到这个参数
```text
[root@VM-0-13-centos nginx-1.18.0]# curl https://127.0.0.1 --insecure
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

---

参考资料:
- [CentOS 7 systemd添加自定义系统服务](https://my.oschina.net/liucao/blog/470458)
- [CentOS 7 下安装 Nginx](https://www.cnblogs.com/liujuncm5/p/6713784.html)
- [开启 Nginx 的SSL模块](https://www.cnblogs.com/ghjbk/p/6744131.html)
  