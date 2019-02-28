---
title: 使用 docker 运行 golang 程序
date: 2019-02-14 17:45:57
tags: 
    - golang
    - docker
categories: golang相关
---
## 前言
最近打算将之前项目中的go程序放到docker来维护和管理，所以打算写个简单的demo来看看效果。
docker 基于 Golang 开发，已经不用解释了，而 Golang 天生适合运行在 docker 容器中，却不是这个原因，这得益于：Golang 的静态编译，当在编译的时候关闭 cgo 的时候，可以完全不依赖系统环境。
<!--more-->
## 编写demo代码
```html
[root@VM_156_200_centos ~]# mkdir docker-golang-demo
[root@VM_156_200_centos ~]# cd docker-golang-demo
[root@VM_156_200_centos docker-golang-demo]# vim main.go
[root@VM_156_200_centos docker-golang-demo]# cat main.go
package main

import (
	"fmt"
	"io/ioutil"
	"net/http"
	"os"
)

func main() {
	resp, err := http.Get("https://www.baidu.com")
	check(err)
	body, err := ioutil.ReadAll(resp.Body)
	check(err)
	fmt.Println(len(body))
}

func check(err error) {
	if err != nil {
		fmt.Println(err)
		os.Exit(1)
	}
}
```
这是一个简单的程序，拉取百度的页面，并计算页面的字节大小。需要注意的是，<font color="red">这里采用了 https ，这边埋了一个伏笔，后面会说到</font>。
## 编写 dockerfile
Docker官方提供了Golang各版本的镜像： [Official Repository - golang](https://hub.docker.com/_/golang/).
```html
[root@VM_156_200_centos docker-golang-demo]# vim dockerfile
[root@VM_156_200_centos docker-golang-demo]# cat dockerfile
FROM golang:latest
RUN mkdir /app 
ADD . /app/ 
WORKDIR /app 
RUN go build -o main . 
CMD ["/app/main"]
```
每一行的意思就是：
- Docker的镜像必须基于某个镜像开始，然后开始创建新的镜像，这里基于 golang:latest 开始创建
- 在镜像里创建/app文件夹
- 将当前所在文件夹内所有内容添加到镜像内的/app文件内
- 将镜像内的/app设置为容器工作目录（这里不可使用RUN cd /app切换当前工作目录）
- 在镜像内编译当前目录下Golang代码
- 最后使用CMD命令运行刚才编译出的程序。

这时候这个目录其实就只有两个文件：
```html
[root@VM_156_200_centos docker-golang-demo]# ls
dockerfile  main.go
```
## 编译镜像
```html
[root@VM_156_200_centos docker-golang-demo]# sudo docker build --rm -t golang-demo-app .
```
编译完成之后，这个需要点时间，就可以看到镜像已经生成了
```html
[root@VM_156_200_centos docker-golang-demo]# docker images
REPOSITORY                       TAG                 IMAGE ID            CREATED             SIZE
golang-demo-app                  latest              86ab72857964        2 hours ago         822 MB
docker.io/golang                 latest              901414995ecd        7 days ago          816 MB
。。。
```
接下来执行一下镜像看看效果：
```html
[root@VM_156_200_centos docker-golang-demo]# sudo docker run -it --rm --name golang-demo-app-1 golang-demo-app
227
```
其中**golang-demo-app**是镜像名，**golang-demo-app-1**是容器名。 执行结果没问题。说明我们成功将一个go程序放到docker容器里面执行了。但是从上面的 images 的 list 可以看到这个 demo 的完成大小竟然有 822 M，对于线上的部署，无论是编译时间还是大小都是不合适的，下面的scratch镜像，用来解决这个问题。
## 使用scratch镜像
Scratch 是一个特殊的镜像，它是一个虚拟镜像，也就是一个空白镜像， 它很赞，它简洁、小巧而且快速， 它没有bug、安全漏洞、延缓的代码或技术债务。这是因为它基本上是空的。除了有点儿被Docker添加的metadata (译注：元数据为描述数据的数据)。
你可以用以下命令创建这个scratch镜像（官方文档上有描述）：
```html
[root@VM_156_200_centos docker-golang-demo]# tar cv --files-from /dev/null | docker import - scratch 
sha256:92e88945fd0715812b59aec9e5bcdb60b09afe5e8d5ad142d1c926c908e46c0f
[root@VM_156_200_centos docker-golang-demo]# docker images
REPOSITORY                       TAG                 IMAGE ID            CREATED             SIZE
scratch                          latest              92e88945fd07        4 seconds ago       0 B
。。。
```
可以看到他的大小几乎为 0。而且利用Golang的静态化编译无依赖性，可以大幅度减少编译时间和镜像大小。
### 编译 go
因此我们第一步就要先把go程序编译成二进制文件。
```html
[root@VM_156_200_centos docker-golang-demo]# docker run  --rm -it -v ${PWD}:/go golang:stretch env GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -ldflags -s -a -installsuffix cgo main.go
```
其中：
- –-rm 表示构建完成删除镜像
- -it 表示交互模式 
- -v ${PWD}:/go 加载卷 
- 剩下是设置 linux 变量，其中 GOOS=linux 是将交叉编译的目标设置为Linux，这样你在Mac或者Win下也不会出现问题， cgo 是为了在静态编译中导入net，  -ldflags -s 使用裁剪模式。

这样子我们就在当前目录下，生成了编译好的二进制文件 main。 而这个其实就是我们平时放在服务器上的二进制文件，是可以被执行的：
```html
[root@VM_156_200_centos docker-golang-demo]# ./main
277
```
### 编辑 dockerfile
```html
[root@VM_156_200_centos docker-golang-demo]# vim dockerfile
[root@VM_156_200_centos docker-golang-demo]# cat dockerfile
FROM scratch
RUN mkdir /app
ADD main /app/
WORKDIR /app
CMD ["/app/main"]
```
逻辑很简单，其实就是把当前目录的 main 文件放到镜像里面，然后最后执行镜像里面的 main 程序。
### 编译镜像
接下来编译镜像：
```html
[root@VM_156_200_centos docker-golang-demo]# docker build --rm -t golang-scratch-demo-app .
Sending build context to Docker daemon 4.509 MB
Step 1/5 : FROM scratch
 ---> 
Step 2/5 : RUN mkdir /app
 ---> Running in f79380951be8

container_linux.go:247: starting container process caused "exec: \"/bin/sh\": stat /bin/sh: no such file or directory"
oci runtime error: container_linux.go:247: starting container process caused "exec: \"/bin/sh\": stat /bin/sh: no such file or directory"
```
发现报错了，第二步的时候报错了，也就是 mkdir 的时候，有用到了 /bin/sh 这个文件， 但是这个文件他是没有在 scratch 镜像里面的，因为这个镜像本来就是空的，不会有这个文件？？
这个是相关资料： 
[Cannot start docker container from scratch](https://stackoverflow.com/questions/42584395/cannot-start-docker-container-from-scratch)
[Issues using /bin/sh in CMD command in scratch docker container](https://github.com/moby/moby/issues/17896)
所以我后面就改了一下dockerfile，就不创建app目录了，直接将程序放到镜像的根目录：
```html
[root@VM_156_200_centos docker-golang-demo]# vim dockerfile
[root@VM_156_200_centos docker-golang-demo]# cat dockerfile
FROM scratch
ADD main /
CMD ["/main"]
```
然后再试下：
```html
[root@VM_156_200_centos docker-golang-demo]# docker build --rm -t golang-scratch-demo-app-2 .
Sending build context to Docker daemon 4.509 MB
Step 1/3 : FROM scratch
 ---> 
Step 2/3 : ADD main /
 ---> bca8bd34a8ed
Removing intermediate container f5ec92a8ace6
Step 3/3 : CMD /main
 ---> Running in 7f05155640f3
 ---> 33c14eb6fb33
Removing intermediate container 7f05155640f3
Successfully built 33c14eb6fb33
```
这样就编译成功了。
```html
[root@VM_156_200_centos docker-golang-demo]# docker images
REPOSITORY                       TAG                 IMAGE ID            CREATED             SIZE
golang-scratch-demo-app-2        latest              33c14eb6fb33        11 seconds ago      4.51 MB
。。。
golang-demo-app                  latest              86ab72857964        About an hour ago   822 MB
```
可以看到才 4.5M, 比之刚才的 822M，简直差了几百倍。
### 挂载镜像
```html
[root@VM_156_200_centos docker-golang-demo]# sudo docker run -it --rm --name my-golang-scratch-demo golang-scratch-demo-app-2
Get https://www.baidu.com: x509: certificate signed by unknown authority
```
发现报了一个错误？？？这是一个非常常见的问题：为了进行SSL请求，我们需要SSL根证书。
### 获取证书
根据操作系统，这些证书可以在许多不同的地方。如果您查看Go的x509库，可以查看Go搜索的所有位置。对于许多Linux发行版，这是**/etc/ssl/certs/cacert.pem**。首先，我们将把我们的机器（或Linux VM或在线证书提供者）的**cacert.pem**复制到我们的存储库中。然后，我们将在Docker文件中添加一个ADD，将这个文件放在Go所期望的位置：
下载 cacert.pem 到当前工作目录:
```html
[root@VM_156_200_centos docker-golang-demo]# wget https://curl.haxx.se/ca/cacert.pem
--2019-02-14 17:17:56--  https://curl.haxx.se/ca/cacert.pem
Connecting to 127.0.0.1:8118... connected.
Proxy request sent, awaiting response... 200 OK
Length: 219596 (214K) [application/x-pem-file]
Saving to: ‘cacert.pem’

100%[===============================================================================================================================================================================================================>] 219,596     --.-K/s   in 0.05s   

2019-02-14 17:17:56 (4.42 MB/s) - ‘cacert.pem’ saved [219596/219596]
```
再次编辑 dockerfile：
```html
[root@VM_156_200_centos docker-golang-demo]# cat dockerfile
FROM scratch
ADD cacert.pem /etc/ssl/certs/
ADD main /
CMD ["/main"]
```
再次重新生成镜像：
```html
[root@VM_156_200_centos docker-golang-demo]# docker build --rm -t golang-scratch-demo-app-2 .
Sending build context to Docker daemon 4.729 MB
Step 1/4 : FROM scratch
 ---> 
Step 2/4 : ADD cacert.pem /etc/ssl/certs/
 ---> c160fd8c7d94
Removing intermediate container 4d84742b28c7
Step 3/4 : ADD main /
 ---> d182f68177ae
Removing intermediate container f411337673ee
Step 4/4 : CMD /main
 ---> Running in 254686e2208b
 ---> a7d182445c8d
Removing intermediate container 254686e2208b
Successfully built a7d182445c8d
```
然后重新挂载：
```html
[root@VM_156_200_centos docker-golang-demo]# sudo docker run -it --rm --name my-golang-scratch-demo golang-scratch-demo-app-2
227
```
这次就成功了。

---

参考资料：
[使用docker scratch 空镜像构建golang docker 服务](https://blog.csdn.net/freewebsys/article/details/79224625)
[Golang的docker尝试](https://my.oschina.net/dingdayu/blog/1554105?utm_campaign=studygolang.com&utm_medium=studygolang.com&utm_source=studygolang.com)

