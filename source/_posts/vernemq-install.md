---
title: webrtc 的 signal 服务器 VerneMQ 的搭建
date: 2018-05-30 00:25:38
tags: mqtt
categories: webrtc相关
---
这段时间正在做一个基于webrtc 技术的项目，其中就涉及到了 signal 服务器的搭建， 因为都是长链接， 相对于比较重的websocket，我选择更轻量级的 mqtt 协议来进行SDP 和 ICE 信息的传输， 
参考了目前市面上实现了mqtt协议的服务，最后选择了 vernemq 这一个开源的服务来实现， 地址如下： https://vernemq.com/, 
好处就是一个是开源的服务，另一个就是基于分布式的，当然还有一些其他的。 

接下来就是搭建了，这个其实官方文档写的已经很详细了，有docker安装，也有下载二进制安装， 还有直接下载源码编译安装。这边我主要是用docker安装和二进制安装两种方式

### 1. docker 安装
文档如下：https://vernemq.com/docs/installation/docker.html

我的环境是centos7，在已经安装docker的环境下，直接执行 
{% codeblock %}
docker run -e "DOCKER_VERNEMQ_ALLOW_ANONYMOUS=on" --name vernemq1 -d erlio/docker-vernemq
{% endcodeblock %}
因为默认是有权限校验的，所以通过 <b>DOCKER_VERNEMQ_ALLOW_ANONYMOUS</b> 将权限关掉，允许任何人可以连上来。
实际安装的log如下：
<!--more-->
{% codeblock %}
[root@VM_156_200_centos ~]# docker run -e "DOCKER_VERNEMQ_ALLOW_ANONYMOUS=on" --name vernemq1 -d erlio/docker-vernemq
Unable to find image 'erlio/docker-vernemq:latest' locally
Trying to pull repository docker.io/erlio/docker-vernemq ...
latest: Pulling from docker.io/erlio/docker-vernemq
f2b6b4884fc8: Already exists
8fb9ffc6cdae: Pull complete
4795a2a28cbd: Pull complete
e832eaa452fa: Pull complete
2ee372502efc: Pull complete
e794b0ff5b5d: Pull complete
6f71f31fa569: Pull complete
60258bf8f648: Pull complete
Digest: sha256:5a5a59c2d10fad26e7e8183b0460c4562e7b61c3b1fe397d042d9e69b3c4930d
Status: Downloaded newer image for docker.io/erlio/docker-vernemq:latest
f4a4c6601eba6d3b50af5f1d09bf5a45993ce6dec0968006a572568817c67e7f
{% endcodeblock %}
这样就跑起来了，查看一下命令看下，可以看到已经跑出来了
{% codeblock %}
[root@VM_156_200_centos ~]# docker ps
CONTAINER ID        IMAGE                  COMMAND                  CREATED              STATUS              PORTS                                                                            NAMES
f4a4c6601eba        erlio/docker-vernemq   "start_vernemq"          About a minute ago   Up About a minute   1883/tcp, 4369/tcp, 8080/tcp, 8883/tcp, 8888/tcp, 9100-9109/tcp, 44053/tcp       vernemq1
61b852332b5a        node-hello             "npm start"              4 days ago           Up 4 days           0.0.0.0:3001->3001/tcp                                                           node-hello-1
{% endcodeblock %}
我们知道vernemq是有分布式的，因此这边再建一个节点进入进去，命令如下：
{% codeblock %}
docker run -e "DOCKER_VERNEMQ_DISCOVERY_NODE=<IP-OF-VERNEMQ1>" --name vernemq2 -d erlio/docker-vernemq
{% endcodeblock %}
首先我们找到 vernemq1 节点的 ip 地址， 通过这个指令 <b>docker inspect <containername/cid> | grep \"IPAddress\"</b>
具体的操作log如下：
{% codeblock %}
[root@VM_156_200_centos ~]# docker inspect f4a4c6601eba | grep \"IPAddress\"
            "IPAddress": "172.17.0.5",
                    "IPAddress": "172.17.0.5",
{% endcodeblock %}
得到 ip 为： <b>172.17.0.5</b>
接下来就加进去：
{% codeblock %}
[root@VM_156_200_centos ~]# docker run -e "DOCKER_VERNEMQ_DISCOVERY_NODE=172.17.0.5" --name vernemq2 -d erlio/docker-vernemq
ebc8dca4feba9d6e0457452be909985a68519aef29376d403b2d782e7ea5e60d
[root@VM_156_200_centos ~]# docker ps
CONTAINER ID        IMAGE                  COMMAND                  CREATED             STATUS              PORTS                                                                            NAMES
ebc8dca4feba        erlio/docker-vernemq   "start_vernemq"          5 seconds ago       Up 4 seconds        1883/tcp, 4369/tcp, 8080/tcp, 8883/tcp, 8888/tcp, 9100-9109/tcp, 44053/tcp       vernemq2
f4a4c6601eba        erlio/docker-vernemq   "start_vernemq"          5 minutes ago       Up 5 minutes        1883/tcp, 4369/tcp, 8080/tcp, 8883/tcp, 8888/tcp, 9100-9109/tcp, 44053/tcp       vernemq1
61b852332b5a        node-hello             "npm start"              4 days ago          Up 4 days           0.0.0.0:3001->3001/tcp                                                           node-hello-1
a1228658c74b        nsqio/nsq              "/nsqadmin --looku..."   3 weeks ago         Up 3 weeks          4150-4151/tcp, 4160-4161/tcp, 4170/tcp, 0.0.0.0:32770->4171/tcp                  nsq_nsqadmin_1
3612b4785be5        nsqio/nsq              "/nsqd --lookupd-t..."   3 weeks ago         Up 3 weeks          4160-4161/tcp, 4170-4171/tcp, 0.0.0.0:32772->4150/tcp, 0.0.0.0:32771->4151/tcp   nsq_nsqd_1
a24d5cc10d29        nsqio/nsq              "/nsqlookupd"            3 weeks ago         Up 3 weeks          4150-4151/tcp, 4170-4171/tcp, 0.0.0.0:32769->4160/tcp, 0.0.0.0:32768->4161/tcp   nsq_nsqlookupd_1
a9ee06d55082        nginx:v3               "nginx -g 'daemon ..."   3 weeks ago         Up 3 weeks          0.0.0.0:80->80/tcp                                                               webserver3
89e76cdbfe98        nginx                  "nginx -g 'daemon ..."   3 weeks ago         Up 3 weeks          0.0.0.0:8080->80/tcp                                                             webserver
{% endcodeblock %}
这样就建了两个节点了。
如果我们叫检查整个集群的状态的话，用这个：
{% codeblock %}
[root@VM_156_200_centos ~]# docker exec vernemq1 vmq-admin cluster show
+------------------+-------+
|       Node       |Running|
+------------------+-------+
|VerneMQ@172.17.0.5| true  |
|VerneMQ@172.17.0.6| true  |
+------------------+-------+
{% endcodeblock %}
当然 vernemq 也有提供一个 http 的管理后台：
具体可以看这个： https://vernemq.com/docs/http-administration/
如果要执行 vernemq 指令的话， 可以用这种方式来执行
{% codeblock %}
[root@VM_156_200_centos ~]# docker exec vernemq1 vernemq ping
pong
{% endcodeblock %}
前半部分是 docker 指定要执行指令的容器， 后面半部分的 vernemq ping 就是正常的要执行的指令

