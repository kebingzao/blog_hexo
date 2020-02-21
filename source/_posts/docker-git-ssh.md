---
title: docker 容器配置 gitlab ssh 遇到的问题
date: 2020-02-21 13:22:31
tags: 
- golang
- docker
- git
categories: golang相关
---
## 前言
通过 {% post_link go-module %} 我们已经能够使用 `Go Module` 来安装项目的依赖包。 接下来我们试一下我们正式线上的项目，会有没有问题。

## 映射 ssh 文件夹
我们将一个叫做 `resque-worker-server` 的项目，放到容器中的共享目录 `/go/src/` 中，并初始化 mod:
```text
go mod init gitlab.airdroid.com/airdroid_background/resque-worker-server
```
然后接下来就直接构建 `go build`, 但是发现报错了:

![png](1.png)
<!--more-->
原来在下载一个内部公共库的时候，显示报错了。 `gitlab.airdroid.com` 这个是我们内部的代码库，要有权限才能拉代码。 当然我的 windows 宿主机是有权限的。在 `C:\User\admin\.ssh` 目录中，所以只要将这个目录当做共享目录去挂载，并且要挂载到 `/.ssh`， 应该就可以了。

因为是 windows 环境，所以就将这个文件夹设置为共享文件夹，然后重启 tool box。

![png](2.png)

不过这边还有一个问题，就是我这个容器已经在运行了，那么怎么在已经运行的容器中增加挂载目录呢？ 直接在原有的容器上 run 肯定是不行的。 有一种方式，就是修改这个容器的配置文件，然后再重启 docker 服务。 当然最简单的方式就是重新 run 一个新的容器。

## 重新run容器的问题
如果要重 run 容器的话，会有一个问题，就是因为我们的共享目录只有容器的 `/go/src/` 目录，就是我们放项目代码的地方。 而放依赖的目录 `/go/pkg/mod` 并不是共享目录。所以一旦我们新建了别的容器，那么这些依赖包又要重新下载了。这样明显不好。 所以在挂载目录上，应该把整个 go path `/go` 都当做共享目录，这样子所有的资料都会在宿主机上了。这个容器销毁也没关系，重新run一个容器重新挂载就行了，资料还在。

然后我们接下来将另一个目录设置成新的容器的 `/go` 挂载目录，然后设置为共享文件夹，最后重新 run 容器的时候，这两个目录都挂载上
```text
$ docker run -it -v /f/docker-golang-src/go-1.13-2:/go -v /c/admin/.ssh:/.ssh--name golang-1.13-2 kbz/golang-1.13
```
这时候整个容器的 go path 目录 `/go` 就全部挂载到宿主机上了，后面就算这个容器不要了，那也没啥影响了。 然后我们再试一下能不能下载内部库:

![png](3.png)

还是报一样的错误?? 会不会是用 http 的协议是不行的啊，因为我们 .ssh 目录走的是 ssh 协议。所以我们就针对这个域名转一下下载的协议，将 http 下载转为 ssh 下载:
```text
git config --global url."git@gitlab.airdroid.com:".insteadOf "http://gitlab.airdroid.com/"
```
这时候再执行一下，发现虽然还是报错了，但是错误码不一样了？

![png](4.png)

发现换成需要输入密码，但是密码一直输错，怀疑是我挂全局代理的原因。 所以我把全局代理关掉。然后再试一下，发现又报错了，这次是另一个错误：

![png](5.png)

发现请求的路径错了， 变成 `proxy.golang.org` 域名的了。难怪会显示超时了。 所以 `proxy` 这个也要改下。

## 构建优化镜像
针对上面的两个要配置的项，我直接优化镜像，在创建镜像的时候，直接将这些配置写进去。新的 dockerfile 是这样子的:
```text
# 最新版本构建容器
FROM golang:1.13
MAINTAINER cheuk
# 创建构建目录
RUN mkdir /gobuilder
# SSH配置
RUN echo "StrictHostKeyChecking no" >> /etc/ssh/ssh_config
# 链接替换
RUN git config --global url."git@gitlab.airdroid.com:".insteadOf "http://gitlab.airdroid.com/"
# 工作目录
WORKDIR /gobuilder
# 环境变量配置
ENV GOPATH $GOPATH:/gobuilder
ENV GO111MODULE off
ENV GOPROXY "https://goproxy.cn,direct"
```
默认的镜像的配置是这样子，通过这个配置，重新生成一个新的镜像
```text
docker build --rm -t kbz/golang-1.13-new .
```
当我 run 容器的时候，就使用这个镜像，也会进行一下环境变量的配置:
```text
$ docker run -it \ 
-v /f/docker-golang-src/go-1.13-2:/go  \
-v /c/admin/.ssh:/.ssh  \
-e GO111MODULE=on \
-e GOPROXY=https://goproxy.cn,direct \
-e GOPRIVATE=*.airdroid.com  \
--name golang-1.13-new kbz/golang-1.13-new
```
可以看到我们这边除了修改 `GOPROXY` 的代理配置之外，还设置了 `GOPRIVATE` 变量，这样子在下载 `*.airdroid.com` 域名的依赖包的时候，就走直连，不走代理。接下来我们在编译一下:

