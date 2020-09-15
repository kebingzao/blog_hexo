---
title: CentOS 7 systemctl reload nginx.service 不检查配置文件的问题
date: 2020-09-15 15:44:31
tags: nginx
categories: nginx相关
---
## 前言
之前在测试 nginx 的 location 的匹配规则和优先级的时候，有测试了这么一种情况:
```text
    location /images/test.png {
        return 200  'config1';
    }

    location ^~ /images/test.png {
        return 200  'config2';
    }

    location ~ \/images\/test\.png$ {
        return 200  'config3';
    }

    location ~ /images/ {
        return 200  'config4';
    }
```
然后我每次执行的时候:
```text
[root@VM_156_200_centos conf]# sudo systemctl reload nginx.service
[root@VM_156_200_centos conf]# curl http://127.0.0.1/images/test.png
config4
```
神奇的情况发生了， 执行结果竟然是 `config4`, 怎么看都觉得是 `config2` 才是对的。 因为如果普通匹配和正则匹配都存在的话， `^~` 的匹配如果是最长的话，那么肯定是以 `^~` 为准的。 那么为啥是 `config4` ????
<!--more-->
## 排查
而且我每次重载 nginx 配置的时候， 也没有报错啊。 当时这个问题困扰了我很久， 一直到我有一次不用 `systemctl reload nginx.service` 来重载配置文件，而是用原生的 `nginx -s reload` 来重载配置的时候，结果竟然发现报错了:
```text
[root@VM_156_200_centos sbin]# ./nginx -s reload
nginx: [emerg] duplicate location "/images/test.png" in /usr/local/nginx/conf/nginx.conf:86
```
那么问题来了，为啥都是重载的操作， 用 `nginx -s reload` 就会报错，而用 `systemctl reload nginx.service` 就不会报错??

首先要先确定 nginx.conf 是不是真的有错。

### duplicate location 的问题
报的错误是 
```text
duplicate location "/images/test.png"
```
有两个重复的配置， 那么是哪两个呢？？ 从源码来看的话:

![1](1.png)

lq 指向第一个待整理的 location 节点。lx 指向第一个 URI 不以 lq 指向 的节点 URI 为前缀的节点。 当这两个都存在同一个模式的匹配的时候, 比如都是普通字符串匹配，或者都是正则匹配的时候，那么就会报这个错误。简单来说就是这里判断的，只要文件名一样，而且都是 普通匹配 或者 都是 正则匹配，就会这个错。

那么对于本例来说， config1 和 config2 都是普通字符串匹配，并且后面匹配的串都是一样的，都是 `/images/test.png`, 所以就会报这个错。 因此 `nginx -s reload` 这个是对的。 所以只要将 config1 注释掉，或者改为 后面的匹配字符串不一样 ：
```text
location /images/ {
    return 200  'config1';
}
```
这样子也是可以的。 至于测试结果之前为啥是 `config4` ? 这个是因为我前一次的匹配测试结果刚好就是 `config4`, 所以接下来改成那样子之后，用 `systemctl reload nginx.service` 重载的时候，所以其实配置文件一直都没有生效， 还是之前旧的。

### systemctl reload nginx.service 的问题
既然排除了是 nginx.conf 文件有问题， 那么再一个问题，为啥 `nginx -s reload` 可以识别到错误， 而 `systemctl reload nginx.service` 并没有识别到错误？

