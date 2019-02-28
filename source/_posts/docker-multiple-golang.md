---
title: 使用 docker 来让多个 golang 环境并存并实现多版本打包和编译
date: 2019-02-28 10:24:48
tags: 
    - golang
    - docker
categories: docker相关
---
## 前言
现在我们使用 **CI/CD** 是通过 **gitlab + Jenkins** 的方式， 也就是无论是部署到生产环境还是部署到测试环境，都会通过一台叫做部署Jenkins的构建/部署服务器。
以Golang程序的部署来说，比如我要将一个go程序更新部署到测试环境，那么通过 gitlab 的项目分支构建的 webhook，会触发到 Jenkins 对应的构建任务，这时候 Jenkins 就会去 pull gitlab 对应的项目代码，然后开始打包编译成二进制文件，最后在上传到对应的测试服，然后执行测试服的脚本，比如 supervisor 脚本，然后将这个二进制程序启动起来。
但是现在我们 Jenkins 打包服务器上面的 golang 环境还是比较旧的版本，而且是直接装在服务器上的，只有一个版本，现在为了提高 golang 程序的执行效率和使用更新的语法，我后面打算慢慢将 golang 的版本升级上来，但是因为我们的 go 服务实在太多，而且有复杂的，也有简单的。 所以刚开始不可能一下子就把所有的go版本全部升上来，这样风险太大，所以肯定会有一部分的 go 程序还是用的旧版本，但是一部分 go 程序已经要换成新版本了。
所以这个也就要求 Jenkins 打包服务器上，要并存多个 golang 版本的环境，并且 src 还不能共用，并且还要支持同一时间的多个不同的 golang 版本的程序一起构建。
<!--more-->
## GVM 方案
刚开始有考虑过通过 gvm 来做，让它安装并管理 go 的多个版本， 具体实作： {% post_link gvm-install %}。
后面确实也折腾成功了，也可以成功管理和切换多个 go 版本，但是有两个问题：
- 每次切换 go 版本，都要再重新指定该 go 版本的 gopath 路径，这个很麻烦，暂时没有找到解决的方法
- 一次只能切换一个 go 版本，也就是我们不能同时有多个 go 版本同时在跑， 因为 **go env** 只会有一个当前的版本。所以我们没办法实现同一个时间多个 go 版本同时进行编译的情况，恰恰这又是 Jenkins 构建服务器会出现的情况

所以最后只能认为这个 GVM 方案只适合于个人开发环境来使用，不适合构建服务器来使用。

## 每个 go 程序都用 Docker 容器装载
既然要有同时存在多个 go 版本存在的情况，那么我们首先想到的就是 Docker 的容器环境隔离，每一个容器都是一个单独的环境。我完全可以每一个 go 程序都用 Docker 容器装载，然后再运行打包编译。
这边也有一个 Demo 实作：  {% post_link golang-use-docker %}。 这样是可以达到我们要的效果的，不过也有几个问题：
- src 目录的维护太多了，因为每个 go 程序都是一个容器，因此都占用一个完整的 go 环境，这样 src 目录的资源不仅没法复用，而且维护也很麻烦
- 对于公共库的维护很麻烦，我们有一个私有的 go 公共库，专门提取了一些常用的方法和功能，基本上每一个 go 程序都会用到这个公共库，那么这个公共库的维护就会很麻烦，因为每次这个公共库一更新，那么基本上所有项目的 go 程序的容器都得重新 update 这个公共库

事实上以上这两个问题也不是什么大问题啦，因为我们只需要在每次构建的该 go 程序的时候，进入该项目目录，然后执行 **go get** ，go 就会去自动下载这个程序所需要的依赖包。 但是这个也会导致这个容器的体积会非常膨胀就是了。
而且这种方式针对公共库也会有一个隐患： 如果该项目本来就存在，那么执行 go get 是不会再 update 的，所以每次我们公共库更新的时候，就得到所有的容器里面去更新最新的公共库代码，不然是不会更新的。 
因此我们想了另一种方案。