![png](6.png)

发现还是不行，这时候就有点怀疑是不是 git 对象有问题。所以就在容器里面执行:
```text
git config --list
```

![png](7.png)

结果发现真的有问题。确实没有完整的 git 参数， 发现  git 的指令只有run 容器加入的那一条，其他都没有。 但是宿主机是正常的:

![png](8.png)

为啥不一样，我们不是已经挂载了 .ssh 的目录了吗。仔细检查了一下，发现挂载 .ssh 目录的时候， 容器对应的目录写错了， 应该是 `/root/.ssh` 这个目录才对，我们写成 `/.ssh`, 难怪 git 的配置读不到。所以就重新改一下，然后重新 run 容器:
```text
$ docker run -it \ 
-v /f/docker-golang-src/go-1.13-2:/go  \
-v /c/admin/.ssh:/root/.ssh  \
-e GO111MODULE=on \
-e GOPROXY=https://goproxy.cn,direct \
-e GOPRIVATE=*.airdroid.com  \
--name golang-1.13-new kbz/golang-1.13-new
```
然后重新编译，发现还是报错了，不过这次的错误又不一样了：

![png](9.png)

`docker` 读取不了 `/root/.ssh/config` 这个文件？？  显示没有权限。但是很奇怪的是，我可以在容器中 `.ssh` 目录中创建文件，并删掉。 就是 `config` 这个文件读不了。 很奇怪。
```text
chown root:root config

chmod 777 -R  .
```
而且无论是将所有者改成 `root`， 还是将权限改成 `777`， 都没有用？？ 后面看了一下他的用户组信息，发现了端倪了:

![png](10.png)

这个 `1000 staff` 明显是 windows 的用户组，难怪容器会读取失败。 后面就换成不挂载这个目录了。而是将原先宿主机的这个目录拷贝一份到 go path 的共享目录中。然后通过 cp 指令将这些目录里面的文件，都拷贝到 `/root/.ssh`, 这时候就可以看到用户组是对的了，变成了 `root root`, 然后重新执行一下，又报错了:
```text
root@d8c7dd77bf2b:/go/src/resque-worker-server# go build
go: gitlab.airdroid.com/airdroid-utils/iGong@v1.2.5-0.20191223063622-a79d9328e1e
a: invalid version: git fetch -f http://gitlab.airdroid.com/airdroid-utils/iGong
.git refs/heads/*:refs/heads/* refs/tags/*:refs/tags/* in /go/pkg/mod/cache/vcs/
276c7feb3089680424c92efb44b16e7ad26db48cd757decffab21a103d8ea5e6: exit status 12
8:
        no such identity: C:/Users/admin/.ssh/id_rsa: No such file or directory
        git@gitlab.airdroid.com: Permission denied (publickey,gssapi-keyex,gssap
i-with-mic,password).
        fatal: Could not read from remote repository.

        Please make sure you have the correct access rights
        and the repository exists.
```
原来 config 文件中的地址还是 window 的那个地址。所以改成 linux 对应的地址就行了:
```text
# gitlab
Host gitlab.airdroid.com
    HostName gitlab.airdroid.com
    PreferredAuthentications publickey
    IdentityFile /root/.ssh/id_rsa

# github
Host github.com
    HostName github.com
    PreferredAuthentications publickey
    IdentityFile /root/.ssh/id_rsa_github
```
然后再执行一下，还是报错了，不过这个错误，也很眼熟:
```text
root@d8c7dd77bf2b:/go/src/resque-worker-server# go build
go: gitlab.airdroid.com/airdroid-utils/iGong@v1.2.5-0.20191223063622-a79d9328e1e
a: invalid version: git fetch -f http://gitlab.airdroid.com/airdroid-utils/iGong
.git refs/heads/*:refs/heads/* refs/tags/*:refs/tags/* in /go/pkg/mod/cache/vcs/
276c7feb3089680424c92efb44b16e7ad26db48cd757decffab21a103d8ea5e6: exit status 12
8:
        @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
        @         WARNING: UNPROTECTED PRIVATE KEY FILE!          @
        @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
        Permissions 0755 for '/root/.ssh/id_rsa' are too open.
        It is required that your private key files are NOT accessible by others.
        This private key will be ignored.
        Load key "/root/.ssh/id_rsa": bad permissions
        git@gitlab.airdroid.com: Permission denied (publickey,gssapi-keyex,gssap
i-with-mic,password).
        fatal: Could not read from remote repository.

        Please make sure you have the correct access rights
        and the repository exists.
```
发现 `id_rsa` 这个文件 `755` 的权限太大了，要改小一点，改成 `600` 就可以了:
```text
root@d8c7dd77bf2b:~/.ssh# chmod 600 id_rsa
```
这下子真的可以了:
```text
root@d8c7dd77bf2b:/go/src/resque-worker-server# go build
go: downloading gitlab.airdroid.com/zhuoh/goworker v0.0.0-20191016053759-
0f13a
go: downloading github.com/robfig/cron v1.2.0
go: downloading gitlab.airdroid.com/airdroid-utils/iGong v1.2.5-0.2019122
-a79d9328e1ea
go: extracting github.com/robfig/cron v1.2.0
go: downloading gopkg.in/mgo.v2 v2.0.0-20190816093944-a6b53ec6cb22
go: extracting gitlab.airdroid.com/zhuoh/goworker v0.0.0-20191016053759-9
f13a
```
可以了下载了，并且可以运行了。而且容器中的 git 参数也正常了:
```text
root@d8c7dd77bf2b:/go/src/resque-worker-server# git config --list
url.git@gitlab.airdroid.com:.insteadof=http://gitlab.airdroid.com/
core.repositoryformatversion=0
core.filemode=false
core.bare=false
```

