---
title: docker 升级到最新版
date: 2019-02-22 13:57:09
tags: docker
categories: docker相关
---
## 前言
之前在 {% post_link docker-container-proxy %} 需要将 docker 升级到最新版本，所以这边弄一下，我是 CentOS 7 系统
## 卸掉旧的
首先找出主机上所有 docker 的包，然后卸掉：
```html
[root@VM_156_200_centos ~]# rpm -qa | grep docker
docker-common-1.13.1-91.git07f3374.el7.centos.x86_64
docker-1.13.1-91.git07f3374.el7.centos.x86_64
docker-client-1.13.1-91.git07f3374.el7.centos.x86_64
```
然后分别卸掉,用 remove 指令：
```html
[root@VM_156_200_centos ~]# yum remove docker-common-1.13.1-91.git07f3374.el7.centos.x86_64
[root@VM_156_200_centos ~]# yum remove docker-1.13.1-91.git07f3374.el7.centos.x86_64
[root@VM_156_200_centos ~]# yum remove docker-client-1.13.1-91.git07f3374.el7.centos.x86_64
```
<!--more-->
这时候就会提示找不到 docker 了：
```html
[root@VM_156_200_centos ~]# docker
-bash: /usr/bin/docker: No such file or directory
```
## 升级到最新
接下来使用 curl 升级到最新版
```html
[root@VM_156_200_centos ~]# curl -fsSL https://get.docker.com/ | sh
```
## 重启
然后重启docker：
```html
[root@VM_156_200_centos ~]# systemctl restart docker
```
## 设置开机自启
设置docker开机自启：
```html
[root@VM_156_200_centos ~]# systemctl enable docker
Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.
```
## 验证
查看Docker版本信息：
```html
[root@VM_156_200_centos ~]# docker version
Client:
Version:           18.09.2
```
这时候就是 18.09， 就是最新版本了。
重新执行一下指令，容器和镜像还在，不过容器都是关闭的，要重新 start 才行：
```html
[root@VM_156_200_centos ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                           PORTS                                                                        NAMES
6b68c09cf1ff        kbz/golang-1.10-2   "bash"                   About an hour ago   Exited (0) About an hour 

。。。


[root@VM_156_200_centos ~]# docker images
REPOSITORY                  TAG                 IMAGE ID            CREATED             SIZE
kbz/golang-1.10-2           latest              24e1a3abcfb2        47 hours ago        760MB
kbz/golang-1.10             latest              61b609c1428e        3 days ago          760MB
scratch                     latest              92e88945fd07        7 days ago          0B
golang-scratch-demo-app-2   latest              a7d182445c8d        7 days ago          4.72MB
```







