---
title: VerneMQ 使用docker进行集群部署
date: 2018-11-19 14:06:02
tags: 
    - mqtt
    - vernemq
    - docker
categories: webrtc相关
---
## 前言
之前在测试环境，已经有进行集群部署了： {% post_link vernemq-cluster %}。但是这种部署方式不够灵活，尤其是像后面生产环境要在全球多个点部署好几台，其实很不方便。再加上如果有版本迭代的话，就更不方便了。所以后面考量了一下，决定还是用docker来进行集群部署。
虽然 VerneMQ 有使用docker搭建的教程： [传送门](https://vernemq.com/docs/installation/docker.html), 但是在使用中，使用 VerneMQ 原生镜像，修改测试文件无效，只能在 run 命令中指定参数，这个就非常长了，所以就使用自定义 Dockerfile ,自己创建镜像。
而且这个 docker 部署集群，会分别在中国区，新加坡，美区，欧洲 四个区域部署。
<!--more-->
## Docker 安装
这个就没啥好说，肯定服务器要先装docker： [参考链接](https://docs.docker.com/install/linux/docker-ce/centos/#install-using-the-repository)
## 镜像制作
Dockerfile 构建镜像, 创建一个文件 Dockerfile， 内容如下：
```html
FROM centos:latest

RUN yum -y update \

&& yum install -y wget \

&& wget https://bintray.com/artifact/download/erlio/vernemq/rpm/centos7/vernemq-1.6.1-1.el7.x86_64.rpm \

&& yum install vernemq-1.6.1-1.el7.x86_64.rpm -y \

&& rm -rf vernemq-1.6.1-1.el7.x86_64.rpm 

CMD ["vernemq","start"]
```
接下来 build 镜像,进入 Dockfile 所在目录，命令如下:
```html
docker build -t airdroid/vernemq:1.6.1 .
```
通过docker images 查看镜像:
```html
[root@VM_16_14_centos ~]# docker images
REPOSITORY             TAG                 IMAGE ID            CREATED             SIZE
airdroid/vernemq       1.6.1               f5514b4637e7        19 hours ago        336MB
```
## 运行容器
既然镜像已经创建了，那么接下来就是将容器跑起来了。以中国区为例，其他区相同，–name修改为对应区域即可：
```html
docker run -dit \
-p 1889:1889 -p 1883:1883 -p 44053:44053 -p 1900:1900 \
–name vernemq-release-cn-1 \
-v /etc/vernemq:/etc/vernemq -v /data/server/docker/vernemq/log:/var/log/vernemq \
f5514b4637e7 /bin/bash
```
参数解释：
- **-dit**: 以交互式的方式在后台运行
- **-p**:端口映射，将容器的服务端口映射到容器外部，mqtt 我们需要将 1883 1889 1900 44053 映射
- **–name**: 容器别名，便于后续识别和操作便利
- **-v**: 交容器内的日志和配置文件映射到宿主机，便于在容器外操作
- **f5514b4637e7**: 镜像 vernemq id

容器启动后，可以通过**docker ps** 查看容器是否正常:
```html
[root@VM_16_14_centos ~]# docker ps
CONTAINER ID        IMAGE                    COMMAND                  CREATED             STATUS              PORTS                                                                                              NAMES
16cb036546c8        f5514b4637e7             "/bin/bash"              19 hours ago        Up 18 hours         0.0.0.0:1883->1883/tcp, 0.0.0.0:1889->1889/tcp, 0.0.0.0:1900->1900/tcp, 0.0.0.0:44053->44053/tcp   vernemq-release-cn-1
```
可以看到已经在运行了，用ping命令试一下：
```html
[kbz@VM_16_14_centos docker]$ sudo docker exec vernemq-release-cn-1 vernemq ping
pong
```
修改 **vernemq.conf** 配置文件，每个区的配置文件都会有一些差别，比如 nodename，webhook地址，clustering 等等。以中国区为例：
```html
allow_anonymous = off
allow_register_during_netsplit = on
allow_publish_during_netsplit = on
allow_subscribe_during_netsplit = on
allow_unsubscribe_during_netsplit = on
allow_multiple_sessions = off
max_inflight_messages = 20
max_online_messages = 1000
max_offline_messages = 1000
max_message_size = 0
upgrade_outgoing_qos = off
listener.max_connections = 10000
listener.nr_of_acceptors = 10
listener.tcp.default = 127.0.0.1:1883
listener.vmq.clustering = 0.0.0.0:44053
listener.http.default = 0.0.0.0:1999
listener.mountpoint = off
systree_enabled = on
systree_interval = 20000
graphite_enabled = off
graphite_host = localhost
graphite_port = 2003
graphite_interval = 20000
shared_subscription_policy = prefer_local
plugins.vmq_passwd = off
plugins.vmq_acl = off
plugins.vmq_diversity = on
plugins.vmq_webhooks = on
vmq_webhooks.mywebhook1.hook = on_unsubscribe
vq_webhooks.mywebhook1.endpoint = http://172.xx.16.14:7777/webhook
vmq_webhooks.mywebhook2.hook = on_subscribe
vmq_webhooks.mywebhook2.endpoint = http://172.xx.16.14:7777/webhook
vmq_webhooks.mywebhook3.hook = on_register
vmq_webhooks.mywebhook3.endpoint = http://172.xx.16.14:7777/webhook
vmq_webhooks.mywebhook4.hook = on_publish
vmq_webhooks.mywebhook4.endpoint = http://172.xx.16.14:7777/webhook
vmq_webhooks.mywebhook5.hook = on_offline_message
vmq_webhooks.mywebhook5.endpoint = http://172.xx.16.14:7777/webhook
vmq_webhooks.mywebhook6.hook = on_deliver
vmq_webhooks.mywebhook6.endpoint = http://172.xx.16.14:7777/webhook
vmq_webhooks.mywebhook7.hook = on_client_wakeup
vmq_webhooks.mywebhook7.endpoint = http://172.xx.16.14:7777/webhook
vmq_webhooks.mywebhook8.hook = on_client_offline
vmq_webhooks.mywebhook8.endpoint = http://172.xx.16.14:7777/webhook
vmq_webhooks.mywebhook9.hook = on_client_gone
vmq_webhooks.mywebhook9.endpoint = http://172.xx.16.14:7777/webhook
plugins.vmq_bridge = off
vmq_acl.acl_file = /etc/vernemq/vmq.acl
vmq_acl.acl_reload_interval = 10
vmq_passwd.password_file = /etc/vernemq/vmq.passwd
vmq_passwd.password_reload_interval = 10
vmq_diversity.script_dir = /usr/share/vernemq/lua
vmq_diversity.auth_postgres.enabled = off
vmq_diversity.auth_mysql.enabled = off
vmq_diversity.auth_mongodb.enabled = off
vmq_diversity.auth_redis.enabled = on
vmq_diversity.redis.host = 172.26.xx.xx
vmq_diversity.redis.port = 6382
vmq_diversity.redis.database = 3
log.console = file
log.console.level = debug
log.console.file = /var/log/vernemq/console.log
log.error.file = /var/log/vernemq/error.log
log.syslog = off
log.crash = on
log.crash.file = /var/log/vernemq/crash.log
log.crash.maximum_message_size = 64KB
log.crash.size = 10MB
log.crash.rotation = $D0
log.crash.rotation.keep = 5
nodename = VerneMQ@10.0.0.1
distributed_cookie = vmq_airdroid
erlang.async_threads = 64
erlang.max_ports = 262144
leveldb.maximum_memory.percent = 70
log.console=file
erlang.distribution.port_range.minimum = 6000
erlang.distribution.port_range.maximum = 7999
listener.tcp.default = 0.0.0.0:1883
listener.ws.default = 0.0.0.0:1889
listener.vmq.clustering = 10.0.0.1:44053
listener.http.metrics = 0.0.0.0:1900
```
可以直接进去容器里面修改配置文件： **sudo docker exec -it vernemq-release-cn-1 /bin/bash**
```html
[kbz@VM_16_14_centos docker]$ sudo docker exec -it vernemq-release-cn-1 /bin/bash
[root@16cb036546c8 /]# cd /etc/vernemq/
[root@16cb036546c8 vernemq]# ll
total 76
-rw-r--r-- 1 root root 33880 Nov 17 13:54 vernemq.conf
-rw-r--r-- 1 root root 33880 Nov 19 02:35 vernemq.conf_bak
-rw-r--r-- 1 root root     8 Nov  2 14:49 vmq.acl
[root@16cb036546c8 vernemq]# vim vernemq.conf
```
当然还有一种更简单的方式，前面在run容器的时候，已经通过**-v**参数将配置文件映射到宿主机了，这样我们就可以直接在容器外进行操作了：
```html
[kbz@VM_16_14_centos log]$ cd /etc/vernemq/
[kbz@VM_16_14_centos vernemq]$ ll
total 76
-rw-r--r-- 1 root root 33880 Nov 17 13:54 vernemq.conf
-rw-r--r-- 1 root root 33880 Nov 19 02:35 vernemq.conf_bak
-rw-r--r-- 1 root root     8 Nov  2 14:49 vmq.acl
```
修改配置文件之后，接下来重启 vernemq 服务，容器不用重启，只需要重启容器里面的 vernemq 服务即可：
```html
docker exec vernemq-release-cn-1 vernemq restart
```
### 配置其他三个区
这样子中国区 VerneMQ 服务就配置好了。接下来就配置其他三个区的：
首先要先把当前的容器镜像导出来，然后通过香港中转服务上传分发到其他三台服务器：
```html
docker export -o vernemq.tar 容器ID
```
接下来就在其他区的服务器上，导入这个镜像：
```html
docker import vernemq.tar airdroid/vernemq:1.6.1
```
这样子镜像也就有了，接下来就跟之前中国区的操作一样， run -> edit config -> vernemq restart
我们可以看下log (之前run的时候，也已经把log的路径也映射到宿主机了，因此不用进入容器，直接在宿主机上访问即可)：
```html
[kbz@VM_16_14_centos vernemq]$ cd /data/server/docker/vernemq/log/
[kbz@VM_16_14_centos log]$ tail console.log
2018-11-19 06:57:08.209 [debug] <0.222.0>@plumtree_broadcast:exchange:500 started plumtree_metadata_manager exchange with 'VerneMQ@10.0.0.2' (<0.9782.4>)
```
## weave 网络设置
通过上面的步骤，我们已经在4台机子上用docker部署了VerneMQ服务了，所以接下来要设置集群了。 而集群有一个前提就是各个服务器的容器之间的通信问题。
当容器分布在多个不同的主机上时，这些容器之间的相互通信变得复杂起来。容器在不同主机之间都使用的是自己的私有IP地址，不同主机的容器之间进行通讯需要将主机的端口映射到容器的端口上，而且IP地址需要使用主机的IP地址。
虽然 Docker 1.9 Overlay Network 已经可以实现跨主机网络互通，但是在使用 Dockerfile 原生网络,处理容器间通信的时候，不仅配置复杂，而且稳定性差。 
容器间通信，我们采用开源的weave 容器网络解决方案，可以简单的理解，把weave 当作一个交换机，把各个区域的容器接入到weave,达到网络通信 和数据传输的目的。
### Weave介绍
Weave是Github上一个比较热门的Docker容器网络方案，具有非常良好的易用性且功能强大。Weave通过创建虚拟网络使Docker容器能够跨主机通信并能够自动相互发现。通过weave网络，由多个容器构成的基于微服务架构的应用可以运行在任何地方：主机，多主机，云上或者数据中心。应用程序使用网络就好像容器是插在同一个网络交换机上一样，不需要配置端口映射，连接等。在weave网络中，使用应用容器提供的服务可以暴露给外部，而不用管它们运行在何处。类似地，现存的内部系统也可以接受来自于应用容器的请求，而不管容器运行于何处。
一个Weave网络由一系列的'peers'构成----这些weave路由器存在于不同的主机上。每个peer都由一个名字，这个名字在重启之后保持不变.这个名字便于用户理解和区分日志信息。每个peer在每次运行时都会有一个不同的唯一标识符（UID）.对于路由器而言，这些标识符不是透明的，尽管名字默认是路由器的MAC地址。
Weave路由器之间建立起TCP连接，通过这个连接进行心跳握手和拓扑信息交换，这些连接可以通过配置进行加密。peers之间还会建立UDP连接，也可以进行加密，这些UDP连接用于网络包的封装，这些连接是双工的而且可以穿越防火墙。Weave网络在主机上创建一个网桥,每个容器通过veth pari连接到网桥上，容器由用户或者weave网络的IPADM分配IP地址。
简单的来说就是：weave 就是相当于以docker 容器的形式，启动一个新的虚拟网卡，每个区域的主机通过这个交换机连接起来， 然后再给容器分配子网，达到通信的目的。
### 下载安装
```html
wget https://github.com/zettio/weave/releases/download/latest_release/weave
```
```html
chmod +weave
```
```html
cp weave  /usr/local/bin/
```
### 启动容器并创建虚拟网络
执行 **weave launch**, 拉取镜像，并开启容器:
```html
[root@VM_16_14_centos ~]# docker ps
400ab0e2b551        weaveworks/weave:2.5.0   "/home/weave/weaver …"   2 days ago          Up 19 hours           weave
```
这样子第一台宿主机的weave就部署成功了，接下来部署第二台宿主机。第二台宿主机部署步骤稍微有点不同,我们需要为这台宿主机的 weave 路由器指定第一台宿主机的 IP 地址,命令如下:
```html
weave launch <first-host-IP-address>
```
比如,在新加坡的服务器执行:
```html
weave launch 172.xx.16.14
```
依次类推，在美区执行新加坡的IP，在欧洲执行美区IP。 这样子就通过 weave 将整个虚拟网络都串起来了。
最后，通过weave status peers ,查看路由表，四个区域的主机都已建立了连接，如下图:
```html
[root@VM_16_14_centos ~]# weave status peers
  xx:xx:a6:26:3f:5e(VM_16_14_centos)
 -> 172.xx.0.12:6783      xx:xx:53:67:0a:e0(VM_0_12_centos)     established
 <- 172.xx.0.10:48189     xx:xx:fc:0a:3f:3d(VM_0_10_centos)     established
 -> 172.xx.32.5:6783      xx:xx:30:4e:2f:cd(VM_32_5_centos)     established
  xx:xx:fc:0a:3f:3d(VM_0_10_centos)
 -> 172.xx.16.14:6783     xx:xx:a6:26:3f:5e(VM_16_14_centos)    established
 <- 172.xx.32.5:44402     xx:xx:30:4e:2f:cd(VM_32_5_centos)     established
 -> 172.xx.0.12:6783      xx:xx:53:67:0a:e0(VM_0_12_centos)     established
  xx:xx:53:67:0a:e0(VM_0_12_centos)
 -> 172.xx.32.5:6783      xx:xx:30:4e:2f:cd(VM_32_5_centos)     established
 <- 172.xx.16.14:14404    xx:xx:a6:26:3f:5e(VM_16_14_centos)    established
 <- 172.xx.0.10:44685     xx:xx:fc:0a:3f:3d(VM_0_10_centos)     established
  xx:xx:30:4e:2f:cd(VM_32_5_centos)
 <- 172.xx.16.14:28463    xx:xx:a6:26:3f:5e(VM_16_14_centos)    established
 <- 172.xx.0.12:43704     xx:xx:53:67:0a:e0(VM_0_12_centos)     established
 -> 172.xx.0.10:6783      xx:xx:fc:0a:3f:3d(VM_0_10_centos)     established
```
### 给容器分配网段
既然四个区域的宿主机都建立了连接，但是容器如果通信，还需要给各个区域的容器分配ip,这里分配的是10.0.0.0/24 的网段，命令如下：
```html
中国:   weave attach 10.0.0.1/24 vernemq-release-cn-1
新加坡:  weave attach 10.0.0.2/24 vernemq-release-sg-1
美国:   weave attach 10.0.0.3/24 vernemq-release-us-1
法国:   weave attach 10.0.0.4/24 vernemq-release-sa-1
```
接下来查看容器分配的ip有没有生效，以中国区为例,如下图:
```html
[root@VM_16_14_centos ~]# docker exec vernemq-release-cn-1 ifconfig
ethwe: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1376
      inet 10.0.0.1  netmask 255.255.255.0  broadcast 10.0.0.255
      ether xx:xx:12:64:38:91  txqueuelen 0  (Ethernet)
      RX packets 543583  bytes 54337232 (51.8 MiB)
      RX errors 0  dropped 0  overruns 0  frame 0
      TX packets 546281  bytes 54916728 (52.3 MiB)
      TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
      inet 127.0.0.1  netmask 255.0.0.0
      loop  txqueuelen 1  (Local Loopback)
      RX packets 60255  bytes 6723522 (6.4 MiB)
      RX errors 0  dropped 0  overruns 0  frame 0
      TX packets 60255  bytes 6723522 (6.4 MiB)
      TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```
可以看 10.0.0.1 已经是了。
### 建立连接
默认情况下，容器间都是相互ping不通的。需要使用weave connect命令在weave的路由器之间建立连接,连接方式如下:
```html
weave connect 宿主机ip
```
比如，在中国区执行：
```html
# 注意，因为虚拟网络已经建立起来了，因此只需要在一台执行即可实现全部通信，也就是说在中国区这一台，就可以直接执行 connect 这4台
weave connect 172.xx.0.12
```
测试 ping 的情况，在中国区(10.0.0.1)测试如下，其他区域类似：
```html
[root@VM_16_14_centos ~]# docker exec vernemq-release-cn-1 ping 10.0.0.2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=45.9 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=45.9 ms
64 bytes from 10.0.0.2: icmp_seq=3 ttl=64 time=46.0 ms
64 bytes from 10.0.0.2: icmp_seq=4 ttl=64 time=45.9 ms
[root@VM_16_14_centos ~]# docker exec vernemq-release-cn-1 ping 10.0.0.3
PING 10.0.0.3 (10.0.0.3) 56(84) bytes of data.
64 bytes from 10.0.0.3: icmp_seq=1 ttl=64 time=159 ms
64 bytes from 10.0.0.3: icmp_seq=2 ttl=64 time=159 ms
64 bytes from 10.0.0.3: icmp_seq=3 ttl=64 time=159 ms
64 bytes from 10.0.0.3: icmp_seq=4 ttl=64 time=159 ms
[root@VM_16_14_centos ~]# docker exec vernemq-release-cn-1 ping 10.0.0.4
PING 10.0.0.4 (10.0.0.4) 56(84) bytes of data.
64 bytes from 10.0.0.4: icmp_seq=1 ttl=64 time=199 ms
64 bytes from 10.0.0.4: icmp_seq=2 ttl=64 time=199 ms
64 bytes from 10.0.0.4: icmp_seq=3 ttl=64 time=199 ms
64 bytes from 10.0.0.4: icmp_seq=4 ttl=64 time=199 ms
64 bytes from 10.0.0.4: icmp_seq=5 ttl=64 time=199 ms
```
这样子就可以实现不同主机之间容器的通信了。
## 集群部署
各个区域的容器建正常通信以后，就可以建立mqtt cluster 了。直接在中国区这一台执行：
```html
docker exec vernemq-release-cn-1  vmq-admin cluster join discovery-node=VerneMQ@10.0.0.2
```
```html
docker exec vernemq-release-cn-1  vmq-admin cluster join discovery-node=VerneMQ@10.0.0.3
```
```html
docker exec vernemq-release-cn-1  vmq-admin cluster join discovery-node=VerneMQ@10.0.0.4
```
当然如果嫌弃前缀太长的话，也可以进入到容器里面再执行。
接下来测试集群是否正常：
```html
[root@VM_16_14_centos ~]# docker exec vernemq-release-cn-1 vmq-admin cluster show
+----------------+-------+
|      Node      |Running|
+----------------+-------+
|VerneMQ@10.0.0.1| true  |
|VerneMQ@10.0.0.2| true  |
|VerneMQ@10.0.0.3| true  |
|VerneMQ@10.0.0.4| true  |
+----------------+-------+
```
正常work，至此，集群搭建完成。
## 部署过程中遇到的问题总结
### 使用vernqmq 原生镜像，修改测试文件无效
解决方式： 自定义Dockerfile ,创建镜像
### 使用Dockerfile 原生网络,处理容器间通信，配置复杂，稳定性差
解决方式： 使用docker 容器网络 weave 组件，处理
### 配置欧洲docker时，和欧洲宿主机的ip冲突
配置欧洲docker时，和欧洲宿主机的ip冲突，欧洲的宿主机ip 为 172.17.0.0 网段，而docker 虚拟网络的IP也是172.17.0.0 网段，造成了冲突。
解决方案： 修改docker 默认的虚拟网卡IP，这里我们设置的是192.168.0.0/24 网段，配置如下：
修改docker daemon.json 文件，内容如下：
```html
[root@VM_16_14_centos ~]# cat /etc/docker/daemon.json 
{
"bip": "192.168.0.1/24"
}
```
重启docker 服务
```html
systemctl restart docker
```
查看docker 默认虚拟网卡的IP，是否更改
```html
[root@VM1614_centos ~]# ifconfig

docker0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
      inet 192.168.0.1  netmask 255.255.255.0  broadcast 192.168.0.255
      ether xx:xx:d0:f1:3e:c3  txqueuelen 0  (Ethernet)
      RX packets 410728  bytes 29667929 (28.2 MiB)
      RX errors 0  dropped 0  overruns 0  frame 0
      TX packets 454886  bytes 669202312 (638.2 MiB)
      TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```
但是还有一个问题存在： 如果后面重启docker 或重启容器时，容器的ip会消失。 所以解决方法就是 重新执行 weave 绑定ip：
```html
weave attach ip 容器名
```
举个例子，比如中国区，这个由于意外原因，重启系统或重启容器了，需要执行如下命令：
```html
weave attach 10.0.0.1/24 vernemq-release-cn-1
```
这个 weave ip  是及时生效的，附加后，马上生效，不需要重启容器。 
<font color=red>（这个很容易漏掉，如果有脚本维护的话，那么就要写入到shell中，不然很容易忘记）</font>
## 其他补充
容器的操作可以通过2种方式试下，一种是在容器外，一种是容器内：
```html
容器内： docker exec -it 容器id 或容器名  /bin/bash
容器外： docker exec 容器id 或容器名 shell命令
```
常用指令：
```html
查看当前镜像:  docker images

查看当前运行的容器: docker ps

测试当前vmtt 服务是否正常: docker exec vernmemq-release-xxx-1 ping

查看当前集群是否正常: docker exec vernemq-release-xxx-1 vmq-admin cluster show

mqtt 服务 开始、停止、重启: docker exec vernemq-release-xxx-1 vernemq start|stop|restart
```
文件映射到宿主机：
启动容器时，已经将容器内的配置文件目录和log 映射到容器外，具体路径如下:
查看及修改配置配置文件:
```html
cat|vim  /etc/vernemq/vernemq.conf 
```
查看日志文件:
```html
cat /data/server/docker/vernemq/log/*.log
```
最后感谢树礼的整理。