## 针对 go 版本环境来用 Docker 容器挂载
这种方案也是要用 Docker 容器挂载的，不过它不是一个 go 程序一个容器，而是一个 go 版本环境，一个容器。 假设我们有新旧两个 go 版本，那么就只会有两个容器。使用该环境的所有的 go 程序的所有的依赖都是共用的，因此维护也比较简单。
接下来我们具体实作一下：
### 选择一个 go 版本创建镜像
首先从[Docker Official Images](https://hub.docker.com/_/golang/) 选择一个你要的版本，我们选择： go1.10 版本。
在用户目录创建一个存放 dockerfile 的目录,并创建一个 dockerfile
```html
[root@VM_156_200_centos ~]# mkdir docker-go1.10
[root@VM_156_200_centos ~]# cd docker-go1.10/
[root@VM_156_200_centos docker-go1.10]# vim dockerfile
[root@VM_156_200_centos docker-go1.10]# cat dockerfile
FROM golang:1.10
RUN mkdir /app
ADD . /app/
WORKDIR /app
```
接下来 build 成镜像：
```html
[root@VM_156_200_centos docker-go1.10]# sudo docker build --rm -t kbz/golang-1.10 .
...
Successfully built 61b609c1428e
```
这样子就安装好了， 查看一下：
```html
[root@VM_156_200_centos docker-go1.10]# docker images
REPOSITORY                       TAG                 IMAGE ID            CREATED             SIZE
kbz/golang-1.10                  latest              61b609c1428e        28 seconds ago      760 MB
```
### 创建对应的容器
```html
[root@VM_156_200_centos docker-go1.10]# sudo docker run -it  --name golang-1.10-1  kbz/golang-1.10
root@4943d5f1558c:/app#
root@4943d5f1558c:/app# ls
dockerfile
root@4943d5f1558c:/app# cd ..
root@4943d5f1558c:/# ls
app  bin  boot    dev  etc  go  home  lib  lib64    media  mnt  opt  proc  root  run  sbin    srv  sys  tmp  usr  var
root@4943d5f1558c:/# go env
GOARCH="amd64"
GOBIN=""
GOCACHE="/root/.cache/go-build"
GOEXE=""
GOHOSTARCH="amd64"
GOHOSTOS="linux"
GOOS="linux"
GOPATH="/go"
GORACE=""
GOROOT="/usr/local/go"
GOTMPDIR=""
GOTOOLDIR="/usr/local/go/pkg/tool/linux_amd64"
GCCGO="gccgo"
CC="gcc"
CXX="g++"
CGO_ENABLED="1"
CGO_CFLAGS="-g -O2"
CGO_CPPFLAGS=""
CGO_CXXFLAGS="-g -O2"
CGO_FFLAGS="-g -O2"
CGO_LDFLAGS="-g -O2"
PKG_CONFIG="pkg-config"
GOGCCFLAGS="-fPIC -m64 -pthread -fmessage-length=0 -fdebug-prefix-map=/tmp/go-build408694441=/tmp/go-build -gno-record-gcc-switches"
root@4943d5f1558c:/# go env
GOARCH="amd64"
GOBIN=""
GOCACHE="/root/.cache/go-build"
GOEXE=""
GOHOSTARCH="amd64"
GOHOSTOS="linux"
GOOS="linux"
GOPATH="/go"
GORACE=""
GOROOT="/usr/local/go"
GOTMPDIR=""
GOTOOLDIR="/usr/local/go/pkg/tool/linux_amd64"
GCCGO="gccgo"
CC="gcc"
CXX="g++"
CGO_ENABLED="1"
CGO_CFLAGS="-g -O2"
CGO_CPPFLAGS=""
CGO_CXXFLAGS="-g -O2"
CGO_FFLAGS="-g -O2"
CGO_LDFLAGS="-g -O2"
PKG_CONFIG="pkg-config"
GOGCCFLAGS="-fPIC -m64 -pthread -fmessage-length=0 -fdebug-prefix-map=/tmp/go-build928433244=/tmp/go-build -gno-record-gcc-switches"
root@4943d5f1558c:/# go version
go version go1.10.8 linux/amd64
root@4943d5f1558c:/app# exit
exit
```
现在我们使用 **-i**（交互式）和 **-t**（临时终端）参数运行一个容器，然后输入一些交互命令。可以看到创建容器了之后，就直接进入容器的交互界面了，而且目录就是工作目录 app， 然后上一级目录还有很多其他的目录。最后退出。
这边需要注意的一点就是， run 的时候，如果加上 **--rm** 指令的话，会意味着如果交互模式退出来之后，这个容器也会被删掉(**docker ps -a**找不到)，所以除非是临时容器，不然不要加 **--rm**。这时候就可以看到这个容器的 go 版本是 **1.10.8**， 而且 gopath 是 **/go** 这个目录。
这时候如果退出交互模式的话，这个容器也会停止，即执行 **docker ps** 会找不到，但是容器还在，通过 **docker ps -a** 可以找到。
如果想再开启的话，就执行：
```html
[root@VM_156_200_centos docker-go1.10]# docker start  golang-1.10-1
golang-1.10-1
[root@VM_156_200_centos docker-go1.10]# docker ps
CONTAINER ID        IMAGE                   COMMAND                  CREATED             STATUS              PORTS                         NAMES
4943d5f1558c        kbz/golang-1.10         "bash"                   43 minutes ago      Up 2 seconds                                      golang-1.10-1
。。。
```
这时候就启动起来了。 这时候如果要进入容器的交互模式的话，就执行：
```html
[root@VM_156_200_centos docker-go1.10]# docker exec -it golang-1.10-1  /bin/bash
root@4943d5f1558c:/app# go version
go version go1.10.8 linux/amd64
```
当然如果想启动起来的时候，就直接接入交互模式的话，也可以，就执行：
```html
[root@VM_156_200_centos docker-go1.10]# docker start -ai golang-1.10-1
root@4943d5f1558c:/app# ls
dockerfile
```
启动的时候，使用 -a 参数将容器的输出导出到终端，同时使用 -i 参数进行交互式的操作。不过这样子，如果退出的话，容器这时候也会跟 run 的时候一样停止。
这样子 go1.10 环境的容器就搭好了。接下来就开始实作了
### 在容器里面运行 go 程序
首先我们将宿主机的 go 程序文件放到这个容器的 src 目录就行了, 使用:
```html
docker cp 宿主机文件路径  镜像名称:镜像中文件存放路径
```
可以将宿主机的目录拷贝到容器的对应目录。
```html
[root@VM_156_200_centos docker-go1.10]# docker cp /root/go/src/go_learn_demo golang-1.10-1:/go/src/
[root@VM_156_200_centos docker-go1.10]# docker exec -it golang-1.10-1  /bin/bash
root@4943d5f1558c:/app# cd /go/src/
root@4943d5f1558c:/go/src# ls
go_learn_demo
root@4943d5f1558c:/go/src# cd go_learn_demo/
root@4943d5f1558c:/go/src/go_learn_demo# ls
flag
root@4943d5f1558c:/go/src/go_learn_demo# cd flag
root@4943d5f1558c:/go/src/go_learn_demo/flag# ls
example1  example1.go  example2.go
root@4943d5f1558c:/go/src/go_learn_demo/flag# go run example1.go
port is  9088
root@4943d5f1558c:/go/src/go_learn_demo/flag# exit
exit
```
可以看到这个简单的程序执行成功了。 接下来换个有依赖包的：
```html
[root@VM_156_200_centos docker-go1.10]# docker cp /root/go/src/goworker golang-1.10-1:/go/src/
[root@VM_156_200_centos docker-go1.10]# docker exec -it golang-1.10-1  /bin/bash
root@4943d5f1558c:/app# cd /go/src
root@4943d5f1558c:/go/src# ls
go_learn_demo  goworker
root@4943d5f1558c:/go/src# cd goworker/
root@4943d5f1558c:/go/src/goworker# ls
hello_worker.go  worker  worker.go
root@4943d5f1558c:/go/src/goworker# go run *.go
hello_worker.go:5:2: cannot find package "github.com/benmanns/goworker" in any of:
    /usr/local/go/src/github.com/benmanns/goworker (from $GOROOT)
    /go/src/github.com/benmanns/goworker (from $GOPATH)
root@4943d5f1558c:/go/src/goworker# cd ../
root@4943d5f1558c:/go/src# go get github.com/benmanns/goworker
package golang.org/x/net/context: unrecognized import path "golang.org/x/net/context" (https fetch: Get https://golang.org/x/net/context?go-get=1: dial tcp 216.239.37.1:443: i/o timeout)
```
发现确实缺少依赖，所以就到 src 去装依赖了（当然也可以进入项目的目录直接执行 go get 安装会更快）。 结果某些包是要翻墙的。但是我检查了一下，宿主机是有设置代理的，但是容器内没有使用宿主机的代理，所以后面我就折腾了一下： {% post_link docker-container-proxy %}, 这样子容器里面也可以使用宿主机的代理了。
所以重新执行了一下，这时候就正常了：
```html
[root@VM_156_200_centos ~]# docker start -ai golang-1.10-1
root@VM_156_200_centos:/app# cd /go/src/
root@VM_156_200_centos:/go/src# ls
goworker
root@VM_156_200_centos:/go/src# go  get github.com/benmanns/goworker
root@VM_156_200_centos:/go/src# ls
github.comgolang.org    goworker  vitess.io
root@VM_156_200_centos:/go/src# cd goworker/
root@VM_156_200_centos:/go/src/goworker# go run *.go
===========start=========
Error: you must specify at least one queue
```
这样子依赖就下载成功了，并且执行成功了。
### src 目录共享
当然后面我们不可能每次构建的时候，都要 **docker cp** 将宿主机的更新文件拷贝一份到容器里面。所以就要用目录挂载的方式，来实现文件共享。关于 docker volume 实现目录共享挂载的请看：{% post_link docker-volume %}
首先在用户根目录下创建一个目录专门用来共享 docker go 版本容器内的目录： **docker-golang-src**， 然后创建一个 **go-10.1** 目录用来存放 **golang-1.10-1** 这个容器的 src 目录的共享目录：
```html
[root@VM_156_200_centos ~]# mkdir  docker-golang-src
[root@VM_156_200_centos ~]# cd docker-golang-src
[root@VM_156_200_centos docker-golang-src]# mkdir go-10.1
```
接下来将这个目录挂载到这个容器的 **/go/src** 目录, 因为要重新run，而且名字不能重复，所以要先把旧的那个 golang-1.10-1 容器去掉, 而且这边要使用宿主机的代理，所以也要加上 **--net host**:
```html
[root@VM_156_200_centos docker-golang-src]# docker rm golang-1.10-1
[root@VM_156_200_centos docker-golang-src]# docker run -it -v /docker-golang-src/go-10.1:/go/src --net host --name golang-1.10-1  kbz/golang-1.10
root@fefacc7b3218:/app# cd /go
root@fefacc7b3218:/go# ls
bin  src
root@fefacc7b3218:/go# cd src
root@fefacc7b3218:/go/src# ls
```
可以通过 docker inspect 去查看挂载的细节, 找到 Mounts 这一行
```html
[root@VM_156_200_centos docker-golang-src]# docker inspect golang-1.10-1
...
        "Mounts": [
            {
                "Type": "bind",
                "Source": "/root/docker-golang-src/go-10.1",
                "Destination": "/go/src",
                "Mode": "",
                "RW": true,
                "Propagation": "rprivate"
            }
        ],
...
```
可以看到已经匹配了。目前这两个目录都是空的，接下来往宿主机的 **go-10.1** 放入一些go的程序文件，看看 容器内的共享目录会不会也会有？
```html
[root@VM_156_200_centos ~]# cp -r /root/go/src/goworker /root/docker-golang-src/go-10.1/
[root@VM_156_200_centos ~]# cd /root/docker-golang-src/go-10.1/
[root@VM_156_200_centos go-10.1]# ls
goworker
```
这时候看到宿主机的目录已经有 goworker 这个目录了，接下来进入容器内看下会不会也有：
```html
[root@VM_156_200_centos docker-golang-src]# docker start  golang-1.10-1
golang-1.10-1

[root@VM_156_200_centos go-10.1]# docker exec -it golang-1.10-1 /bin/bash
root@c0094ca86758:/app# cd /go/src
root@c0094ca86758:/go/src# ls
goworker
```
可以看到也有。那接下来我们在容器内到 goworker 目录，去下载依赖包， 然后看看下载的依赖包会不会也会出现宿主机的共享目录下:
```html
root@c0094ca86758:/go/src# cd goworker/
root@c0094ca86758:/go/src/goworker# go get
root@c0094ca86758:/go/src/goworker# ls
hello_worker.go  worker  worker.go
root@c0094ca86758:/go/src/goworker# cd ..
root@c0094ca86758:/go/src# ls
github.com  golang.org    goworker  vitess.io
root@c0094ca86758:/go/src# cd goworker/
root@c0094ca86758:/go/src/goworker# go run *.go
===========start=========
Error: you must specify at least one queue
```
可以看到 src 已经有对应的依赖了，并且已经运行成功了,接下来退出，然后到 宿主机对应的共享目录看下，这些依赖是不是也有:
```html
[root@VM_156_200_centos go-10.1]# ls
github.com  golang.org  goworker  vitess.io
```
可以看到也是有的，至于为什么不在宿主机的共享目录用 **go get**，而是要在容器里面，这个是因为我们认为宿主机没有 go 环境，也不需要 go 环境， 容器才有对应的 go 版本环境。
接下来直接通过 docker 指令来运行对应容器的指令，而不需要再去进入交互模式：
```html
[root@VM_156_200_centos goworker]# docker exec golang-1.10-1 bash -c "cd /go/src/goworker && go run *.go"
===========start=========
Error: you must specify at least one queue
```
通过这种方式，不需要进入容器内，也可以实现在容器内实现 shell 的效果。那接下来我们就生成go的二进制文件：
```html
[root@VM_156_200_centos goworker]# docker exec golang-1.10-1 bash -c "cd /go/src/goworker && go build"
[root@VM_156_200_centos goworker]# ls
goworker  hello_worker.go  worker.go
```
这时候宿主机对应的共享目录也出现了 goworker 这个编译后的二进制文件了。那这个其实就是 Jenkins 这些CI/CD 所在服务器上要实现的功能，接下来只要把这个二进制文件上传到对应的服务器就行了。
这个只是 go 环境的一个版本，并且只在 docker 的容器里面。 通过一模一样的操作，我们就可以实现多个 go 版本环境的并存和构建了。

## 总结
接下来总结一下，在 Jenkins 等构建服务器上，如果用 docker 容器来实现不同go版本的构建： 
1. 当 gitlab 项目构建的时候，这时候在宿主机对应的该项目所需的 go 版本的src共享目录，进行 git pull，将最新的代码拉下来到宿主机对应的共享目录，本例就是 **/root/docker-golang-src/go-10.1/**
2. 进入对应的go环境的容器，然后到对应的项目中，执行 go get，下载依赖包，本例就是：
```html
docker exec golang-1.10-1 bash -c "cd /go/src/goworker && go get"
```
3. 最后编译项目，生成二进制文件，本例就是：
```html
docker exec golang-1.10-1 bash -c "cd /go/src/goworker && go build"
```
4. 然后在宿主机的共享目录将这个生成的二进制文件，上传到对应的服务器上，就可以了。

通过这种方式，我后面又创建了一个新的版本容器： 1.7.4：
```html
[root@VM_156_200_centos docker-go1.7.4]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                      PORTS                                                                        NAMES
efc6671de52e        kbz/golang-1.7.4    "/bin/bash"              41 seconds ago      Exited (0) 29 seconds ago                                                                                golang-1.7.4-1
```
这样子新旧版本的 go 版本容器都有了。

## 备忘
这边还有一个细节，就是我们针对我们的公共库的问题，一旦我们的公共库更新的时候，这时候也会触发构建，那么就要把所有的 go 版本环境的容器也要一起更新。不过应该还好，容器不会太多，最多就 2-3 个，而且可以通过脚本去处理，也不是什么大问题。







