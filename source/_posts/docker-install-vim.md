---
title: docker 容器安装 vim
date: 2020-06-19 15:09:37
tags: docker
categories: docker相关
---
## 前言
有时候在使用 docker 容器的时候， 容器会默认没有安装 vim， 比如前段时间我在测试 kerento 的时候，要在容器中修改某一个配置文件，然后再重启，这时候在修改的时候，就会提示:
```php
root@f596838cdaaf:/etc/kurento/modules/kurento# vim WebRtcEndpoint.conf.ini 
bash: vim: command not found
```
这时候就会提示 命令不存在。 这个是因为在创建 docker 镜像的时候，并没有把 vim 也放进去， 其实这个是非常常见，因为很多的 docker image 的创建都是基于那个最白版的 Ubuntu 系统，那个里面就没有 vim 命令，只有一些最基础的，比如 `ls, cat` 之类的，我们可以去 /bin/ 查看一下，当前所支持的 shell 指令:
<!--more-->
```php
root@f596838cdaaf:/bin# ls
bash     bzfgrep       chgrp  df             false    hostname    lsblk       mv             readlink    sleep                 systemd-escape                  tar         vdir          zegrep
bunzip2  bzgrep        chmod  dir            fgrep    journalctl  mkdir       networkctl     rm          stty                  systemd-inhibit                 tempfile    wdctl         zfgrep
bzcat    bzip2         chown  dmesg          findmnt  kill        mknod       nisdomainname  rmdir       su                    systemd-machine-id-setup        touch       which         zforce
bzcmp    bzip2recover  cp     dnsdomainname  grep     ln          mktemp      pidof          run-parts   sync                  systemd-notify                  true        ypdomainname  zgrep
bzdiff   bzless        dash   domainname     gunzip   login       more        ps             sed         systemctl             systemd-tmpfiles                umount      zcat          zless
bzegrep  bzmore        date   echo           gzexe    loginctl    mount       pwd            sh          systemd               systemd-tty-ask-password-agent  uname       zcmp          zmore
bzexe    cat           dd     egrep          gzip     ls          mountpoint  rbash          sh.distrib  systemd-ask-password  tailf                           uncompress  zdiff         znew
```
可以看到里面就是没有 vim 指令。所以我们要自己装。

## 安装 vim
当然这时候如果直接用 `apt-get install vim` 安装的话，会报错:
```php
root@f596838cdaaf:/etc/kurento/modules/kurento# apt-get install vim
Reading package lists... Done
Building dependency tree       
Reading state information... Done
E: Unable to locate package vim
```
这时候需要敲：`apt-get update`，这个命令的作用是：同步 `/etc/apt/sources.list` 和 `/etc/apt/sources.list.d` 中列出的源的索引，这样才能获取到最新的软件包:
```text
root@f596838cdaaf:/etc/kurento/modules/kurento# apt-get update
Get:1 http://security.ubuntu.com/ubuntu xenial-security InRelease [109 kB]
...
Fetched 22.8 MB in 7min 22s (51.5 kB/s)
                                                       
Reading package lists... Done
```
等更新完毕以后再敲命令：`apt-get install -y vim` 命令即可。
```php
root@f596838cdaaf:/etc/kurento/modules/kurento# apt-get install -y vim
Reading package lists... Done
Building dependency tree       
...
update-alternatives: using /usr/bin/vim.basic to provide /usr/bin/editor (editor) in auto mode
Processing triggers for libc-bin (2.23-0ubuntu11) ...
```
这样子就安装成功了。

## 配置国内镜像源
实际在使用过程中，运行 `apt-get update`，然后执行 `apt-get install -y vim`，下载地址由于是海外地址，下载速度异常慢而且可能中断更新流程，所以做下面配置：
```php
mv /etc/apt/sources.list /etc/apt/sources.list.bak && \
    echo "deb http://mirrors.163.com/debian/ jessie main non-free contrib" >/etc/apt/sources.list && \
    echo "deb http://mirrors.163.com/debian/ jessie-proposed-updates main non-free contrib" >>/etc/apt/sources.list && \
    echo "deb-src http://mirrors.163.com/debian/ jessie main non-free contrib" >>/etc/apt/sources.list && \
    echo "deb-src http://mirrors.163.com/debian/ jessie-proposed-updates main non-free contrib" >>/etc/apt/sources.list

#然后再更新安装源
apt-get update 
```
这样就可以了。

