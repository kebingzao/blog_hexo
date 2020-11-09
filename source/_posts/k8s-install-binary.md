---
title: CentOS7 使用二进制包安装 kubernetes 集群环境
date: 2020-11-09 15:28:16
tags: kubernetes
categories: kubernetes
---
## 前言
前面经过 {% post_link k8s-install-minikube %} 和 {% post_link k8s-install-kubeadm %} 的实操，接下来为了更深刻的理解 kubernetes， 本节我们来用二进制包的方式来安装 kubernetes 集群环境。

事实上，现在 2020 年了，  kubeadm 已经可以用于 生产环境部署了。 但是毕竟是集成安装的， 很多细节我都不了解。 为了更好的学习 kubernetes，理解里面错综复杂的关系链， 用二进制包的方式来安装 kubernetes 肯定是要的， 本文主要是根据 [Kubernetes The Hard Way](https://github.com/caicloud/kube-ladder/blob/master/tutorials/lab3-manual-installtion.md) 这一篇文章的基础上，一步一步去实践安装的， 本身那一篇的篇幅就很长， 本文再扩展一下，再实践一下，那就更长了。 可以理解为实践篇。
<!--more-->

## 机器准备
本次准备了三台机子， 一台 master， 两台 node (事实上安装完最后会发现 master 这一台只做控制，并没有加入到集群中，真正的集群只有那两台 node)

|环境参数|值|
|---|---|
|系统| CentOS 7 |
|CPU| 2 核 |
|内存| 4 G |
|能否科学上网| 能 (腾讯云位于美西的机器) |
|三台直接可以用内网相互访问| 能 |

这三台的内网 ip:
- 172.26.64.9    worker-master
- 172.26.64.2    worker-1
- 172.26.64.10   worker-2

## 每一台的环境都要先配置
### 1. 关闭防火墙 / 开放端口
```text
[root@worker-master ~]# systemctl stop firewalld
[root@worker-master ~]# systemctl disable firewalld
```
关闭防火墙主要是为了让端口开放，按照官方文档，这些端口都会用到，都不要限制的。

![1](1.png)

如果是云服务器的话，那么就要在安全组那边将这几台的端口都开放:

![1](2.png)

### 2. 禁用SELinux
```text
[root@worker-master ~]# setenforce 0
setenforce: SELinux is disabled
[root@worker-master ~]# sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

### 3. 关闭Swap
```text
[root@worker-master ~]# swapoff -a
```
### 4. CentOS 7 用户还需要执行：
```text
[root@worker-master ~]# cat <<EOF >  /etc/sysctl.d/k8s.conf
> net.bridge.bridge-nf-call-ip6tables = 1
> net.bridge.bridge-nf-call-iptables = 1
> EOF
[root@worker-master ~]# sysctl --system
```
### 5. /etc/hosts 文件增加
```text
172.26.64.2 worker-1
172.26.64.10 worker-2
172.26.64.9 worker-master
```

<font color=red><b>除非另有说明, 以下所有操作都是在 master 上执行</b></font>

## 1. 安装必要工具
接下来在 master 上安装以下必要工具
- [cfssl](https://github.com/cloudflare/cfssl)
- [cfssljson](https://github.com/cloudflare/cfssl)
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

### 1.1 安装 cfssl
cfssl 和 cfssljson 命令行工具用于提供 PKI Infrastructure 基础设施与生成 TLS 证书。

指令:
```text
wget --timestamping \
  https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 \
  https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
chmod +x cfssl_linux-amd64 cfssljson_linux-amd64
sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl
sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
```

执行
```text
[root@worker-master ~]# wget --timestamping \
>   https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 \
>   https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
...
Downloaded: 2 files, 12M in 0.4s (32.3 MB/s)
[root@worker-master ~]# chmod +x cfssl_linux-amd64 cfssljson_linux-amd64
[root@worker-master ~]# sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl
[root@worker-master ~]# sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
```
验证一下:
```text
[root@worker-master ~]# cfssl version
Version: 1.2.0
Revision: dev
Runtime: go1.6
```

### 1.2 安装 kubectl
kubectl 命令行工具用来与 Kubernetes API Server 交互，可以在 Kubernetes 官方网站下载并安装 kubectl。

指令:
```text
wget https://storage.googleapis.com/kubernetes-release/release/v1.15.0/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

执行:
```text
[root@worker-master ~]# wget https://storage.googleapis.com/kubernetes-release/release/v1.15.0/bin/linux/amd64/kubectl
...
2020-10-29 16:43:25 (44.4 MB/s) - ‘kubectl’ saved [42985504/42985504]

[root@worker-master ~]# chmod +x kubectl
[root@worker-master ~]# sudo mv kubectl /usr/local/bin/
[root@worker-master ~]# kubectl version --client
Client Version: version.Info{Major:"1", Minor:"15", GitVersion:"v1.15.0", GitCommit:"e8462b5b5dc2584fdcd18e6bcfe9f1e4d970a529", GitTreeState:"clean", BuildDate:"2019-06-19T16:40:16Z", GoVersion:"go1.12.5", Compiler:"gc", Platform:"linux/amd64"}
```

## 2. 配置 CA 并创建 TLS 证书
我们将使用 CloudFlare's PKI 工具 cfssl 来配置 PKI Infrastructure，然后使用它去创建 Certificate Authority（CA），并为 etcd、kube-apiserver、kubelet 以及 kube-proxy 创建 TLS 证书。

### 2.1 Certificate Authority
创建用于生成其他 TLS 证书的 Certificate Authority。
#### 2.1.1 新建 CA 配置文件
```text
[root@worker-master ~]# cat > ca-config.json <<EOF
> {
>   "signing": {
>     "default": {
>       "expiry": "8760h"
>     },
>     "profiles": {
>       "kubernetes": {
>         "usages": ["signing", "key encipherment", "server auth", "client auth"],
>         "expiry": "8760h"
>       }
>     }
>   }
> }
> EOF
```
#### 2.1.2 新建 CA 凭证签发请求文件
```text
[root@worker-master ~]# cat > ca-csr.json <<EOF
> {
>   "CN": "Kubernetes",
>   "key": {
>     "algo": "rsa",
>     "size": 2048
>   },
>   "names": [
>     {
>       "C": "China",
>       "L": "Shanghai",
>       "O": "Kubernetes",
>       "OU": "Shanghai",
>       "ST": "Shanghai"
>     }
>   ]
> }
> EOF
```
#### 2.1.3 生成 CA 凭证和私钥
```text
[root@worker-master ~]# cfssl gencert -initca ca-csr.json | cfssljson -bare ca
2020/10/29 16:46:45 [INFO] generating a new CA key and certificate from CSR
2020/10/29 16:46:45 [INFO] generate received request
2020/10/29 16:46:45 [INFO] received CSR
2020/10/29 16:46:45 [INFO] generating key: rsa-2048
2020/10/29 16:46:45 [INFO] encoded CSR
2020/10/29 16:46:45 [INFO] signed certificate with serial number 313288276491452486871986487059615120584428274805
```

最后生成这两个文件:
```text
ca-key.pem
ca.pem
```

### 2.2 client 与 server 凭证
本节将创建用于 Kubernetes 组件的 client 与 server 凭证，以及一个用于 Kubernetes admin 用户的 client 凭证。
#### 2.2.1 Admin 客户端凭证
创建 admin client 凭证签发请求文件:
```text
[root@worker-master ~]# cat > admin-csr.json <<EOF
> {
>   "CN": "admin",
>   "key": {
>     "algo": "rsa",
>     "size": 2048
>   },
>   "names": [
>     {
>       "C": "China",
>       "L": "Shanghai",
>       "O": "system:masters",
>       "OU": "Kubernetes",
>       "ST": "Shanghai"
>     }
>   ]
> }
> EOF
```
创建 admin client 凭证和私钥:
```text
[root@worker-master ~]# cfssl gencert \
>   -ca=ca.pem \
>   -ca-key=ca-key.pem \
>   -config=ca-config.json \
>   -profile=kubernetes \
>   admin-csr.json | cfssljson -bare admin
...
specifically, section 10.2.3 ("Information Requirements").
```
结果将生成以下两个文件:
```text
admin-key.pem
admin.pem
```
#### 2.2.2 Kubelet 客户端凭证
Kubernetes 使用 special-purpose authorization mode（被称作 Node Authorizer）授权来自 Kubelet 的 API 请求。

为了通过 Node Authorizer 的授权, Kubelet 必须使用一个署名为 system:node:<nodeName> 的凭证来证明它属于 system:nodes 用户组。本节将会给每台 worker 节点创建符合 Node Authorizer 要求的凭证。

我们有两台 worker， 分别是 `worker-1` 和 `worker-2`。 接下来先为 `worker-1` 节点创建凭证和私钥 (依然是在 master 上操作)。

##### 2.2.2.1 为 worker-1 节点创建凭证和私钥
```text
# 名字就是 worker-1， 下面的 ip 就是 worker-1 的内网 ip
[root@worker-master ~]# export WORKER1_HOSTNAME=worker-1
[root@worker-master ~]# export WORKER1_IP=172.26.64.2
[root@worker-master ~]# cat > worker-1-csr.json <<EOF
> {
>   "CN": "system:node:${WORKER1_HOSTNAME}",
>   "key": {
>     "algo": "rsa",
>     "size": 2048
>   },
>   "names": [
>     {
>       "C": "China",
>       "L": "Shanghai",
>       "O": "system:nodes",
>       "OU": "Kubernetes",
>       "ST": "Shanghai"
>     }
>   ]
> }
> EOF
[root@worker-master ~]# cfssl gencert \
>   -ca=ca.pem \
>   -ca-key=ca-key.pem \
>   -config=ca-config.json \
>   -hostname=${WORKER1_HOSTNAME},${WORKER1_IP} \
>   -profile=kubernetes \
>   worker-1-csr.json | cfssljson -bare worker-1
...
2020/10/29 16:53:36 [INFO] signed certificate with serial number 531259582571192075561426574394248110542500079355
[root@worker-master ~]# ll | grep 'worker-1'
-rw-r--r-- 1 root root 1041 Oct 29 16:53 worker-1.csr
-rw-r--r-- 1 root root  236 Oct 29 16:53 worker-1-csr.json
-rw------- 1 root root 1679 Oct 29 16:53 worker-1-key.pem
-rw-r--r-- 1 root root 1489 Oct 29 16:53 worker-1.pem
```
主要是为了产生这两个文件:
```text
worker-1-key.pem
worker-1.pem
```
##### 2.2.2.2 为 worker-2 节点创建凭证和私钥
同样的方式，为 worker-2 再处理一下
```text

[root@worker-master ~]# export WORKER2_HOSTNAME=worker-2
[root@worker-master ~]# export WORKER2_IP=172.26.64.10
[root@worker-master ~]# cat > worker-2-csr.json <<EOF
> {
>   "CN": "system:node:${WORKER2_HOSTNAME}",
>   "key": {
>     "algo": "rsa",
>     "size": 2048
>   },
>   "names": [
>     {
>       "C": "China",
>       "L": "Shanghai",
>       "O": "system:nodes",
>       "OU": "Kubernetes",
>       "ST": "Shanghai"
>     }
>   ]
> }
> EOF
[root@worker-master ~]# cfssl gencert \
>   -ca=ca.pem \
>   -ca-key=ca-key.pem \
>   -config=ca-config.json \
>   -hostname=${WORKER2_HOSTNAME},${WORKER2_IP} \
>   -profile=kubernetes \
>   worker-2-csr.json | cfssljson -bare worker-2
...
2020/10/29 16:56:21 [INFO] signed certificate with serial number 688958995550700561005532700973761497460653349926
[root@worker-master ~]# ll | grep 'worker-2'
-rw-r--r-- 1 root root 1041 Oct 29 16:56 worker-2.csr
-rw-r--r-- 1 root root  236 Oct 29 16:55 worker-2-csr.json
-rw------- 1 root root 1679 Oct 29 16:56 worker-2-key.pem
-rw-r--r-- 1 root root 1489 Oct 29 16:56 worker-2.pem
```
主要是为了产生这两个文件:
```text
worker-2-key.pem
worker-2.pem
```

#### 2.2.3 Kube-controller-manager 客户端凭证
```text
[root@worker-master ~]# cat > kube-controller-manager-csr.json <<EOF
> {
>   "CN": "system:kube-controller-manager",
>   "key": {
>     "algo": "rsa",
>     "size": 2048
>   },
>   "names": [
>     {
>       "C": "China",
>       "L": "Shanghai",
>       "O": "system:kube-controller-manager",
>       "OU": "Kubernetes",
>       "ST": "Shanghai"
>     }
>   ]
> }
> EOF
[root@worker-master ~]# cfssl gencert \
>   -ca=ca.pem \
>   -ca-key=ca-key.pem \
>   -config=ca-config.json \
>   -profile=kubernetes \
>   kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
...
specifically, section 10.2.3 ("Information Requirements").
[root@worker-master ~]# ll | grep 'kube-controller-manager'
-rw-r--r-- 1 root root 1078 Oct 29 16:58 kube-controller-manager.csr
-rw-r--r-- 1 root root  264 Oct 29 16:57 kube-controller-manager-csr.json
-rw------- 1 root root 1675 Oct 29 16:58 kube-controller-manager-key.pem
-rw-r--r-- 1 root root 1489 Oct 29 16:58 kube-controller-manager.pem
```
结果将产生以下几个文件：
```text
kube-controller-manager-key.pem
kube-controller-manager.pem
```

#### 2.2.4 Kube-proxy 客户端凭证
```text
[root@worker-master ~]# cat > kube-proxy-csr.json <<EOF
> {
>   "CN": "system:kube-proxy",
>   "key": {
>     "algo": "rsa",
>     "size": 2048
>   },
>   "names": [
>     {
>       "C": "China",
>       "L": "Shanghai",
>       "O": "system:node-proxier",
>       "OU": "Kubernetes",
>       "ST": "Shanghai"
>     }
>   ]
> }
> EOF
[root@worker-master ~]# cfssl gencert \
>   -ca=ca.pem \
>   -ca-key=ca-key.pem \
>   -config=ca-config.json \
>   -profile=kubernetes \
>   kube-proxy-csr.json | cfssljson -bare kube-proxy
...
specifically, section 10.2.3 ("Information Requirements").
[root@worker-master ~]# ll | grep 'kube-proxy'
-rw-r--r-- 1 root root 1045 Oct 29 16:59 kube-proxy.csr
-rw-r--r-- 1 root root  240 Oct 29 16:59 kube-proxy-csr.json
-rw------- 1 root root 1675 Oct 29 16:59 kube-proxy-key.pem
-rw-r--r-- 1 root root 1456 Oct 29 16:59 kube-proxy.pem
```
结果将产生以下两个文件：
```text
kube-proxy-key.pem
kube-proxy.pem
```

#### 2.2.5 kube-scheduler 证书
```text
[root@worker-master ~]# cat > kube-scheduler-csr.json <<EOF
> {
>   "CN": "system:kube-scheduler",
>   "key": {
>     "algo": "rsa",
>     "size": 2048
>   },
>   "names": [
>     {
>       "C": "China",
>       "L": "Shanghai",
>       "O": "system:kube-scheduler",
>       "OU": "Kubernetes",
>       "ST": "Shanghai"
>     }
>   ]
> }
> EOF
[root@worker-master ~]# cfssl gencert \
>   -ca=ca.pem \
>   -ca-key=ca-key.pem \
>   -config=ca-config.json \
>   -profile=kubernetes \
>   kube-scheduler-csr.json | cfssljson -bare kube-scheduler
...
specifically, section 10.2.3 ("Information Requirements").
[root@worker-master ~]# ll | grep kube-scheduler
-rw-r--r-- 1 root root 1054 Oct 29 17:00 kube-scheduler.csr
-rw-r--r-- 1 root root  246 Oct 29 17:00 kube-scheduler-csr.json
-rw------- 1 root root 1679 Oct 29 17:00 kube-scheduler-key.pem
-rw-r--r-- 1 root root 1464 Oct 29 17:00 kube-scheduler.pem
```
结果将产生以下两个文件：
```text
kube-scheduler-key.pem
kube-scheduler.pem
```

#### 2.2.6 Kubernetes API Server 证书
为了保证客户端与 Kubernetes API 的认证，Kubernetes API Server 凭证中必需包含 master 的静态 IP 地址。
##### 2.2.6.1 创建 Kubernetes API Server 凭证签发请求文件
```text
[root@worker-master ~]# cat > kubernetes-csr.json <<EOF
> {
>   "CN": "kubernetes",
>   "key": {
>     "algo": "rsa",
>     "size": 2048
>   },
>   "names": [
>     {
>       "C": "China",
>       "L": "Shanghai",
>       "O": "Kubernetes",
>       "OU": "Kubernetes",
>       "ST": "Shanghai"
>     }
>   ]
> }
> EOF
```
##### 2.2.6.2 创建 Kubernetes API Server 凭证与私钥
```text
[root@worker-master ~]# export MASTER_IP=172.26.64.9
[root@worker-master ~]# cfssl gencert \
>   -ca=ca.pem \
>   -ca-key=ca-key.pem \
>   -config=ca-config.json \
>   -hostname=10.250.0.1,${MASTER_IP},127.0.0.1,kubernetes.default \
>   -profile=kubernetes \
>   kubernetes-csr.json | cfssljson -bare kubernetes
...
2020/10/29 17:02:51 [INFO] signed certificate with serial number 499334605103379856047696211541729253687736350720
[root@worker-master ~]# ll | grep kubernetes
-rw-r--r-- 1 root root 1021 Oct 29 17:02 kubernetes.csr
-rw-r--r-- 1 root root  224 Oct 29 17:01 kubernetes-csr.json
-rw------- 1 root root 1675 Oct 29 17:02 kubernetes-key.pem
-rw-r--r-- 1 root root 1501 Oct 29 17:02 kubernetes.pem
```
结果产生以下两个文件:
```text
kubernetes-key.pem
kubernetes.pem
```

#### 2.2.7 Service Account 证书
```text
[root@worker-master ~]# cat > service-account-csr.json <<EOF
> {
>   "CN": "service-accounts",
>   "key": {
>     "algo": "rsa",
>     "size": 2048
>   },
>   "names": [
>     {
>       "C": "China",
>       "L": "Shanghai",
>       "O": "Kubernetes",
>       "OU": "Kubernetes",
>       "ST": "Shanghai"
>     }
>   ]
> }
> EOF
[root@worker-master ~]# cfssl gencert \
>   -ca=ca.pem \
>   -ca-key=ca-key.pem \
>   -config=ca-config.json \
>   -profile=kubernetes \
>   service-account-csr.json | cfssljson -bare service-account
...
specifically, section 10.2.3 ("Information Requirements").
[root@worker-master ~]# ll | grep service-account
-rw-r--r-- 1 root root 1029 Oct 29 17:04 service-account.csr
-rw-r--r-- 1 root root  230 Oct 29 17:03 service-account-csr.json
-rw------- 1 root root 1679 Oct 29 17:04 service-account-key.pem
-rw-r--r-- 1 root root 1440 Oct 29 17:04 service-account.pem
```
结果将生成以下两个文件
```text
service-account-key.pem
service-account.pem
```

### 2.3 分发客户端和服务器证书
接下来将客户端凭证以及私钥复制到每个 worker 节点上，因为我们有两个 worker 节点， 所以两个都要处理
```text
[root@worker-master ~]# scp -i /root/k8s  ca.pem worker-1-key.pem worker-1.pem root@172.26.64.2:~/
ca.pem                                                                                                                                                                        100% 1395     4.3MB/s   00:00    
worker-1-key.pem                                                                                                                                                              100% 1679     5.4MB/s   00:00    
worker-1.pem                                                                                                                                                                  100% 1489     5.2MB/s   00:00    
```
```text
[root@worker-master ~]# scp -i /root/k8s  ca.pem worker-2-key.pem worker-2.pem root@172.26.64.10:~/
ca.pem                                                                                                                                                                        100% 1395     3.6MB/s   00:00    
worker-2-key.pem                                                                                                                                                              100% 1679     4.3MB/s   00:00    
worker-2.pem                                                                                                                                                                  100% 1489     4.7MB/s   00:00    
```
不过要注意一点， 要用内网地址， 因为我的云服务器，只有内网互通，不会走公网，不然还得开22 端口权限。

这样子这些对应的文件都放到 worker 节点了。

教程中还有一个细节就是要将服务器凭证以及私钥复制到每个控制节点上:
```text
scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem master:~/
```
不过我们本来就是在 master 上操作，所以这一步就不要处理了。

## 3. 配置和生成 Kubernetes 配置文件
本部分内容将会创建 kubeconfig 配置文件，它们是 Kubernetes 客户端与 API Server 认证与鉴权的保证。

本节将会创建用于 kube-proxy、kube-controller-manager、kube-scheduler 和 kubelet 的 kubeconfig 文件。

### 3.1 kubelet 配置文件
为了确保 Node Authorizer 授权，Kubelet 配置文件中的客户端证书必需匹配 Node 名字。 所以要为每个 worker 节点创建 kubeconfig 配置，所以要一台一台来, 先为 worker-1 先来:
#### 3.1.1 worker-1
```text
[root@worker-master ~]# kubectl config set-cluster kubernetes-training \
>   --certificate-authority=ca.pem \
>   --embed-certs=true \
>   --server=https://${MASTER_IP}:6443 \
>   --kubeconfig=worker-1.kubeconfig
Cluster "kubernetes-training" set.
[root@worker-master ~]# kubectl config set-credentials system:node:worker-1 \
>   --client-certificate=worker-1.pem \
>   --client-key=worker-1-key.pem \
>   --embed-certs=true \
>   --kubeconfig=worker-1.kubeconfig
User "system:node:worker-1" set.
[root@worker-master ~]# kubectl config set-context default \
>   --cluster=kubernetes-training \
>   --user=system:node:worker-1 \
>   --kubeconfig=worker-1.kubeconfig
Context "default" created.
[root@worker-master ~]# kubectl config use-context default --kubeconfig=worker-1.kubeconfig
Switched to context "default".
[root@worker-master ~]# ll | grep kubeconfig
-rw------- 1 root root 6473 Oct 29 17:45 worker-1.kubeconfig
```
结果将会生成以下文件：
```text
worker-1.kubeconfig
```

#### 3.1.1 worker-2
```text
[root@worker-master ~]# kubectl config set-cluster kubernetes-training \
>   --certificate-authority=ca.pem \
>   --embed-certs=true \
>   --server=https://${MASTER_IP}:6443 \
>   --kubeconfig=worker-2.kubeconfig
Cluster "kubernetes-training" set.
[root@worker-master ~]# kubectl config set-credentials system:node:worker-2 \
>   --client-certificate=worker-2.pem \
>   --client-key=worker-2-key.pem \
>   --embed-certs=true \
>   --kubeconfig=worker-2.kubeconfig
User "system:node:worker-2" set.
[root@worker-master ~]# kubectl config set-context default \
>   --cluster=kubernetes-training \
>   --user=system:node:worker-2 \
>   --kubeconfig=worker-2.kubeconfig
Context "default" created.
[root@worker-master ~]# kubectl config use-context default --kubeconfig=worker-2.kubeconfig
Switched to context "default".
[root@worker-master ~]# ll | grep kubeconfig
-rw------- 1 root root 6365 Oct 29 17:49 admin.kubeconfig
-rw------- 1 root root 6487 Oct 29 17:47 kube-controller-manager.kubeconfig
-rw------- 1 root root 6419 Oct 29 17:46 kube-proxy.kubeconfig
-rw------- 1 root root 6437 Oct 29 17:48 kube-scheduler.kubeconfig
-rw------- 1 root root 6473 Oct 29 17:45 worker-1.kubeconfig
-rw------- 1 root root 6473 Oct 29 17:55 worker-2.kubeconfig
```
结果将会生成以下文件：
```text
worker-2.kubeconfig
```
### 3.2 kube-proxy 配置文件
为 kube-proxy 服务生成 kubeconfig 配置文件：
```text
[root@worker-master ~]# kubectl config set-cluster kubernetes-training \
>   --certificate-authority=ca.pem \
>   --embed-certs=true \
>   --server=https://${MASTER_IP}:6443 \
>   --kubeconfig=kube-proxy.kubeconfig
Cluster "kubernetes-training" set.
[root@worker-master ~]# kubectl config set-credentials system:kube-proxy \
>   --client-certificate=kube-proxy.pem \
>   --client-key=kube-proxy-key.pem \
>   --embed-certs=true \
>   --kubeconfig=kube-proxy.kubeconfig
User "system:kube-proxy" set.
[root@worker-master ~]# kubectl config set-context default \
>   --cluster=kubernetes-training \
>   --user=system:kube-proxy \
>   --kubeconfig=kube-proxy.kubeconfig
Context "default" created.
[root@worker-master ~]# kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
Switched to context "default".
```
### 3.3 kube-controller-manager 配置文件
```text
[root@worker-master ~]# kubectl config set-cluster kubernetes-training \
>   --certificate-authority=ca.pem \
>   --embed-certs=true \
>   --server=https://127.0.0.1:6443 \
>   --kubeconfig=kube-controller-manager.kubeconfig
Cluster "kubernetes-training" set.
[root@worker-master ~]# kubectl config set-credentials system:kube-controller-manager \
>   --client-certificate=kube-controller-manager.pem \
>   --client-key=kube-controller-manager-key.pem \
>   --embed-certs=true \
>   --kubeconfig=kube-controller-manager.kubeconfig
User "system:kube-controller-manager" set.
[root@worker-master ~]# kubectl config set-context default \
>   --cluster=kubernetes-training \
>   --user=system:kube-controller-manager \
>   --kubeconfig=kube-controller-manager.kubeconfig
Context "default" created.
[root@worker-master ~]# kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
Switched to context "default".
```
### 3.4 kube-scheduler 配置文件
```text
[root@worker-master ~]# kubectl config set-cluster kubernetes-training \
>   --certificate-authority=ca.pem \
>   --embed-certs=true \
>   --server=https://127.0.0.1:6443 \
>   --kubeconfig=kube-scheduler.kubeconfig
Cluster "kubernetes-training" set.
[root@worker-master ~]# kubectl config set-credentials system:kube-scheduler \
>   --client-certificate=kube-scheduler.pem \
>   --client-key=kube-scheduler-key.pem \
>   --embed-certs=true \
>   --kubeconfig=kube-scheduler.kubeconfig
User "system:kube-scheduler" set.
[root@worker-master ~]# kubectl config set-context default \
>   --cluster=kubernetes-training \
>   --user=system:kube-scheduler \
>   --kubeconfig=kube-scheduler.kubeconfig
Context "default" created.
[root@worker-master ~]# kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
Switched to context "default".
```
### 3.5 Admin 配置文件
```text
[root@worker-master ~]# kubectl config set-cluster kubernetes-training \
>   --certificate-authority=ca.pem \
>   --embed-certs=true \
>   --server=https://127.0.0.1:6443 \
>   --kubeconfig=admin.kubeconfig
Cluster "kubernetes-training" set.
[root@worker-master ~]#
[root@worker-master ~]# kubectl config set-credentials admin \
>   --client-certificate=admin.pem \
>   --client-key=admin-key.pem \
>   --embed-certs=true \
>   --kubeconfig=admin.kubeconfig
User "admin" set.
[root@worker-master ~]#
[root@worker-master ~]# kubectl config set-context default \
>   --cluster=kubernetes-training \
>   --user=admin \
>   --kubeconfig=admin.kubeconfig
Context "default" created.
[root@worker-master ~]# kubectl config use-context default --kubeconfig=admin.kubeconfig
Switched to context "default".
```
### 3.6 分发配置文件
将 kubelet 与 kube-proxy kubeconfig 配置文件复制到每个 worker 节点上：
```text
[root@worker-master ~]# scp -i /root/k8s worker-1.kubeconfig kube-proxy.kubeconfig  root@172.26.64.2:~/
worker-1.kubeconfig                                                                                                                                                           100% 6473    14.4MB/s   00:00    
kube-proxy.kubeconfig                                                                                                                                                         100% 6419    14.3MB/s   00:00
```
```text
[root@worker-master ~]# scp -i /root/k8s  worker-2.kubeconfig kube-proxy.kubeconfig root@172.26.64.10:~/
worker-2.kubeconfig                                                                                                                                                           100% 6473    12.0MB/s   00:00    
kube-proxy.kubeconfig                                                                                                                                                         100% 6419    14.2MB/s   00:00    
```
本来还有一步就是，将 admin、kube-controller-manager 与 kube-scheduler kubeconfig 配置文件复制到每个控制节点上：
```text
scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig master:~/
```
不过这个本来就是在 master 上了，所以不用执行了。

## 4. 配置和生成密钥
Kubernetes 存储了集群状态、应用配置和密钥等很多不同的数据。而 Kubernetes 也支持集群数据的加密存储。

本部分将会创建加密密钥以及一个用于加密 Kubernetes Secrets 的 加密配置文件。

### 4.1 建立加密密钥
```text
[root@worker-master ~]# ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

### 4.2 加密配置文件
生成名为 encryption-config.yaml 的加密配置文件：
```text
[root@worker-master ~]# cat > encryption-config.yaml <<EOF
> kind: EncryptionConfig
> apiVersion: v1
> resources:
>   - resources:
>       - secrets
>     providers:
>       - aescbc:
>           keys:
>             - name: key1
>               secret: ${ENCRYPTION_KEY}
>       - identity: {}
> EOF
```
接下来还有一步就是将 encryption-config.yaml 复制到每个控制节点上：
```text
scp encryption-config.yaml master:~/
```
不过本来我们就是在 master 上了，所以就不需要了。

## 5. 部署 etcd 群集
Kubernetes 组件都是无状态的，所有的群集状态都储存在 etcd 集群中。

### 5.1 下载并安装 etcd 二进制文件
从 [coreos/etcd](https://github.com/etcd-io/etcd) GitHub 中下载 etcd 发布文件：
```text
[root@worker-master ~]# wget --timestamping \
>   "https://github.com/coreos/etcd/releases/download/v3.3.9/etcd-v3.3.9-linux-amd64.tar.gz"
```
解压缩并安装 etcd 服务与 etcdctl 命令行工具：
```text
[root@worker-master ~]# tar -xvf etcd-v3.3.9-linux-amd64.tar.gz
```
```text
[root@worker-master ~]# sudo mv etcd-v3.3.9-linux-amd64/etcd* /usr/local/bin/
```

### 5.2 配置 etcd Server
```text
[root@worker-master ~]# sudo mkdir -p /etc/etcd /var/lib/etcd
[root@worker-master ~]# sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
```
每个 etcd 成员必须有一个整集群中唯一的名字，使用 hostname 作为 etcd name：
```text
[root@worker-master ~]# ETCD_NAME=$(hostname -s)
```
生成 etcd.service 的 systemd 配置文件:
> 请注意：对于使用公网主机（如云服务器等）的操作者， etcd 绑定的服务端口（即以下的 ${MASTER_IP} ）应使用内网 IP 地址。

```text
[root@worker-master ~]# cat <<EOF | sudo tee /etc/systemd/system/etcd.service
> [Unit]
> Description=etcd
> Documentation=https://github.com/coreos
> [Service]
> ExecStart=/usr/local/bin/etcd \\
>   --name master \\
>   --cert-file=/etc/etcd/kubernetes.pem \\
>   --key-file=/etc/etcd/kubernetes-key.pem \\
>   --peer-cert-file=/etc/etcd/kubernetes.pem \\
>   --peer-key-file=/etc/etcd/kubernetes-key.pem \\
>   --trusted-ca-file=/etc/etcd/ca.pem \\
>   --peer-trusted-ca-file=/etc/etcd/ca.pem \\
>   --peer-client-cert-auth \\
>   --client-cert-auth \\
>   --initial-advertise-peer-urls https://${MASTER_IP}:2380 \\
>   --listen-peer-urls https://${MASTER_IP}:2380 \\
>   --listen-client-urls https://${MASTER_IP}:2379,http://127.0.0.1:2379 \\
>   --advertise-client-urls https://${MASTER_IP}:2379,http://127.0.0.1:2379 \\
>   --data-dir=/var/lib/etcd
> Restart=on-failure
> RestartSec=5
> [Install]
> WantedBy=multi-user.target
> EOF
```
### 5.3 启动 etcd Server
```text
[root@worker-master ~]# sudo systemctl daemon-reload
[root@worker-master ~]# sudo systemctl enable etcd
Created symlink from /etc/systemd/system/multi-user.target.wants/etcd.service to /etc/systemd/system/etcd.service.
[root@worker-master ~]# sudo systemctl start etcd
```
### 5.4 验证是否正常
```text
[root@worker-master ~]# ETCDCTL_API=3 etcdctl member list
6911926ad78226a5, started, master, https://172.26.64.9:2380, http://127.0.0.1:2379,https://172.26.64.9:2379
```
正常。

## 6. 部署 Kubernetes 控制节点
本部分将会在控制节点上部署 Kubernetes 控制服务。每个控制节点上需要部署的服务包括：Kubernetes API Server、Scheduler 以及 Controller Manager 等。

### 6.1 创建 Kubernetes 配置目录
```text
[root@worker-master ~]# sudo mkdir -p /etc/kubernetes/config
```
### 6.2 下载并安装 Kubernetes Controller 二进制文件
```text
[root@worker-master ~]# wget --timestamping \
>   "https://storage.googleapis.com/kubernetes-release/release/v1.15.0/bin/linux/amd64/kube-apiserver" \
>   "https://storage.googleapis.com/kubernetes-release/release/v1.15.0/bin/linux/amd64/kube-controller-manager" \
>   "https://storage.googleapis.com/kubernetes-release/release/v1.15.0/bin/linux/amd64/kube-scheduler" \
>   "https://storage.googleapis.com/kubernetes-release/release/v1.15.0/bin/linux/amd64/kubectl"
...
Downloaded: 4 files, 346M in 9.5s (36.2 MB/s)
[root@worker-master ~]# chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
[root@worker-master ~]# sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
```
### 6.3 配置 Kubernetes API Server
```text
[root@worker-master ~]# sudo mkdir -p /var/lib/kubernetes/
[root@worker-master ~]# sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
>   service-account-key.pem service-account.pem \
>   encryption-config.yaml /var/lib/kubernetes/
```
生成 kube-apiserver.service systemd 配置文件：
```text
[root@worker-master ~]# cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
> [Unit]
> Description=Kubernetes API Server
> Documentation=https://github.com/kubernetes/kubernetes
>
> [Service]
> ExecStart=/usr/local/bin/kube-apiserver \\
>   --advertise-address=${MASTER_IP} \\
>   --allow-privileged=true \\
>   --audit-log-maxage=30 \\
>   --audit-log-maxbackup=3 \\
>   --audit-log-maxsize=100 \\
>   --audit-log-path=/var/log/audit.log \\
>   --authorization-mode=Node,RBAC \\
>   --bind-address=0.0.0.0 \\
>   --client-ca-file=/var/lib/kubernetes/ca.pem \\
>   --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota \\
>   --enable-swagger-ui=true \\
>   --etcd-cafile=/var/lib/kubernetes/ca.pem \\
>   --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
>   --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
>   --etcd-servers=http://127.0.0.1:2379 \\
>   --event-ttl=1h \\
>   --experimental-encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
>   --insecure-bind-address=127.0.0.1 \\
>   --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
>   --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
>   --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
>   --kubelet-https=true \\
>   --runtime-config=api/all \\
>   --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
>   --service-cluster-ip-range=10.250.0.0/24 \\
>   --service-node-port-range=30000-32767 \\
>   --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
>   --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
>   --v=2
> Restart=on-failure
> RestartSec=5
>
> [Install]
> WantedBy=multi-user.target
> EOF
```

### 6.4 配置 Kubernetes Controller Manager
生成 kube-controller-manager.service systemd 配置文件：
```text
[root@worker-master ~]# sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
[root@worker-master ~]# cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
> [Unit]
> Description=Kubernetes Controller Manager
> Documentation=https://github.com/kubernetes/kubernetes
>
> [Service]
> ExecStart=/usr/local/bin/kube-controller-manager \\
>   --address=0.0.0.0 \\
>   --allocate-node-cidrs=true \\
>   --cluster-cidr=10.244.0.0/16 \\
>   --cluster-name=kubernetes \\
>   --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
>   --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
>   --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
>   --leader-elect=true \\
>   --root-ca-file=/var/lib/kubernetes/ca.pem \\
>   --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
>   --service-cluster-ip-range=10.250.0.0/24 \\
>   --use-service-account-credentials=true \\
>   --v=2
> Restart=on-failure
> RestartSec=5
>
> [Install]
> WantedBy=multi-user.target
> EOF
```
### 6.5 配置 Kubernetes Scheduler
生成 kube-scheduler.service systemd 配置文件：
```text
[root@worker-master ~]# sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
[root@worker-master ~]# cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
> apiVersion: kubescheduler.config.k8s.io/v1alpha1
> kind: KubeSchedulerConfiguration
> clientConnection:
>   kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
> leaderElection:
>   leaderElect: true
> EOF
[root@worker-master ~]# cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
> [Unit]
> Description=Kubernetes Scheduler
> Documentation=https://github.com/kubernetes/kubernetes
>
> [Service]
> ExecStart=/usr/local/bin/kube-scheduler \\
>   --config=/etc/kubernetes/config/kube-scheduler.yaml \\
>   --v=2
> Restart=on-failure
> RestartSec=5
>
> [Install]
> WantedBy=multi-user.target
> EOF
```
### 6.6 启动控制器服务
```text
[root@worker-master ~]# sudo systemctl daemon-reload
[root@worker-master ~]# sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
Created symlink from /etc/systemd/system/multi-user.target.wants/kube-apiserver.service to /etc/systemd/system/kube-apiserver.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/kube-controller-manager.service to /etc/systemd/system/kube-controller-manager.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/kube-scheduler.service to /etc/systemd/system/kube-scheduler.service.
[root@worker-master ~]# sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
```
请等待 10 秒以便 Kubernetes API Server 初始化。 接下来验证一下

```text
[root@worker-master ~]# kubectl get componentstatuses --kubeconfig admin.kubeconfig
NAME                 STATUS    MESSAGE             ERROR
etcd-0               Healthy   {"health":"true"}   
scheduler            Healthy   ok                  
controller-manager   Healthy   ok      
```

说明正常。

## 7. Kubelet RBAC 授权
本节将会配置 API Server 访问 Kubelet API 的 RBAC 授权。访问 Kubelet API 是获取 metrics、日志以及执行容器命令所必需的。
> 这里设置 Kubelet --authorization-mode 为 Webhook 模式。Webhook 模式使用 SubjectAccessReview API 来决定授权。

创建 system:kube-apiserver-to-kubelet ClusterRole 以允许请求 Kubelet API 和执行大部分来管理 Pods 的任务:
```text
[root@worker-master ~]# cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
> apiVersion: rbac.authorization.k8s.io/v1beta1
> kind: ClusterRole
> metadata:
>   annotations:
>     rbac.authorization.kubernetes.io/autoupdate: "true"
>   labels:
>     kubernetes.io/bootstrapping: rbac-defaults
>   name: system:kube-apiserver-to-kubelet
> rules:
>   - apiGroups:
>       - ""
>     resources:
>       - nodes/proxy
>       - nodes/stats
>       - nodes/log
>       - nodes/spec
>       - nodes/metrics
>     verbs:
>       - "*"
> EOF
clusterrole.rbac.authorization.k8s.io/system:kube-apiserver-to-kubelet created
```
Kubernetes API Server 使用客户端凭证授权 Kubelet 为 `kubernetes` 用户，此凭证用 `--kubelet-client-certificate` flag 来定义。

绑定 `system:kube-apiserver-to-kubelet` ClusterRole 到 `kubernetes` 用户:
```text
[root@worker-master ~]# cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
> apiVersion: rbac.authorization.k8s.io/v1beta1
> kind: ClusterRoleBinding
> metadata:
>   name: system:kube-apiserver
>   namespace: ""
> roleRef:
>   apiGroup: rbac.authorization.k8s.io
>   kind: ClusterRole
>   name: system:kube-apiserver-to-kubelet
> subjects:
>   - apiGroup: rbac.authorization.k8s.io
>     kind: User
>     name: kubernetes
> EOF
clusterrolebinding.rbac.authorization.k8s.io/system:kube-apiserver created
```

## 8. 部署 Kubernetes Workers 节点
本部分将会部署 Kubernetes Worker 节点 (上面的 1-7 步骤都是在 master 上操作的)。每个节点上将会安装以下服务：
- [Docker](https://www.docker.com/)
- [container networking plugins](https://github.com/containernetworking/cni)
- [kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/)
- [kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies/)

因为我们有两个 worker 节点， 接下来先部署 worker-1 节点，等加入集群成功了，并且验证没问题之后， 再部署 worker-2 节点。

所以接下来的操作都是在 worker-1 节点上操作 (通用配置要先配置)的。

### 8.1 安装依赖
安装 OS 依赖组件：
```text
[root@worker-1 ~]# sudo yum install -y socat conntrack ipset
```
安装 docker:
```text
[root@worker-1 ~]# sudo yum install -y yum-utils device-mapper-persistent-data lvm2
[root@worker-1 ~]# sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
[root@worker-1 ~]# sudo yum install -y docker-ce docker-ce-cli containerd.io
[root@worker-1 ~]# sudo systemctl enable docker
Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.
[root@worker-1 ~]# sudo systemctl start docker
```
下载 worker 二进制文件：
```text
[root@worker-1 ~]# wget --timestamping \
>   https://github.com/containernetworking/plugins/releases/download/v0.6.0/cni-plugins-amd64-v0.6.0.tgz \
>   https://storage.googleapis.com/kubernetes-release/release/v1.15.0/bin/linux/amd64/kubectl \
>   https://storage.googleapis.com/kubernetes-release/release/v1.15.0/bin/linux/amd64/kube-proxy \
>   https://storage.googleapis.com/kubernetes-release/release/v1.15.0/bin/linux/amd64/kubelet
```
创建安装目录：
```text
[root@worker-1 ~]# sudo mkdir -p \
>   /opt/cni/bin \
>   /var/lib/kubelet \
>   /var/lib/kube-proxy \
>   /var/lib/kubernetes \
>   /var/run/kubernetes
```
安装 worker 二进制文件:
```text
[root@worker-1 ~]# chmod +x kubectl kube-proxy kubelet
[root@worker-1 ~]# sudo mv kubectl kube-proxy kubelet /usr/local/bin/
```

### 8.2 配置 Kubelet
```text
[root@worker-1 ~]# sudo cp worker-1-key.pem worker-1.pem /var/lib/kubelet/
[root@worker-1 ~]# sudo cp worker-1.kubeconfig /var/lib/kubelet/kubeconfig
[root@worker-1 ~]# sudo cp ca.pem /var/lib/kubernetes/
[root@worker-1 ~]# sudo tar -xvf cni-plugins-amd64-v0.6.0.tgz -C /opt/cni/bin/
```
生成 kubelet.service systemd 配置文件：
```text
[root@worker-1 ~]# cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
> kind: KubeletConfiguration
> apiVersion: kubelet.config.k8s.io/v1beta1
> authentication:
>   anonymous:
>     enabled: false
>   webhook:
>     enabled: true
>   x509:
>     clientCAFile: "/var/lib/kubernetes/ca.pem"
> authorization:
>   mode: Webhook
> clusterDomain: "cluster.local"
> clusterDNS:
>   - "10.250.0.10"
> runtimeRequestTimeout: "15m"
> tlsCertFile: "/var/lib/kubelet/worker-1.pem"
> tlsPrivateKeyFile: "/var/lib/kubelet/worker-1-key.pem"
> EOF
```
```text
[root@worker-1 ~]# cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
> [Unit]
> Description=Kubernetes Kubelet
> Documentation=https://github.com/kubernetes/kubernetes
> After=docker.service
> Requires=docker.service
>
> [Service]
> ExecStart=/usr/local/bin/kubelet \\
>   --config=/var/lib/kubelet/kubelet-config.yaml \\
>   --image-pull-progress-deadline=2m \\
>   --kubeconfig=/var/lib/kubelet/kubeconfig \\
>   --pod-infra-container-image=cargo.caicloud.io/caicloud/pause-amd64:3.1 \\
>   --network-plugin=cni \\
>   --register-node=true \\
>   --v=2
> Restart=on-failure
> RestartSec=5
>
> [Install]
> WantedBy=multi-user.target
> EOF
```
### 8.3 配置 Kube-Proxy
```text
[root@worker-1 ~]# sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```
生成 kube-proxy.service systemd 配置文件：
```text
[root@worker-1 ~]# cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
> kind: KubeProxyConfiguration
> apiVersion: kubeproxy.config.k8s.io/v1alpha1
> clientConnection:
>   kubeconfig: "/var/lib/kube-proxy/kubeconfig"
> mode: "iptables"
> clusterCIDR: "10.244.0.0/16"
> EOF
```
```text
[root@worker-1 ~]# cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
> [Unit]
> Description=Kubernetes Kube Proxy
> Documentation=https://github.com/kubernetes/kubernetes
>
> [Service]
> ExecStart=/usr/local/bin/kube-proxy \\
>   --config=/var/lib/kube-proxy/kube-proxy-config.yaml
> Restart=on-failure
> RestartSec=5
>
> [Install]
> WantedBy=multi-user.target
> EOF
```
### 8.4 启动 worker 服务
```text
[root@worker-1 ~]# sudo systemctl daemon-reload
[root@worker-1 ~]# sudo systemctl enable kubelet kube-proxy
Created symlink from /etc/systemd/system/multi-user.target.wants/kubelet.service to /etc/systemd/system/kubelet.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/kube-proxy.service to /etc/systemd/system/kube-proxy.service.
[root@worker-1 ~]# sudo systemctl start kubelet kube-proxy
```

这样子 worker-1 节点的配置就好了

## 9. 配置 Kubectl
本部分将生成一个用于 admin 用户的 kubeconfig 文件。
> 注意：在生成 admin 客户端证书的目录来运行本部分的指令。

> 注意，这部分操作实践我还是在 worker-1 当中执行。 (但是文档好像是应该在 master 上执行的，不过我后面操作的时候，放到 worker-1 上执行好像也没啥问题)

## 9.1 admin kubeconfig
为 admin 用户生成 kubeconfig 文件：
```text
[root@worker-1 ~]# export MASTER_IP=172.26.64.9
[root@worker-1 ~]# kubectl config set-cluster kubernetes-training \
>   --certificate-authority=ca.pem \
>   --embed-certs=true \
>   --server=https://${MASTER_IP}:6443
Cluster "kubernetes-training" set.
[root@worker-1 ~]# kubectl config set-credentials admin \
>   --client-certificate=admin.pem \
>   --client-key=admin-key.pem
User "admin" set.
[root@worker-1 ~]# kubectl config set-context kubernetes-training \
>   --cluster=kubernetes-training \
>   --user=admin
Context "kubernetes-training" created.
[root@worker-1 ~]# kubectl config use-context kubernetes-training
Switched to context "kubernetes-training".
```
检查远端 Kubernetes 集群的健康状况:
```text
[root@worker-1 ~]# kubectl get componentstatuses
Error in configuration:
* unable to read client-cert /root/admin.pem for admin due to open /root/admin.pem: no such file or directory
* unable to read client-key /root/admin-key.pem for admin due to open /root/admin-key.pem: no such file or directory
```
发现报错了，找不到 admin.pem 和 admin-key.pem 这两个文件， 这两个文件只在 master 上才有， worker-1 没有。 所以我当时的做法是将 master 上的这两个文件拷贝到 worker-1 上:
```text
[root@worker-master ~]# scp -i /root/k8s admin.pem admin-key.pem  root@172.26.64.2:~/
admin.pem                                                                                                                                                                     100% 1432     3.8MB/s   00:00    
admin-key.pem                                                                                                                                                                 100% 1679     5.1MB/s   00:00
```
然后重新执行一下:
```text
[root@worker-1 ~]# kubectl config set-cluster kubernetes-training   --certificate-authority=ca.pem   --embed-certs=true   --server=https://${MASTER_IP}:6443
Cluster "kubernetes-training" set.
[root@worker-1 ~]# kubectl config set-credentials admin   --client-certificate=admin.pem   --client-key=admin-key.pem
User "admin" set.
[root@worker-1 ~]# kubectl config set-context kubernetes-training   --cluster=kubernetes-training   --user=admin
Context "kubernetes-training" modified.
[root@worker-1 ~]# kubectl config use-context kubernetes-training
Switched to context "kubernetes-training".
```
这时候状态就正常了:
```text
[root@worker-1 ~]# kubectl get componentstatuses
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok                  
scheduler            Healthy   ok                  
etcd-0               Healthy   {"health":"true"}   
```
并且也可以列出远端kubernetes cluster 的节点:
```text
[root@worker-1 ~]# kubectl get nodes
NAME       STATUS     ROLES    AGE   VERSION
worker-1   NotReady   <none>   20m   v1.15.0
```
此时节点已经注册成功了，但节点的状态仍然是 NotReady，我们需要继续安装集群网络以使其 Ready。

## 10. 配置 Pod 网络路由
> 注意，这部分操作实践我还是在 worker-1 当中执行。 (但是文档好像是应该在 master 上执行的，不过我后面操作的时候，放到 worker-1 上执行好像也没啥问题)

每个 Pod 都会从所在 Node 的 Pod CIDR 中分配一个 IP 地址。由于网络还没有配置，跨节点的 Pod 之间还无法通信。

我们可以选择 [Cluster Networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) 来实现 Kubernetes 网络模型。

这里我们选择安装 flannel, 这边要注意一个细节，就是这个 yaml 教程是直接取本地的， 也就是从这个 [github 项目](https://github.com/caicloud/kube-ladder)中的文件，所以我是将这个 github 项目拉到本地，然后解压，然后再引用

```text
[root@worker-1 ~]# kubectl apply -f ./kube-ladder-master/tutorials/resources/kube-flannel.yaml
podsecuritypolicy.extensions/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.extensions/kube-flannel-ds-amd64 created
daemonset.extensions/kube-flannel-ds-arm64 created
daemonset.extensions/kube-flannel-ds-arm created
daemonset.extensions/kube-flannel-ds-ppc64le created
daemonset.extensions/kube-flannel-ds-s390x created
```
这样子就创建好了

## 11. 部署 DNS 扩展
部属 kube-dns 群集扩展:
> 这边也是用到这个 github 项目的一个 yaml 配置文件

> 注意，这部分操作实践我还是在 worker-1 当中执行。 (但是文档好像是应该在 master 上执行的，不过我后面操作的时候，放到 worker-1 上执行好像也没啥问题)

```text
[root@worker-1 ~]# kubectl apply -f ./kube-ladder-master/tutorials/resources/coredns.yaml
serviceaccount/coredns created
clusterrole.rbac.authorization.k8s.io/system:coredns created
clusterrolebinding.rbac.authorization.k8s.io/system:coredns created
configmap/coredns created
deployment.extensions/coredns created
service/kube-dns created
```
列出 kube-dns 部署的 Pod 列表 (要等会儿才会变成 running):
```text
[root@worker-1 ~]# kubectl get pods -l k8s-app=kube-dns -n kube-system
NAME                       READY   STATUS    RESTARTS   AGE
coredns-797d65b84c-9757q   1/1     Running   0          2m27s
coredns-797d65b84c-x7b5m   1/1     Running   1          2m27s
```
接下来验证一下，建立一个 busybox 部署:
```text
[root@worker-1 ~]# kubectl run busybox --image=busybox:1.28.3 --command -- sleep 3600
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
deployment.apps/busybox created
```
列出 busybox 部署的 Pod：
```text
[root@worker-1 ~]# kubectl get pods -l run=busybox
NAME                      READY   STATUS              RESTARTS   AGE
busybox-bc6dbfc6d-s5bgc   0/1     ContainerCreating   0          19s
```
查询 busybox Pod 的全名:
```text
[root@worker-1 ~]# kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}"
busybox-bc6dbfc6d-s5bgc
```
在 busybox Pod 中查询 DNS：
```text
[root@worker-1 ~]# kubectl exec -ti busybox-bc6dbfc6d-s5bgc -- nslookup kubernetes
Server:    10.250.0.10
Address 1: 10.250.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.250.0.1 kubernetes.default.svc.cluster.local
```

这边要注意一个细节，之前我没有在 /etc/hosts 上补上 `172.26.64.2 worker-1` 这一行的时候，这个会报错:
```text
[root@worker-1 ~]# kubectl exec -ti busybox-bc6dbfc6d-s5bgc -- nslookup kubernetes
Error from server: error dialing backend: dial tcp: lookup worker-1 on 183.60.83.19:53: no such host
```
所以要 host 才行。

所以 node 节点  worker-1 集群成功了。 不过这有比较疑惑的就是，我的 9,10,11 步骤都是在 worker-1 上执行的，并没有在 master 上执行 (但是运行结果却是正常的), 所以我怀疑是不是 admin 的凭证对就可以了。

## 12. 烟雾测试
虽然现在集群中只有一台 worker-1 (注意，跟之前的 kubeadm 不一样的是，本次的 master 并没有在集群中，只是控制节点而已), 但是也算集群了，本部分将会运行一系列的测试来验证 Kubernetes 集群的功能正常。

### 12.1 数据加密
本节将会验证 encrypt secret data at rest 的功能。
> 本节是在 master 机器上进行测试的

创建一个 Secret:
```text
[root@worker-master ~]# kubectl create secret generic kubernetes-training \
>   --from-literal="mykey=mydata"
```
查询存在 etcd 里 16 进位编码的 kubernetes-training secret:
```text
[root@worker-master ~]# ETCDCTL_API=3 etcdctl get \
>   --endpoints=http://127.0.0.1:2379 \
>   --cacert=/etc/etcd/ca.pem \
>   --cert=/etc/etcd/kubernetes.pem \
>   --key=/etc/etcd/kubernetes-key.pem\
>   /registry/secrets/default/kubernetes-training | hexdump -C
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 6b 75 62 65 72 6e  |s/default/kubern|
00000020  65 74 65 73 2d 74 72 61  69 6e 69 6e 67 0a 6b 38  |etes-training.k8|
00000030  73 3a 65 6e 63 3a 61 65  73 63 62 63 3a 76 31 3a  |s:enc:aescbc:v1:|
00000040  6b 65 79 31 3a 36 b4 09  2b c5 d7 30 17 85 f3 88  |key1:6..+..0....|
00000050  91 0c 38 5e 06 e2 ce 72  40 f8 c1 be d9 b7 f7 ae  |..8^...r@.......|
00000060  59 da e5 43 04 f4 36 a3  e6 5e 08 17 f5 a9 fb cd  |Y..C..6..^......|
00000070  eb 06 7e 95 67 66 cb 8b  d3 aa 00 cf 89 03 a5 af  |..~.gf..........|
00000080  bd 4d db 22 5d b2 d1 cc  3c 3e 6e 7d 8b cd 8a 1e  |.M."]...<>n}....|
00000090  b8 3f d1 d3 48 51 34 0f  f9 65 4e ed 89 c7 d0 6d  |.?..HQ4..eN....m|
000000a0  38 3f a2 1c 52 1b bc b8  29 f8 f2 32 15 47 58 e2  |8?..R...)..2.GX.|
000000b0  14 b3 c5 bb ab 4a 9a 34  2a b9 d5 69 b2 f8 ae ba  |.....J.4*..i....|
000000c0  b7 62 50 47 7a 4d 42 02  f4 1a 8b 13 67 70 a4 c8  |.bPGzMB.....gp..|
000000d0  b8 e3 2a 86 0a 44 0a 7d  4c c6 18 e2 ee 0c 4c 5a  |..*..D.}L.....LZ|
000000e0  1d 11 0d ca 16 0a                                 |......|
000000e6
```
Etcd 的密钥以 k8s:enc:aescbc:v1:key1 为前缀, 表示使用密钥为 key1 的 aescbc 加密数据。

### 12.2 部署
本节将会验证建立与管理 Deployments 的功能。

创建一个 Deployment 用来搭建 nginx Web 服务：
```text
[root@worker-master ~]# kubectl run nginx --image=nginx
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
deployment.apps/nginx created
```
列出 nginx deployment 的 pods:
```text
[root@worker-master ~]# kubectl get pods -l run=nginx
NAME                     READY   STATUS    RESTARTS   AGE
nginx-7bb7cd8db5-dct2j   1/1     Running   0          53s
```

### 12.3 测试端口转发
本节将会验证使用 port forwarding 从远端进入容器的功能。

查询 nginx pod 的全名:
```text
[root@worker-master ~]# POD_NAME=$(kubectl get pods -l run=nginx -o jsonpath="{.items[0].metadata.name}")
```
将本地机器的 8080 端口转发到 nginx pod 的 80 端口：
```text
[root@worker-master ~]# kubectl port-forward $POD_NAME 8080:80
Forwarding from [::1]:8080 -> 80
```
开一个新的终端来做 HTTP 请求测试:
```text
[root@worker-master ~]# curl --head http://127.0.0.1:8080
HTTP/1.1 200 OK
Content-Type: application/json
Date: Thu, 29 Oct 2020 11:10:32 GMT
```
是正常的。

### 12.4 查看容器日志
```text
[root@worker-master ~]# kubectl logs $POD_NAME
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
```

### 12.5 执行容器命令
```text
[root@worker-master ~]# kubectl exec -ti $POD_NAME -- nginx -v
nginx version: nginx/1.19.3
```

### 12.6 验证服务
将 nginx 部署导出为 NodePort 类型的 Service：
```text
[root@worker-master ~]# kubectl expose deployment nginx --port 80 --type NodePort
service/nginx exposed
```
查询 nginx 服务分配的 Node Port：
```text
[root@worker-master ~]# kubectl get services -o wide
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE     SELECTOR
kubernetes   ClusterIP   10.250.0.1    <none>        443/TCP        64m     <none>
nginx        NodePort    10.250.0.64   <none>        80:31091/TCP   2m42s   run=nginx
```
使用 Node IP 地址 + nginx 服务的 Node Port 做 HTTP 请求测试, 因为当前集群只有 worker-1， 所以是
```text
[root@worker-master ~]# curl -I http://172.26.64.2:31091
HTTP/1.1 200 OK
Server: nginx/1.19.3
Date: Thu, 29 Oct 2020 11:16:33 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 29 Sep 2020 14:12:31 GMT
Connection: keep-alive
ETag: "5f7340cf-264"
Accept-Ranges: bytes
```
所以测试通过。

这边要注意一个细节，虽然  master 节点还没有加入 集群。 但是他是控制节点的， 所以他也可以执行指令， 当然 worker-1 节点也是可以的

master 执行指令:
```text
[root@worker-master ~]# kubectl get nodes
NAME       STATUS   ROLES    AGE   VERSION
worker-1   Ready    <none>   21h   v1.15.0
```
worker-1 节点，执行指令:
```text
[root@worker-1 ~]# kubectl get nodes
NAME       STATUS   ROLES    AGE   VERSION
worker-1   Ready    <none>   21h   v1.15.0
```

## 13. 将 worker-2 也加入到集群
将 worker-2 节点加入到集群中，很简单，在 worker-2 节点上，执行 8，9 步骤的操作即可 (10 和 11 步骤不用再处理)
> 依然将 master 的 admin 的凭证拷贝过来

```text
[root@worker-2 ~]# sudo yum install -y socat conntrack ipset
[root@worker-2 ~]# sudo yum install -y yum-utils device-mapper-persistent-data lvm2
[root@worker-2 ~]# sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
[root@worker-2 ~]# sudo yum install -y docker-ce docker-ce-cli containerd.io
[root@worker-2 ~]# sudo systemctl enable docker
[root@worker-2 ~]# sudo systemctl start docker
[root@worker-2 ~]# wget --timestamping \
>   https://github.com/containernetworking/plugins/releases/download/v0.6.0/cni-plugins-amd64-v0.6.0.tgz \
>   https://storage.googleapis.com/kubernetes-release/release/v1.15.0/bin/linux/amd64/kubectl \
>   https://storage.googleapis.com/kubernetes-release/release/v1.15.0/bin/linux/amd64/kube-proxy \
>   https://storage.googleapis.com/kubernetes-release/release/v1.15.0/bin/linux/amd64/kubelet
[root@worker-2 ~]# sudo mkdir -p \
>   /opt/cni/bin \
>   /var/lib/kubelet \
>   /var/lib/kube-proxy \
>   /var/lib/kubernetes \
>   /var/run/kubernetes
[root@worker-2 ~]# chmod +x kubectl kube-proxy kubelet
[root@worker-2 ~]# sudo mv kubectl kube-proxy kubelet /usr/local/bin/
[root@worker-2 ~]# sudo cp worker-2-key.pem worker-2.pem /var/lib/kubelet/
[root@worker-2 ~]# sudo cp worker-2.kubeconfig /var/lib/kubelet/kubeconfig
[root@worker-2 ~]# sudo cp ca.pem /var/lib/kubernetes/
[root@worker-2 ~]# sudo tar -xvf cni-plugins-amd64-v0.6.0.tgz -C /opt/cni/bin/
[root@worker-2 ~]# cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
> kind: KubeletConfiguration
> apiVersion: kubelet.config.k8s.io/v1beta1
> authentication:
>   anonymous:
>     enabled: false
>   webhook:
>     enabled: true
>   x509:
>     clientCAFile: "/var/lib/kubernetes/ca.pem"
> authorization:
>   mode: Webhook
> clusterDomain: "cluster.local"
> clusterDNS:
>   - "10.250.0.10"
> runtimeRequestTimeout: "15m"
> tlsCertFile: "/var/lib/kubelet/worker-2.pem"
> tlsPrivateKeyFile: "/var/lib/kubelet/worker-2-key.pem"
> EOF
[root@worker-2 ~]# cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
> [Unit]
> Description=Kubernetes Kubelet
> Documentation=https://github.com/kubernetes/kubernetes
> After=docker.service
> Requires=docker.service
>
> [Service]
> ExecStart=/usr/local/bin/kubelet \\
>   --config=/var/lib/kubelet/kubelet-config.yaml \\
>   --image-pull-progress-deadline=2m \\
>   --kubeconfig=/var/lib/kubelet/kubeconfig \\
>   --pod-infra-container-image=cargo.caicloud.io/caicloud/pause-amd64:3.1 \\
>   --network-plugin=cni \\
>   --register-node=true \\
>   --v=2
> Restart=on-failure
> RestartSec=5
>
> [Install]
> WantedBy=multi-user.target
> EOF
[root@worker-2 ~]# sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
[root@worker-2 ~]# cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
> kind: KubeProxyConfiguration
> apiVersion: kubeproxy.config.k8s.io/v1alpha1
> clientConnection:
>   kubeconfig: "/var/lib/kube-proxy/kubeconfig"
> mode: "iptables"
> clusterCIDR: "10.244.0.0/16"
> EOF
[root@worker-2 ~]# cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
> [Unit]
> Description=Kubernetes Kube Proxy
> Documentation=https://github.com/kubernetes/kubernetes
>
> [Service]
> ExecStart=/usr/local/bin/kube-proxy \\
>   --config=/var/lib/kube-proxy/kube-proxy-config.yaml
> Restart=on-failure
> RestartSec=5
>
> [Install]
> WantedBy=multi-user.target
> EOF
[root@worker-2 ~]# sudo systemctl daemon-reload
[root@worker-2 ~]# sudo systemctl enable kubelet kube-proxy
[root@worker-2 ~]# sudo systemctl start kubelet kube-proxy
[root@worker-2 ~]# export MASTER_IP=172.26.64.9
[root@worker-2 ~]# kubectl config set-cluster kubernetes-training \
>   --certificate-authority=ca.pem \
>   --embed-certs=true \
>   --server=https://${MASTER_IP}:6443
[root@worker-2 ~]# kubectl config set-credentials admin \
>   --client-certificate=admin.pem \
>   --client-key=admin-key.pem
[root@worker-2 ~]# kubectl config set-context kubernetes-training \
>   --cluster=kubernetes-training \
>   --user=admin
[root@worker-2 ~]# kubectl config use-context kubernetes-training
```
这样子就好了，接下来检查一下状态:
```text
[root@worker-2 ~]# kubectl get componentstatuses
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok                  
scheduler            Healthy   ok                  
etcd-0               Healthy   {"health":"true"}   
```
查看状态:
```text
[root@worker-2 ~]# kubectl get nodes
NAME       STATUS   ROLES    AGE     VERSION
worker-1   Ready    <none>   22h     v1.15.0
worker-2   Ready    <none>   2m25s   v1.15.0
```

到这边，第二个 node 集群就成功了。 我们知道一旦集群加入成功了，那么当前集群的服务，应该也都是能用的， 所以直接用他的 ip 来请求 nginx 的服务，看一下之前部署的服务是否也在 worker-2 生效：
```text
[root@worker-2 ~]# curl -I http://172.26.64.10:31091
HTTP/1.1 200 OK
Server: nginx/1.19.3
Date: Fri, 30 Oct 2020 08:57:15 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 29 Sep 2020 14:12:31 GMT
Connection: keep-alive
ETag: "5f7340cf-264"
Accept-Ranges: bytes
```

可以看到已经成功了， 说明第二个集群节点也是可以的。

## 总结
这样子 kubernetes 的二进制包的集群环境安装就结束了。 篇幅不短， 事实上，我就算按照教程将其用二进制包的方式来安装集群环境，但是很多细节和操作仍然是一知半解啊， 只能继续学习了， 任重而道远啊，一起加油 XD