## 总结
看似一直报错，其实就是一直在试错的过程。主要是几个点要抓住。
1. 容器读取 git 配置的目录是 `/root/.ssh` 这个目录，不要记错了。
2. 在 linux 或者 mac OS， 可以直接挂载 `.ssh` 到容器的 `/root/.ssh`, 这样子就可以让容器使用宿主机的 git 配置了，如果是 windows 的话，因为用户组规则不一样，不能直接挂载，而是要复制过去，并且要修改 config 里面的路径，改成 linux 的路径。
3. 针对内部库，要设置 `GOPRIVATE` 参数，直接走直连，不能走代理配置
4. 针对代理，如果被墙了，可以配置成这个 `GOPROXY=https://goproxy.cn,direct` 
5. 针对内部库的 http 下载，通过设置 git 配置: 
  ```text
  git config --global url."git@gitlab.airdroid.com:".insteadOf "http://gitlab.airdroid.com/"
  ```
  将其转换为用 ssh 方式下载。并且通过配合镜像的这个指令:
  ```text
  RUN echo "StrictHostKeyChecking no" >> /etc/ssh/ssh_config
  ```
  将首次的询问免除掉。直接下载

## 再总结
在这个过程中，我们发现如果我们将 `GOPATH` 目录挂载到宿主机之后，并且将 `.ssh` 也挂载之后 (如果是 windows，不能直接挂载，但是可以在构建镜像的 dockerfile 上将这个目录注入进去，这样子镜像里面就有这些文件了)，其实这个容器是否存在已经没有关系。

所以还有一种用法，就是将这个容器当做纯粹的 go 环境，不需要进入到容器中，只要这个容器还在就行了，直接在命令行执行:
```text
docker exec golang-1.13-new bash -c "cd /go/src/goworker && go build"
```
这样就行了。

如果命令比较长，就可以将这个命令做成一个 shell 文件。然后放到挂载目录中(要可以在容器中找到), 然后就可以执行了(前提是容器还在运行，如果没有运行的话，可以用 `docker start`来运行):
```text
docker exec golang-1.13-new bash -c "cd /go/shell/goworker.sh"
```

还有一种方式，就是我容器也不保留了，每次执行完命令就注销掉。比如以下这个命令:
```text
docker run --rm -it \
-v $GOPATH/src:/go/src \
-v $(pwd):/gobuilder \
-v ~/.ssh/:/root/.ssh \
-e GO111MODULE=on \
-e GOPROXY=https://goproxy.cn,direct \
-e GOPRIVATE=*.airdroid.com \
gobuilder /bin/sh -c 'go mod download && go build '"$*"
```
执行完之后，通过 `--rm` 直接就销毁掉了。不用担心会有遗漏的容器垃圾。 这个命令比较长，后面可以直接保存成 shell 文件，然后每次在宿主机只要执行这个文件就行了。


