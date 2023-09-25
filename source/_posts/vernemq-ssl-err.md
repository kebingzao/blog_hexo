---
title: 记一次大陆地区 VerneMQ 出现部分证书校验失败的情况
date: 2023-08-25 13:45:05
tags: 
- mqtt
- vernemq
- nginx
categories: webrtc相关
---
## 原因
前段时间有发生过一个事情，就是 测试人员在他的 windows 或者 mac 中有出现连接 mqtt VerneMQ 失败的情况, 而且比较神奇的地方在于:
1. 如果搭梯子的话，是可以连接成功的
2. 不搭梯子，将 chrome 浏览器换成 Firefox， 也是可以连接的
3. 在一些机器上，通过修改 DNS 解析，比如改成 `8.8.8.8`, 也可以正常

经过排查之后，发现确实在连接失败的时候，VerneMQ 服务器会出现部分证书相关的错误提示:
```text
 [info] <0.20467.939> TLS server: In state certify received CLIENT ALERT: Fatal - Certificat Unknown
```

## 排查
首先排查证书问题，确定我们的证书是对的，nginx 都可以正常使用。而且我们的 crt 证书文件是有包含整个验证环节的 fullchain 的聚合证书(包含多个证书链内容的合成证书)。

然后对比一下 chrome 和 Firefox 的证书校验机制，Firefox 带了自己的 CA 证书链， chrome 的 CA 走的是系统的， 因此怀疑是 VerneMQ 证书链对合成证书的验证环节有问题??

分析浏览器该站点的证书链路发现，我们用的证书有属于二级证书签发，中间有一个 Go Daddy Secure CA 校验节点
<!--more-->
![1](1.png)

所以我们怀疑是 VerneMQ 对合成证书的支持有问题， 导致 VerneMQ 返回证书校验失败，这时候浏览器就需要去 https://certs.godaddy.com/repository 下载并校验，而大陆地区的 `certs.godaddy.com` 域名应该在 8 月初在大陆地区被拦截了。而由于走不一样的 DNS 回的 `certs.godaddy.com` 的 IP 不同，可能存在部分还能访问的 IP 节点，因此也可以解释`部分机器可以访问` / `改 DNS 后可以访问` / `搭梯子可以访问`的现象。

后面查了一下 VerneMQ 的 issue，发现有类似的 issue: [SSL/TLS Server and Client Authentication with Intermediate Certificates](https://github.com/vernemq/vernemq/issues/551)


## 解决
后面使用 nginx 来代理 wss 请求，反向代理到 VerneMQ 的 ws 请求

将原先 vernemq 的 1890 端口的 wss 的配置项，去掉:
```text
[kbz@VM-16-8-centos ~]$ cat /etc/vernemq/vernemq.conf | grep 'wss'
#listener.wss.default = 0.0.0.0:1890
#listener.wss.certfile = /etc/vernemq/foo.cn.crt
#listener.wss.keyfile = /etc/vernemq/foo.cn.key
```

然后用 nginx 转发 1890 的 wss 协议到 vernemq 的 1889 的 ws 协议:
```text
[kbz@VM-16-8-centos ~]$ cat /etc/nginx/sites-enabled-foo/signal.foo.cn.conf
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}
upstream mqtt_cn{
        server 127.0.0.1:1889 fail_timeout=0;
}

server {
    server_name  signal.foo.cn;
    listen       1890 ssl http2;
    include      ssl/ssl_go_foo.cn.conf;
    access_log /var/log/nginx/signal.foo.cn.access.log main;
    access_log /var/log/nginx/signal.foo.cn.4xx5xx.log combined if=$loggable;
    error_log /var/log/nginx/signal.foo.cn.error.log warn;

    location / {
      proxy_pass http://mqtt_cn;
      proxy_read_timeout 1800s;
      proxy_send_timeout 1800s;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_http_version 1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
    }
}
```

这样子有几个好处:
1. nginx 兼容性强，久经我们各个服务考验，行为比较统一，文档也较全，可以确定我们目前用到的各种证书配置方式都是支持的
2. 遇到证书有问题时候（包括续期），Nginx 的重启成本比较低（秒级），而 VerneMQ 这类业务服务的重启较慢（分钟级），同时正常 nginx 代理多个服务的时候，证书是统一换的，因此在更换证书的时候，一般只需要更换一处即可

不过要注意一个点，既然是通过 nginx 转发，意味着会使用到本机的端口，因此除了要将 nginx 的 worker 设置大一点之外，还需要将内核的可映射端口调大才行:
```text
worker_connections 50000; #nginx worker_connections 尽快设置大一些
net.ipv4.ip_local_port_range = 1024 65000 #内核端口范围
```


