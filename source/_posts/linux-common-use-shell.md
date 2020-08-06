---
title: 记录一下自己常用的几个 shell 指令(备忘)
date: 2020-08-06 11:42:05
tags: linux
categories: 系统相关
---
## 前言
感觉自己年纪越来越大，记性越来越不好使了， 所以想做个备忘。将自己常用的一些 shell 指令记一下。 一方面是有时候想用的时候，忘记了。 一方面是有些指令略长， 有时候也会忘记了一部分细节。 所谓好记性不如烂笔头。 还是做下备忘比较靠谱。 (天天用就还好，没那么容易忘，问题是有些指令半个月一个月才用一次，每次都会忘记一部分细节，也是蛋疼)

## 常用指令
### 1.查看程序重启是不是 oom 导致的
有时候程序会无缘无故重启，这时候就要看 error log，是 panic 还是 fatal， 但是有时候 error log 也是空的，但是这时候就要看是不是因为内存原因被系统杀死了，所以查看指令如下:
<!--more-->
```text
sudo dmesg -T | egrep -i -B100 'killed process'
```
如果发现是 `Out of memory` :
```text
[Sun Jun 28 00:01:32 2020] Out of memory: Kill process 14458 (dataflow) score 265 or sacrifice child
[Sun Jun 28 00:01:32 2020] Killed process 14458 (dataflow) total-vm:1573556kB, anon-rss:1029640kB, file-rss:0kB, shmem-rss:0kB
```
那么就要考虑是不是机器的内存不够用了。要不要升内存之类的。

### 2.使用 rz 上传文件，并且覆盖文件
我们有时候会需要往服务器上传文件，正常指令是这样子的:
```text
rz -E
```
但是他并不会覆盖源文件， 所以如果要覆盖源文件的话，那么就是加上参数 `-y`:
```text
[kbz@centos156 lang]$ sudo rz -E -y
z waiting to receive.**B0100000023be50
```

### 3.netstat 工具检测开放端口
有时候我们要检查某一个端口有没有开放，这时候就可以用这个:
```text
netstat -anlp | grep [port]
```
比如
```text
[kbz@centos156 ~]$ netstat -anlp | grep 28881
(No info could be read for "-p": geteuid()=1003 but you should be root.)
tcp6       0      0 :::28881                :::*                    LISTEN      -    
```

### 4.使用 alias 别名来保存
有时候一些指令太长了，又要经常打，好麻烦，所以可以用别名来保存， 具体看 {% post_link linux-alias %}

### 5.grep 多级查询
有时候我们需要进行多次过滤的查询，那么就可以用 grep 配合管道来实现。 比如一下两个是一样的效果
```text
[kbz@VM_32_5_centos ~]$ cat /var/log/supervisor/data.info.lo* | grep 'replace it' | grep '30241' | grep  'e6844a71057d059ab4402f8f9c1ab849' | wc -l
21678
[kbz@VM_32_5_centos ~]$ cat /var/log/supervisor/data.info.lo* | grep 'replace it' | grep '30241' | grep -c 'e6844a71057d059ab4402f8f9c1ab849'
21678
```

### 6.使用 locate 快速查找文件
有时候想要在服务器查找某一个文件，或者某一个目录，就要用这个命令 locate， 会比较快。

比如我想在服务器上找到 zookeeper 这个服务所在的目录，就可以这样做：

```text
[kbz@centos156 ~]$ locate zookeeper
/data/code/go/src/github.com/hashicorp/consul/website/source/intro/vs/zookeeper.html.markdown
/data/code/go/src/github.com/hashicorp/serf/website/source/intro/vs-zookeeper.html.markdown
/data/code/go/src/github.com/samuel/go-zookeeper
/data/code/go/src/github.com/samuel/go-zookeeper/.git
/data/code/go/src/github.com/samuel/go-zookeeper/.gitignore
```
非常快，比 `find --name` 更快,  locate命令其实是find -name的另一种写法，但是要比后者快得多，原因在于它不搜索具体目录，而是搜索一个数据库/var/lib/locatedb，这个数据库中含有本地所有文件信息。Linux系统自动创建这个数据库，并且每天自动更新一次，所以使用locate命令查不到最新变动过的文件。为了避免这种情况，可以在使用locate之前，先使用updatedb 命令，手动更新数据库。 还可以指定目录，比如:
```text
locate /etc/sh
```
搜索etc目录下所有以sh开头的文件

### 7.grep 从多个文件查询字符串
这个也很简单，多个文件，就用 * 通配符，比如:
```text
$ grep 'paypal agreement execute error:' '2017-11-'*.log
```

### 8.查看某个进程有没有启动
```text
ps -aux | grep 'bitdatasrv'
```
比如:
```text
[kbz@localhost supervisor]$ ps -aux | grep 'bitdatasrv'
Warning: bad syntax, perhaps a bogus '-'? See /usr/share/doc/procps-3.2.7/FAQ
kbz        837  0.0  0.0  61200   760 pts/4    S+   07:38   0:00 grep bitdatasrv
airdroid 32235  0.2  0.0 260432 14048 ?        Sl   06:35   0:08 /data/server/goSrv/bitdatasrv/bitdatasrv
```

### 9.设置软链接
比如给 nginx 配置文件设置软链接:
```text
sudo ln -s ../sites/www.com.conf www.com.conf
```


### 10.拷贝文件夹
```text
cp -r /source/a /target/a
```
其实有几个参数可以参考:
- `-b` 同名,备分原来的文件
- `-f` 强制覆盖同名文件
- `-r` 按递归方式保留原目录结构复制文件

### 11.使用 tail -f 追踪实时log
这个几乎是最常见的，而且可以通过 -n 来显示最后的几条，比如:
```text
tail -f -n 1000 a.log
```
显示 a.log 的 最后1000 行，并实时跟进这个log的更新，并显示到console上。

### 12.删除目录
```text
rm -rf dir
```
强行删除目录dir下的所有文件、子目录下的所有文件和目录、删除dir本身。

### 13.解压zip包
以前经常要把前端打的zip包传到服务器上，并解压。最简单的就是:
```text
unzip xxx.zip
```
解压 xxx.zip 到当前目录。 还有一个用法就是解压到指定目录 `-d`:

`unzip xxx.zip -d ..`   或者  `unzip -d .. xxx.zip`  其实就是解压 xxx.zip 到上级目录， 因为有时候会覆盖文件，这时候shell会提示是否覆盖，这时候就要手动打 y（覆盖一个） 或者 a（覆盖全部）， 如果用 -o 参数，那么就会在不提示的情况下进行覆盖:
```text
unzip -o xxx.zip
```
解压 xxx.zip 到当前目录并且在不提示的情况下进行全部覆盖, 或者是:
```text
unzip -o -d /tmp/ xxx.zip
```
解压到 /tmp/目录，并且在不提示的情况下进行全部覆盖。

### 14.获取出口 ip
```text
[kbz@centos156 ~]$ curl http://members.3322.org/dyndns/getip
59.xx.xx.156
```

### 15.获取线路
一般用来查看，挂代理是否成功,比如: 
```text
[kbz@ip-11-0-2-38 ~]$ curl cip.cc
IP	: 54.xxx.xxx.31
地址	: 日本  东京都  东京
运营商	: amazon.com

数据二	: 日本 | 东京都亚马逊(Amazon)公司数据中心

数据三	: 

URL	: http://www.cip.cc/54.xx.xx.31
```