---

### 2. 二进制rpm包 安装
之所以后面又重新用 rpm包安装是因为之前用docker安装的时候，每次启动的时候，都要输入一大堆的配置项，感觉不太方便，所以后面就直接装全局的服务了，这样子可以直接读取和修改配置文件，会显得比较方便
通过 https://vernemq.com/downloads/index.html 下载一个 rpm 文件
选择 redhat 系列的
然后直接解压安装
{% codeblock %}
[root@VM_156_200_centos ~]# rpm -Uvh vernemq-1.3.1-1.el7.centos.x86_64_\(1\).rpm
Preparing...                          ################################# [100%]
Updating / installing...
   1:vernemq-1.3.1-1.el7.centos       ################################# [100%]
{% endcodeblock %}
安装完之后，就会有一个 vernemq 目录
这时候就可以启动了
{% codeblock %}
[root@VM_156_200_centos vernemq]# service vernemq start


Starting vernemq (via systemctl):                          [  OK  ]
{% endcodeblock %}

用一下 ping 指令
{% codeblock %}
[root@VM_156_200_centos vernemq]# vernemq ping
pong
{% endcodeblock %}
成功了。

配置文件在这边
{% codeblock %}
[root@VM_156_200_centos vernemq]# cd /etc/vernemq/
[root@VM_156_200_centos vernemq]# ll
total 36
-rw-r--r-- 1 root root 30782 Mar 21 00:06 vernemq.conf
-rw-r--r-- 1 root root     8 Mar 21 00:06 vmq.acl
{% endcodeblock %}

---

总结： 这个就是安装过程，那么就下来就是使用了，当然是结合权限校验来使用，下一篇会讲怎么用权限校验: {% post_link vernemq-verify %}