通过查看 nginx.service 的重载指令:
```text
ExecReload=/bin/kill -s HUP $MAINPID
```
发现这个重载指令并不是 `nginx -s reload` 这个指令，而是上面那个指令。 其实大部分情况在这两个指令应该是相差不大的，但是在 CentOS 7 的时候，刚好会有问题，他不会去重载的时候去校验 nginx.conf 的合法性。 具体可以看: [how to reload nginx - systemctl or nginx -s?](https://superuser.com/questions/710986/how-to-reload-nginx-systemctl-or-nginx-s) 

所以解决方式应该将这个指令改成:
```text
ExecReload=/usr/local/nginx/sbin/nginx -s reload
```
其他指令保持不变。 接下来我们看一下，如果换成这个重载指令的话，会不会可以识别到错误, 还是原来那个有问题的 nginx.conf 的配置文件:
```text
[root@VM_156_200_centos sbin]#  systemctl daemon-reload
[root@VM_156_200_centos sbin]#  systemctl reload  nginx.service
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@VM_156_200_centos sbin]# systemctl status nginx.service
● nginx.service - nginx - high performance web server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: active (running) (Result: exit-code) since Tue 2020-09-15 11:54:04 CST; 2h 2min ago
     Docs: http://nginx.org/en/docs/
  Process: 29306 ExecReload=/usr/local/nginx/sbin/nginx -s reload (code=exited, status=1/FAILURE)
Main PID: 9616 (nginx)
    Tasks: 2
   Memory: 4.4M
   CGroup: /system.slice/nginx.service
           ├─9616 nginx: master process /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
           └─9618 nginx: worker process

Sep 15 11:54:04 VM_156_200_centos systemd[1]: Starting nginx - high performance web server...
Sep 15 11:54:04 VM_156_200_centos systemd[1]: Started nginx - high performance web server.
Sep 15 13:56:40 VM_156_200_centos systemd[1]: Reloading nginx - high performance web server.
Sep 15 13:56:40 VM_156_200_centos systemd[1]: nginx.service: control process exited, code=exited status=1
Sep 15 13:56:40 VM_156_200_centos nginx[29306]: nginx: [emerg] duplicate location "/images/test.png" in /usr/local/nginx/conf/nginx.conf:86
Sep 15 13:56:40 VM_156_200_centos systemd[1]: Reload failed for nginx - high performance web server.
```
可以看到 reload 报错了，通过 status 可以查看报错的信息是对的， 所以证明 reload 的指令修改是对的。

接下来测试一下，修改后的合法的 nginx.conf， 能不能也生效，将 `config1` 注释掉，并且将 `config4` 改成 `config8` :
```text
    #location /images/test.png {
      #  return 200  'config1';
    #}    

    location ^~ /images/test.png {
        return 200  'config2';
      }

    location ~ /images/test\.png$ {
         return 200  'config3';
      }

    location ~ /images/ {
        return 200  'config8';
    }
```
```text
[root@VM_156_200_centos sbin]#  systemctl reload  nginx.service
[root@VM_156_200_centos sbin]# curl 127.0.0.1/images/1
config8
```
可以看到 reload 指令成功，并且配置生效了。 所以这样子改是没有问题的。

### systemctl start 和 stop 也会有问题吗
既然 systemctl reload 之前有问题，那么我也有点好奇，start 指令和 stop 指令会不会也有问题呢？  start 指令是一样的，应该不会有问题， 但是 stop 指令可是不一样哦，会有问题吗，这个也实测一下，同样将 nginx.conf 改到有问题的那个:
```text
[root@VM_156_200_centos ~]# systemctl stop nginx.service
[root@VM_156_200_centos ~]# systemctl start  nginx.service
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@VM_156_200_centos ~]# systemctl status  nginx.service
● nginx.service - nginx - high performance web server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Tue 2020-09-15 11:47:43 CST; 3min 7s ago
     Docs: http://nginx.org/en/docs/
  Process: 6696 ExecStop=/bin/kill -s QUIT $MAINPID (code=exited, status=0/SUCCESS)
  Process: 6636 ExecReload=/bin/kill -s HUP $MAINPID (code=exited, status=0/SUCCESS)
  Process: 6728 ExecStartPre=/usr/local/nginx/sbin/nginx -t -c /usr/local/nginx/conf/nginx.conf (code=exited, status=1/FAILURE)
Main PID: 3797 (code=exited, status=0/SUCCESS)

Sep 15 11:47:43 VM_156_200_centos systemd[1]: Starting nginx - high performance web server...
Sep 15 11:47:43 VM_156_200_centos systemd[1]: nginx.service: control process exited, code=exited status=1
Sep 15 11:47:43 VM_156_200_centos nginx[6728]: nginx: [emerg] duplicate location "/images/test.png" in /usr/local/nginx/conf/nginx.conf:86
Sep 15 11:47:43 VM_156_200_centos nginx[6728]: nginx: configuration file /usr/local/nginx/conf/nginx.conf test failed
Sep 15 11:47:43 VM_156_200_centos systemd[1]: Failed to start nginx - high performance web server.
Sep 15 11:47:43 VM_156_200_centos systemd[1]: Unit nginx.service entered failed state.
Sep 15 11:47:43 VM_156_200_centos systemd[1]: nginx.service failed.
```
先关闭，发现是没问题的。 但是启动的时候，就启动失败了。 用 status 查看了一下，发现就是 `duplicate location`, 所以先 stop， 再 start 的话，这时候因为 nginx.conf 有问题，所以是启动不起来的。会报错。

## 总结
也就是说，如果在 CentOS 7 系统中，如果 nginx 要加入到系统服务的话， 那么 nginx.service 的 reload 指令就要变成 `/usr/local/nginx/sbin/nginx -s reload`, 完整的这个文件如下:
```text
[root@VM_156_200_centos ~]# cat /usr/lib/systemd/system/nginx.service 
[Unit]
Description=nginx - high performance web server
Documentation=http://nginx.org/en/docs/
After=network.target remote-fs.target nss-lookup.target
  
[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStartPre=/usr/local/nginx/sbin/nginx -t -c /usr/local/nginx/conf/nginx.conf
ExecStart=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true
  
[Install]
WantedBy=multi-user.target
```





