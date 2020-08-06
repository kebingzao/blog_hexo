---
title: linux 使用 alias 来简化 docker 命令
date: 2020-08-06 11:27:50
tags: linux
categories: 
- 系统相关
- docker 相关
---
## 前言
自从 用 docker 来部署 mqtt vernemq 之后，后面的指令也会变得很长，比如，如果要进入容器的目录，就要这样子：
```text
sudo docker exec -it vernemq-release-cn-1 /bin/bash
```
每次都要输入这么一大串，很麻烦，所以后面就考虑用添加别名的方式来弄。

## 编写 alias 命令
Linux操作系统中打开一些应用，有时需要进入对应的文件夹，打开对应的程序，不是很方便。alias命令是一种命令别名命名法，可以将一些复杂的命令简化成一个我们自己命名的相对简单好记的命令。能够极其方便我们的操作。

## 实操过程
<!--more-->
这个只对这个账号，所以首先回到自己的个人目录：
```text
[kbz@VM_16_14_centos ~]$ cd
[kbz@VM_16_14_centos ~]$ ll
total 0
[kbz@VM_16_14_centos ~]$ ls -a
.  ..  .bash_history  .bash_logout  .bash_profile  .bashrc  .cache  .config  .ssh  .viminfo
```
编辑 `.bashrc` 这个文件，将这个命令加进去
```text
[kbz@VM_16_14_centos ~]$ sudo vim .bashrc
[kbz@VM_16_14_centos ~]$ cat .bashrc
# .bashrc
alias vmqpath='sudo docker exec -it vernemq-release-cn-1 /bin/bash'
# Source global definitions
if [ -f /etc/bashrc ]; then
    . /etc/bashrc
fi

# Uncomment the following line if you don't like systemctl's auto-paging feature:
# export SYSTEMD_PAGER=

# User specific aliases and functions
```
然后执行一下这个文件：
```text
[kbz@VM_16_14_centos ~]$ . ~/.bashrc
```
这样别名就创建好了，接下来直接输出 vmqpath 试试：
```text
[kbz@VM_16_14_centos ~]$ vmqpath
[root@16cb036546c8 /]# ll
total 18344
-rw-r--r--   1 root root    12030 Oct  6 19:15 anaconda-post.log
lrwxrwxrwx   1 root root        7 Oct  6 19:14 bin -> usr/bin
drwxr-xr-x   5 root root      360 Nov 17 14:35 dev
drwxr-xr-x   1 root root     4096 Nov 17 13:53 etc
```
可以看到命令执行了。 而且如果别名设置很多的话，那么就可以用这个指令先筛选：
```text
[kbz@VM_16_14_centos ~]$ alias | grep 'vmq'
alias vmqpath='sudo docker exec -it vernemq-release-cn-1 /bin/bash'
```






