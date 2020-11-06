---
title: 使用 play with kubernetes 搭建 5 个节点的集群
date: 2020-11-06 17:30:32
tags: kubernetes
categories: kubernetes
---
## 前言
之前有通过 {% post_link k8s-install-minikube %} 我们已经可以在 minikube 学习 kubernetes 了， 但是这个毕竟是单机的， 最多熟悉一下指令， 没办法搞集群， 所以还得搞个可以玩集群的环境， 刚好线上有一个 [Play with Kubernetes](https://labs.play-with-k8s.com/) 可以允许练习 k8s， 只能说， 好人啊。

而且只需要 github 或者 docker 账号就可以登入使用， session 有 4 个小时，并且最大允许 5 个 node 做集群。 接下来我们就来用这个来搭建 5 个节点的集群， 包括一个 master 和 4 个工作节点。

## 实操
首先登入进入，点击 `ADD NEW INSTANCE`, 创建一个 master 实例，这时候就会初始化一个 terminal 控制台
<!--more-->

![1](1.png)

可以看到这些基础环境，比如 docker， kubectl， 都帮我们安装好了，而且 kubectl 的版本是 `v1.18.4`。 接下来就按照他上面提示的教程，初始化集群。

### 1. 初始化集群
```text
[node1 ~]$ kubeadm init --apiserver-advertise-address $(hostname -i) --pod-network-cidr 10.5.0.0/16
Initializing machine ID from random generator.
I1016 06:30:15.344605     753 version.go:252] remote version is much newer: v1.19.3; falling back to: stable-1.18
W1016 06:30:15.626185     753 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
[init] Using Kubernetes version: v1.18.10
...
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
...
kubeadm join 192.168.0.13:6443 --token 8finc4.d5vx4wi7dw0baeel \
    --discovery-token-ca-cert-hash sha256:bcd50483e67279f397b41dfb80a90bb2143af81b76c50df4d27c6c06b27fd918
```
集群就初始化完成了。
### 2. 初始化网络配置
```text
[node1 ~]$ kubectl apply -n kube-system -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 |tr -d '\n')"
serviceaccount/weave-net created
clusterrole.rbac.authorization.k8s.io/weave-net created
clusterrolebinding.rbac.authorization.k8s.io/weave-net created
role.rbac.authorization.k8s.io/weave-net created
rolebinding.rbac.authorization.k8s.io/weave-net created
daemonset.apps/weave-net created
```
### 3. 设置用户权限
接下来按照提示执行几个用户权限设置:
```text
[node1 ~]$ mkdir -p $HOME/.kube
[node1 ~]$ cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
cp: '/etc/kubernetes/admin.conf' and '/root/.kube/config' are the same file
[node1 ~]$ chown $(id -u):$(id -g) $HOME/.kube/config
```
这样子 master 节点就创建好了。

### 4. 创建第二个工作节点
继续点击 `ADD NEW INSTANCE` 再添加一个实例，然后直接加入到集群中，这个语句就在上面 master node节点创建完之后，里面的一个提示语:
```text
[node2 ~]$ kubeadm join 192.168.0.38:6443 --token nb3gra.insm0wisj1vgar88 \
>     --discovery-token-ca-cert-hash sha256:1f0889a86d80e9bb44c10d07564d1868ba674691e47af5b379f0181f9d864916
Initializing machine ID from random generator.
W1016 07:14:45.620118    1478 join.go:346] [preflight] WARNING: JoinControlPane.controlPlane settings will be ignored when control-plane flag is not set.
...
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```
这样子就有两个 node 节点了，在 master 节点查看:
```text
[node1 ~]$ kubectl get nodes
NAME    STATUS   ROLES    AGE     VERSION
node1   Ready    master   8m28s   v1.18.4
node2   Ready    <none>   43s     v1.18.4
```

### 5. 继续创建 3 个工作节点
接下来依样画葫芦， 再创建三个节点，一共 5 个
```text
[node1 ~]$ kubectl get nodes
NAME    STATUS     ROLES    AGE     VERSION
node1   Ready      master   11m     v1.18.4
node2   Ready      <none>   3m42s   v1.18.4
node3   Ready      <none>   73s     v1.18.4
node4   NotReady   <none>   22s     v1.18.4
node5   NotReady   <none>   8s      v1.18.4
```
过了一会儿， 就都变成了 ready
```text
[node1 ~]$ kubectl get nodes
NAME    STATUS   ROLES    AGE   VERSION
node1   Ready    master   34m   v1.18.4
node2   Ready    <none>   26m   v1.18.4
node3   Ready    <none>   24m   v1.18.4
node4   Ready    <none>   23m   v1.18.4
node5   Ready    <none>   23m   v1.18.4
```
这样子包含 5 个集群节点的 k8s 集群就创建好了。 

## 验证一下
接下来在 master 中，部署 nginx pod 实践一下
```text
[node1 ~]$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/application/nginx-app.yaml
service/my-nginx-svc created
deployment.apps/my-nginx created
```
查看状态：
```text
[node1 ~]$ kubectl get pods -o wide
NAME                        READY   STATUS              RESTARTS   AGE   IP          NODE    NOMINATED NODE   READINESS GATES
my-nginx-6b474476c4-bssgb   0/1     ContainerCreating   0          16s   <none>      node3   <none>           <none>
my-nginx-6b474476c4-pxjwc   1/1     Running             0          16s   10.42.0.1   node5   <none>           <none>
my-nginx-6b474476c4-r7dk9   0/1     ContainerCreating   0          16s   <none>      node4   <none>           <none>
```
可以看到有 3 个，这个是因为这个 yaml 文件，是设置副本集为 3 个， 所以有 3 个 pod， 查看了一下，发现只有一台是 running，其他两台还是处理中, 等待一下，然后再查看
```text
[node1 ~]$ kubectl get pods -o wide
NAME                        READY   STATUS    RESTARTS   AGE   IP          NODE    NOMINATED NODE   READINESS GATES
my-nginx-6b474476c4-bssgb   1/1     Running   0          55s   10.36.0.1   node3   <none>           <none>
my-nginx-6b474476c4-pxjwc   1/1     Running   0          55s   10.42.0.1   node5   <none>           <none>
my-nginx-6b474476c4-r7dk9   1/1     Running   0          55s   10.39.0.1   node4   <none>           <none>
```
都 ready 了，查看服务状态:
```text
[node1 ~]$ kubectl get svc
NAME           TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes     ClusterIP      10.96.0.1       <none>        443/TCP        37m
my-nginx-svc   LoadBalancer   10.100.122.59   <pending>     80:32503/TCP   65s
```
可以看到`my-nginx-svc`服务已经启动，内部80端口被映射到了外部 32503 端口， 查看 32503 端口是否监听
```text
[node1 ~]$ ss -anlp|grep 32503
tcp    LISTEN     0      128       *:32503                 *:*                   users:(("kube-proxy",pid=3144,fd=10))
```
32503 端口的确被监听，通过 kube-proxy 网络管理实现
```text
[node1 ~]$ kubectl expose deploy/my-nginx --port 80
service/my-nginx exposed
```
将pod上的80端口暴漏给master节点，  查看服务地址:
```text
[node1 ~]$ kubectl get svc my-nginx -o go-template --template '{{ .spec.clusterIP }}'
10.101.141.240
```
对这个地址进行 curl 访问:
```text
[node1 ~]$ curl 10.101.141.240
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
可以看到访问成功。接下来查看各个状态细节
```text
[node1 ~]$ kubectl get pods
NAME                        READY   STATUS    RESTARTS   AGE
my-nginx-6b474476c4-bssgb   1/1     Running   0          3m55s
my-nginx-6b474476c4-pxjwc   1/1     Running   0          3m55s
my-nginx-6b474476c4-r7dk9   1/1     Running   0          3m55s
[node1 ~]$ kubectl get services
NAME           TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes     ClusterIP      10.96.0.1        <none>        443/TCP        40m
my-nginx       ClusterIP      10.101.141.240   <none>        80/TCP         106s
my-nginx-svc   LoadBalancer   10.100.122.59    <pending>     80:32503/TCP   4m3s
[node1 ~]$ kubectl get deployments
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
my-nginx   3/3     3            3           4m19s
```

一切正常。

---
参考文档:
- [第一次使用 Play with Kubernetes](https://ithelp.ithome.com.tw/articles/10208876)
- [Kubernetes Play with Kubernetes搭建5个节点的K8s集群](https://blog.csdn.net/Zhaopanp_Crise/article/details/102759569)
- [play with k8s 多节点在线部署](https://cloud.tencent.com/developer/article/1530648)




