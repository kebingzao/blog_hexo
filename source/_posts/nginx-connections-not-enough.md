---
title: 记一次 nginx worker_connections 最大可连接数不够用的情况
date: 2022-08-17 20:24:18
tags: nginx
categories: nginx相关
---
## 前言
前段时间在对某一个长连接的 wss 服务进行端口优化的时候， 因为原先的端口是一个不常用端口，想改成一个常用的端口，比如 tls 的 443 端口， 又因为这一台服务器已经有安装 nginx，并启用了 443 端口。

所以就采用了 nginx 转发 ws 端口， 然后变成 wss 的 443 端口。 具体可以看:  {% post_link nginx-proxy-wss-https %}

因为 nginx 的端口转发会需要映射到服务器的可用端口，所以就将服务器的可用端口调成一个比较大的值:
```text
[kbz@VM-16-9-centos ~]$ cat /proc/sys/net/ipv4/ip_local_port_range
1024    65000
```

但是只顾了这个要扩大端口号，而忘了也要调整 nginx 的最大可连接数。 导致上线之后没有多久就出现长连接连不上的情况，查看了一下 nginx 的 error log，发现:
```text
 [alert] 23725#0: *1972528 8000 worker_connections are not enough while connecting to upstream, client: 109.xxx.xxx.92, 
```

## 解决
查了一下，确实是 nginx 的配置文件中，配置的单核最大可连接数只有 8000:
```text
events {
  worker_connections 8000;
}
```
然后 `worker_processes` 是 2 核 (几个 CPU 一般就可以调用几个)， 所以 nginx 的最大可连接数就是 2 * 8000 = 16000 个， 所以一旦长连接超过了这个数量，就会报上述的错误。

所以解决的方式也很简单，后面将其改成了 50000， 配合我们的 2 核(2 个 cpu)， 最大可支持 10w 的最大可连接数 (不可能到 7w， 因为服务器本身的可转发端口就满了)





