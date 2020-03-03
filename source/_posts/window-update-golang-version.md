---
title: windows 7 升级 golang 版本到 1.13
date: 2020-03-03 15:14:31
tags: 
- golang
- windows
categories: golang相关
---
## 前言
虽然在之前的文章有写过，怎么在 windows 7 上通过 docker 来实现多个 golang 环境, 并使用了 Go Module:
- {% post_link windows7-docker-multiple-golang %}
- {% post_link go-module %}

但是事实上在本机进行开发的时候，还是本地环境装的 golang 环境开发起来更方便 (macOS 还好，差别不大，但是 windows 7 的 docker 实在不怎么友好)， 所以就打算将本机环境的 golang 版本升级到 1.13 版本。

## 升级 golang 版本
查看了一下，发现本机环境装的 golang 还是 `1.7.6` 这个版本，太老了
```text
F:\airdroid_code\go\src\resque-worker-server>go version
go version go1.7.4 windows/amd64
```
<!--more-->
查看了一下，原环境的 GOROOT 和 GOPATH 路径:
```text
set GOPATH=F:\airdroid_code\go
set GORACE=
set GOROOT=C:\Go
```
接下来去下载新的版本的安装包: [下载地址](https://studygolang.com/dl), 不过最新版是 `1.14` 版本，而我们只要 `1.13` 版本的，所以就下载了一个 `1.13.8` 的 `msi`, 然后双击直接安装，刚开始会提示你本机已经有一个 golang 版本，会提示你是否要卸载掉：

![png](1.png)

点击 `yes`, 直接卸掉旧版本，然后选择同样的安装目录来覆盖安装:

![png](2.png)

最后安装完之后，执行一下命令:
```text
F:\airdroid_code\web-readme>go version
go version go1.13.8 windows/amd64
```
发现已经升级上来了。 但是发现他的 GOPATH 路径已经不是原来的 `F:\airdroid_code\go`, 而是变成用户目录下的 go 目录:

![png](3.png)

而我们看了一下， admin 下的用户变量，确实是用户目录下的目录。 所以这个要改一下，改过来

![png](4.png)

改成

![png](5.png)

这时候用 `go env` 就显示正常了。

![png](6.png)

## 找个项目试一下
接下来就找一个项目试一下，go module 是否正常。 之前的  GOPATH 目录下的 pkg 只有这两个目录:

![png](7.png)

当用 go module 安装之后：
```text
F:\airdroid_code\go\src\resque-worker-server>go build
go: downloading github.com/robfig/cron v1.2.0
go: downloading gitlab.airdroid.com/airdroid-utils/goworker v0.0.0-20200219070700-6eb070ff12d8
go: downloading gitlab.airdroid.com/airdroid-utils/iGong v1.3.5
...
go: finding golang.org/x/sys v0.0.0-20190507160741-ecd444e8653b
build gitlab.airdroid.com/airdroid_background/resque-worker-server: cannot load github.com/youtube/vitess/go/pools: github.com/youtube/vitess@v2.1.1+incompatible: read tcp 192.168.40.51:56366->172.217.160.80:443: wsarecv: An esta
blished connection was aborted by the software in your host machine.
```
这边有个报错，下载 `github.com/youtube/vitess/go/pools` 下载失败。这边有两种解决情况：
- 一种就是打开全局的 vpn 下载。                                     
- 一种就是修改 `goproxy` 的配置，改成国内的下载地址

我用的第二种，就是修改 proxy 地址, 原先设置的默认地址是:
```text
$ go env | grep proxy
set GOPROXY=https://proxy.golang.org,direct
```
后面改成:
```text
go env -w GOPROXY=https://goproxy.cn,direct
```
这样就修改成功了:
```text
$ go env | grep proxy
set GOPROXY=https://goproxy.cn,direct
```
然后重新 go build 编译一下，这时候就正常了:
```text
F:\airdroid_code\go\src\resque-worker-server>go build
go: downloading github.com/youtube/vitess v2.1.1+incompatible
go: extracting github.com/youtube/vitess v2.1.1+incompatible
go: downloading github.com/golang/glog v0.0.0-20160126235308-23def4e6c14b
go: extracting github.com/golang/glog v0.0.0-20160126235308-23def4e6c14b
go: finding github.com/youtube/vitess v2.1.1+incompatible
go: finding github.com/golang/glog v0.0.0-20160126235308-23def4e6c14b
```
同时生成 `.exe` 文件了:

![png](8.png)

并且可以执行成功。 而且 `$GOPATH/pkg/mod` 也可以看到 mod 依赖:

![png](9.png)

这边有个细节就是，因为这个项目有拉取内部库的包，正常情况下是要配置 GOPRIVATE 这个配置，让 go 在拉包的时候， 直接采用直连去拉包, 之前在 {% post_link docker-git-ssh %} 就是这样配的。
```text
go env -w GOPRIVATE=*.airdroid.com
```
但是这次并不需要这样配(所以我这个配置还是空的)：
```text
$ go env | grep PRIVATE
set GOPRIVATE=
```
这个是因为我用了另外一个机制，就是 git 配置了:
```text
git config --global url."git@gitlab.airdroid.com:".insteadOf "http://gitlab.airdroid.com/"
```
这时候只要是内部包的拉取，都会从 http 下载方式转换为 ssh 的下载方式，直接通过本机的 ssh 配置进行请求。从而绕过了 GOPRIVATE 设置。
```text
$ git config --list | grep url
url.git@gitlab.airdroid.com:.insteadof=http://gitlab.airdroid.com/
```
而且通过这种方式还可以避免内部库是 http 的问题，因为下载包的时候，只能是 https 协议，如果是 http 协议，那么拉包的时候，就会报错。
## IDE 识别当前项目的 GO Module 依赖路径
虽然可以进行构建了。但是我发现在我的 IDE 中 (用的是 IntelliJ IDEA), 如果是其他项目的依赖还好，可以识别，但是如果是本项目的内部依赖，这时候就会识别不了:

![png](10.png)

这个项目在 go.mod 中的名字就是
```text
module gitlab.airdroid.com/airdroid_background/resque-worker-server
```
但是这个 IDE 在加载这个依赖的时候，没办法认出这其实是本项目的代码。而是去 pkg/mod 中去找，那肯定是找不到。 而且发现 setting 中也没有关于 GO Module 的相关配置:

![png](11.png)

后面打算换同系列中，但是专门用于 golang 项目的 [Goland IDE](https://www.jetbrains.com/go/)。 安装完之后，发现还是爆红，不过我发现他有在 拉包，可能在加载索引吧:

![png](12.png)

看了一下 setting， 终于发现有 Go Module 相关的配置了，不过发现proxy 竟然设置的是直连， 难怪会慢了。

![png](13.png)

而且这边跟 cmd 里面的 GOPROXY 设置的不一样。 所以我就改成一样了：

![png](14.png)

最后重启了一下机子，发现下载也正常了，本项目依赖包的识别也正常了， 代码跳转也正常了:

![png](15.png)

所以后面 golang 项目，只能用 Goland 这个 IDE 了。 不过也有可能是我用的 IDEA 的版本太旧了。新版本可能有兼容 GO Module 的配置，这个后面有空再试下。
