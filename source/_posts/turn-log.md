---
title: coturn 输出log没有带日期格式的问题
date: 2018-11-28 11:37:18
tags: turn
categories: webrtc相关
---
## 前言
通过 {% post_link turn-install %} 我们知道自己搭的turn server 可以用于 webrtc 的 turn server转发了，但是后面发现在进行转发的过程中，输出的log，竟然没有带日期格式，也就是说，当我们在看log找问题的时候，根本找不到对应的时间点？ 这样可不利于bug排除？
<!--more-->
```html
[kbz@VM_16_13_centos turn]$ tail /var/log/supervisor/turnserver.info.log
5242696: session 003000000000010725: closed (2nd stage), user <1543455600:a97eeb03cd2512ee15906079a1c418_21_24_18154505> realm <pano> origin <>, local 172.16.16.13:3478, remote 125.77.202.250:52366, reason: allocation timeout
5242696: session 003000000000010725: delete: realm=<pano>, username=<1543455600:a97eeb03cd2512ee15906079a1c418_21_24_18154505>
5242696: session 003000000000010725: peer 120.41.148.188 deleted
```
## 解决
因为是直接配置输出到控制台，然后让supervisor 捕获的, coturn 的配置文件是这样配置的：
```html
log-file=stdout
```
所以通过 log-file 输出的 log 应该就是没有带上日期。 后面有尝试了一下输出到 syslog， 就是启用这个配置：
```html
syslog
```
这样虽然有日期，但是太杂乱了，因为这个是整个系统的，整个日志非常多，也很难排查。

既然 coturn 在输出的时候没有日期，那么我们的做法就是让 supervisor 捕获的时候，去加上日期格式。这边就涉及到一个工具包：[moreutils](https://joeyh.name/code/moreutils/)。 这个工具包的一个 ts 工具就是可以让输出的内容加上时间格式。
注意：服务器内置也有一个 ts 命令，但是这个不是我们要的，而是 openssl 的一个指令
```html
[kbz@VM_32_5_centos ~]$ man ts

TS(1)                                                                                                        OpenSSL                                                                                                       TS(1)

NAME
       openssl-ts, ts - Time Stamping Authority tool (client/server)
```
因为服务器是 centos7， 因此直接用 yum 安装即可：
```html
[root@VM_156_200_centos ~]# yum install moreutils
Loaded plugins: fastestmirror, langpacks
Repository epel is listed more than once in the configuration
epel                                                                                                                                                                                                           | 3.2 kB  00:00:00     
extras                                                                                                                                                                                                         | 3.4 kB  00:00:00     
os                                                                                                                                                                                                             | 3.6 kB  00:00:00     
updates                                                                                                                                                                                                        | 3.4 kB  00:00:00     
(1/2): epel/7/x86_64/updateinfo                                                                                                                                                                                | 933 kB  00:00:00     
(2/2): epel/7/x86_64/primary     
......
Dependency Installed:
  perl-IO-Tty.x86_64 0:1.10-11.el7                                          perl-IPC-Run.noarch 0:0.92-2.el7                                          perl-Time-Duration.noarch 0:1.06-17.el7                                         

Complete!
```
安装完之后，就有这个 ts 这个命令了:
```html
[kbz@VM_32_5_centos ~]$ man ts

TS(1)                                                                                                                                                                                                                      TS(1)

NAME
       ts - timestamp input
```
然后接下来试着输出：
```html
[root@VM_156_200_centos ~]# echo -e "foo\nbar\nbaz" | ts
Nov 28 11:24:08 foo
Nov 28 11:24:08 bar
Nov 28 11:24:08 baz
```
也可以带上具体的格式：
```html
[root@VM_156_200_centos ~]# echo -e "foo\nbar\nbaz" | ts '[%Y-%m-%d %H:%M:%S]'
[2018-11-28 11:24:17] foo
[2018-11-28 11:24:17] bar
[2018-11-28 11:24:17] baz
```
这样就会自动带上时间格式了。所以接下来只要改 supervisor 的命令就行了：
```html
[kbz@VM_32_5_centos ~]$ cat /etc/supervisor/conf.d/turnserver.conf
[program:turnserver]
#command = /usr/bin/turnserver -c /etc/turnserver/turnserver.conf -v
command = turnserver -c /etc/turnserver/turnserver.conf -v | ts '[%Y-%m-%d %H:%M:%S]'
user = xxx
autostart = true
autorestart = true
stdout_logfile = /var/log/supervisor/turnserver.info.log
stdout_logfile_maxbytes = 100MB
stdout_logfile_backups = 5
stderr_logfile = /var/log/supervisor/turnserver.error.log
```
这样就可以了，就是加了一个 ts 的管道输出。这样日志就有日期了：
```html
[kbz@VM_32_5_centos ~]$ tail /var/log/supervisor/turnserver.info.log
[2018-11-28 02:22:02] 0: IPv4. TCP listener opened on : 192.168.0.1:3478
[2018-11-28 02:22:02] 0: IPv4. UDP listener opened on: 192.168.0.1:3479
[2018-11-28 02:22:02] 0: IPv4. TCP listener opened on : 192.168.0.1:3479
```

## 后记 -- 2022-03-09
后面发现这样子好像也有其他问题导致， 刚好 coturn 的 `4.5.2` 的版本有优化了 log 输出的方式，允许定义 log 的输出格式: [4.5.2/ChangeLog](https://github.com/coturn/coturn/blob/upstream/4.5.2/ChangeLog)
```text
- merge PR #618 (by Paul Wayper)
		* Print full date and time in logs
		* Add new options: "new-log-timestamp" and "new-log-timestamp-format"
```

只要将版本升级上来，并且配置这两个参数，就可以了。



















