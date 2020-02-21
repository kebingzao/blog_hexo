---
title: 初试 Go Module
date: 2020-02-20 13:54:12
tags: 
- golang
- docker
categories: golang相关
---
## 前言
最近打算把服务端的一些 Go 服务升级到 `1.13`, 并且用上 `Go Module` 这个 Go 的第三方包管理机制。 虽然早在 `Go 1.11` 版本发布的时候，就推出了 `Go Module` 这个机制了，但是毕竟那时候还在跟 `Go dep` 撕逼呢, 而且是第一个版本出来的新机制，有点不敢用，怕入坑之后爬不出来。 一直等到了 `1.13` 版本，这个机制已经相对成熟了，而且几乎取代了 `vendor` 机制了，也被广大 gopher 接受了，所以是时候升级上来了。而且升级的代价其实不高，因为 Go 的版本发布几乎都遵守[Go1兼容协议](https://golang.org/doc/go1compat), 也就是说我们几乎不需要改代码依然有很大的概率可以编译成功(事实上改动的确实不多，无非就是针对 `Go Module` 机制，将一些第三方依赖的引用进行调整，具体的业务逻辑几乎不用动)。改动成本那么低，又可以告别 `Go path`, 采用更加优雅的 `Go Module`, 何乐而不为，所以本文主要是讲一下 `Go Module` 的一些简单试用。

<!--more-->
## 环境搭起来
操作环境： windows 7， docker 环境
ps: 之所以不在 linux 上实操，一个是因为我的本地开发环境是 windows 7，而且有权限可以下载内部库的代码。 一旦换了我在腾讯云上私有机器，没有权限下载内部库。所以考量了一下，反正 windows 7 也可以操作，就直接用 windows 7，不过操作起来肯定没有 linux 那么好用就是了，还得配一下对应的工具。

首先在 docker 上，安装一个 `1.13` 的 Go 版本的镜像, 这一块就不说了，具体看我之前的文章:
- {% post_link docker-multiple-golang %}
- {% post_link windows7-docker-multiple-golang %}

打开 `docker toolbox`, 找到对应的目录，里面的 `dockerfile` 的内容是这样子的:
```text
FROM golang:1.13
RUN mkdir /app
ADD . /app/
WORKDIR /app
```
然后在这个目录下执行:
```text
docker build --rm -t kbz/golang-1.13 .  
```
这时候就可以生成一个镜像，然后镜像名字为: `kbz/golang-1.13`

![png](1.png)

我们不用线上的原始 `1.13` 的镜像，而是 `dockerfile` 自己装的 `kbz/golang-1.13` 镜像 (事实上后续我们还会对这个镜像做一些额外参数的处理)

接下来我们就要生成容器，并挂载 `/go/src/` 目录了。 (最好在生成容器的时候，就要把目录挂载好，不然等容器创建了，再挂载目录就很麻烦了，只能到这个docker 容器的配置文件去修改，而且还很容易出错，最后还要重启 docker)

对于 windows 7 来说 (linux 或者 mac OS 根本不用这一步)，要挂载目录之前，要先把这个目录通过 `Oracle VM VirualBox` 设置成共享文件

![png](2.png)

![png](3.png)

这边要注意两个细节:
1. 一定要先设置成共享目录成功，才能挂载生成容器。不能颠倒顺序，颠倒顺序就没法生效。
2. 设置的时候，key 一定要跟挂载目录的 key 一样的，不然会找不到，也没法生效，比如本例中，我共享文件夹的名称是 `/f/docker-golang-src/go-1.13` 这个名称，那么我 run 容器挂载的时候，也是用这个名称

最后设置好之后，要重启 `docker-machine.exe` 来使得测试生效。 

![png](4.png)

接下来就开始生成容器并且挂载目录，将 docker 容器的 `/go/src` 挂载到宿主机的 `/f/docker-golang-src/go-1.13`:
```text
docker run -it -v /f/docker-golang-src/go-1.13:/go/src --name golang-1.13 kbz/golang-1.13
```
随便建一个文件，可以看到目录同步了:

![png](5.png)

![png](6.png)

这样子，基本环境就好了。接下来就可以这个容器里面进行实操了。

## 初始化 go mod
接下来将一些 go 项目的代码挪到了宿主机对应的共享目录。 

![png](7.png)

在啥依赖都没有安装的情况下，我们试着 `go build` 去编译看看，果然报错了:

![png](8.png)

这个报错其实是正常的，因为我们并没有在 `GOPATH` 的目录安装任何依赖。所以编译的时候，会去 `GOPATH` 目录找依赖，肯定找不到。

接下来我们用 go mod 来初始化这个项目:
```text
go mod init goworker
```
可以看到在目录下，多了一个文件 `go.mod`, 打开这个文件只有 3 行代码:
```text
module goworker

go 1.13
```
这时候并没有安装任何的依赖。 同时通过:
```text
go list -m -json all
```
查看这个项目的所有的依赖安装包，结果发现只有这个项目:

![png](9.png)

接下来我们要开始安装依赖了，可以通过 `go mod download` 来显示下载依赖，也可以直接用 `go build` 编译的时候，会直接下载依赖:

![png](10.png)

编译成功之后，依赖也加载完了，这时候除了原来的 `go.mod` 之外，有多了一个 `go.sum` 文件:

![png](11.png)

同时也多了一个编译好的二进制文件，接下来我们执行一下这个二进制文件，看看是否正常:

![png](12.png)

是正常的，接下来我们看下 `go list` 有没有变化:
```text
go list -m -json all
```
![png](13.png)

可以看到依赖就变得很多了。

## go mod 的依赖存放位置
我们知道没有用 `Go Module`, 依赖都会放在 `$GOPATH/src` 这个目录下，那么用了 `Go Module` 机制之后，依赖的安装位置就会跑到 `GOPATH/pkg/mod` 这个目录下面:

![png](14.png)

这样的一个好处就是，不用每次代码都跟依赖放同一个目录了，保证了 workspace 的干净。(以前密密麻麻的一堆依赖和一堆项目。每次找项目都要找很久)，而且因为有安装依赖了，`go.mod` 内容也变了:
```text
module goworker

go 1.13

require (
    github.com/benmanns/goworker v0.1.3
    github.com/cihub/seelog v0.0.0-20170130134532-f561c5e57575 // indirect
    github.com/garyburd/redigo v1.6.0 // indirect
    github.com/golang/glog v0.0.0-20160126235308-23def4e6c14b // indirect
    github.com/youtube/vitess v2.1.1+incompatible // indirect
    golang.org/x/net v0.0.0-20200202094626-16171245cfb2 // indirect
)
```
通过观察我们可以看到有些依赖后面是有标记 `indirect`, 这个是因为同样都是依赖， 但是有些依赖是直接依赖，比如本例的 `github.com/benmanns/goworker` 就是直接依赖，因为会在项目的代码里面直接引用，而这种`github.com/cihub/seelog` 就是间接依赖，也就是直接依赖的依赖，项目代码里面不会引用到。 

而且我们可以通过:
```text
go mod why -m xxx
```
来查看某一个包是否有被项目依赖，而且可以查看依赖的顺序

![png](15.png)

如果两者之间不存在任何依赖。就会提示没有依赖。

接下来去 `/go/pkg/mod` 目录看一下这个依赖的存放位置:

![png](16.png)

可以看到这个版本是 `0.1.3`, 说明这个版本是单纯存放的。那么有没有可能我降级成 `0.1.2` 的时候，还会下载另一个依赖呢？？

## go mod 的升级和降级
接下来我们试下对 `benmanns/goworker` 这个包的版本进行降级，查了一下，发现这个库的上一个版本是 `0.1.2`, 所以我们直接将当前的 `0.1.3` 版本降为 `0.1.2` 版本
```text
go get  github.com/benmanns/goworker@v0.1.2
```
语法很简单，后面跟上具体的版本，如果比当前使用的版本低，就是降级，如果比当前使用的版本高，那么就是升级，所以现在就是降级:

![png](17.png)

可以看到下载完成了。 而且可以看到 `go.mod` 文件变了， 由 `0.1.3` 变成 `0.1.2` 了:

![png](18.png)

而且 `go.sum` 也变了，增加了 `0.1.2` 的校验信息:

![png](19.png)

但是我们还注意一个细节，就是他并没有像 `go.mod` 那样把 `0.1.3` 的信息覆盖掉，而是新增条目，`0.1.3` 的校验信息还在。 对于这种情况，我们可以通过执行:
```text
go mod tidy
```
来清除多余的条目。

![png](20.png)

这时候就可以看到 `0.1.3` 在 `go.sum` 的那两条就不见了。 接下来我们看下构建是否正常:

![png](21.png)

这边构建失败了，说明确实是 `0.1.2` 版本，因为旧版本因为少了两个引用的参数，导致会报错。 

接下来我们看下 `/go/pkg/mod` 里面是不是这两个版本的代码都有呢？

![png](22.png)

可以看到 `0.1.2` 和 `0.1.3` 的版本都有。所以我们后面要用哪个版本，直接通过 `go mod` 来进行调整即可。也就是说，只要我们想要重新更新为 `0.1.3`, 只要执行：
```text
go get  github.com/benmanns/goworker@v0.1.3
```
这样就升级上来了。

## 更新依赖
除了升级和降级之后，我们还可以用更新的方式，来更新依赖。比如我再把他降到  `0.1.2`, 然后通过:
```text
go get -u github.com/benmanns/goworker
```
升级到最新版。最新版就是 `0.1.3`:

![png](23.png)

可以看到先降级到 `0.1.2`， 然后通过 `go get -u` 又升级到最新版了。

如果仅仅要升级`patch`号，而不升级`minor`号，可以使用`go get -u=patch A` 。比如：如果`golang.org/x/text`有`v0.1.1`版本，那么
```text
go get -u=patch golang.org/x/text
```
会将`go.mod`中的`text`后面的版本号变为`v0.1.1`，而不是`v0.3.0`。

如果`go get`后面不接具体`package`，则`go get`仅针对于`main package`。处于`module-aware`工作模式下的`go get`更新某个依赖(无论是升版本还是降版本)时，会自动计算并更新其间接依赖的包的版本。

## verify 校验
`go.sum` 记录每个依赖库的版本和对应的内容的校验和(一个哈希值)。每当增加一个依赖项时，如果`go.sum`中没有，则会将该依赖项的版本和内容校验和添加到`go.sum`中。go命令会使用这些校验和与缓存在本地的依赖包副本元信息(比如：`$GOPATH/pkg/mod/cache/download/golang.org/x/text/@v` 下面的`v0.3.0.ziphash`)进行比对校验。如果我修改了`$GOPATH/pkg/mod/cache/download/golang.org/x/text/@v/v0.3.0.ziphash` 中的值，那么当我执行下面`verify` 命令时会报错。

当然正常情况下，执行是没问题的:
```text
root@xxxxxx:/go/src/go_learn_demo/goworker# go mod verify
all modules verified
```
但是一旦我将某一个改了,将上面的斜杠去掉， 就会报错:

![png](24.png)

![png](25.png)

补上之后，就又正常了。

## replace 语法
`Go Module` 对大陆 Gopher 的一个不友好的方式就是获取某些第三方包的时候，会被墙，这其中最常见的就是`golang.org/x`下面的各种优秀的包。但是在传统的`GOPATH mode`下，我们可以先从`golang.org/x/xxx`的`mirror`站点`github.com/golang/xxx`上`git clone`这些包，然后将其重命名为`golang.org/x/xxx`。这样也能勉强通过开发者本地的编译。又或将这些包放入`vendor`目录并提交到`repo` 中，也能实现正确的构建。

但是`go module`引入后，一旦工作在`module-aware mode`下，`go build`将不 care GOPATH 下或是 vendor 下的包，而是到 `GOPATH/pkg/mod` 查询是否有`module`的`cache`，如果没有，则会去下载某个版本的`module`，而对于`golang.org/x/xxx`下面的`module`，在大陆地区往往会`get`失败。

当然一种常见的情况就是通过 replace 语法，将这个包指向另一个包，最后编译的时候，其实编译的就是指向的包。 但是代码可以不用改。这时候就可以解决下载不了的情况。因为你可以到 `github.com/golang/xxx` 把包下载下来，然后将 `golang.org/x/xxx` 用 replace 指向过去，就可以了。代码都不用改。 当然如果遇到被墙的情况，最好就直接配置 `go proxy`,  你可以这样配置:
```text
export GOPROXY=https://goproxy.io
```
这样就不用担心被墙了。

另外一种用 replace 的场景就是，如果有一些包本地还有，但是网上已经被迁移了，会导致找不到，这时候在不改代码的情况下，可以将这个包用 replace 语法指向这个包新的地址。 

为了演示 replace 这个语法，我举个例子:
比如 `benmanns/goworker` 并没有下载下来，或者是说我们有对这个进行了修改。首先我把这个项目进行了 `fork` 了。这样子地址就是 https://github.com/kebingzao/goworker ，接下来我不直接对代码进行修改，而是将这个第三方库的资源换成我 `fork` 过来的这个地址。 (而且要说明的是，为了让效果有变化，我 fork 的版本其实是有问题的 `0.1.2` 的版本)。

原来的 `go.mod` 长这样子:
```text

module goworker

go 1.13

require (
    github.com/benmanns/goworker v0.1.3
    github.com/cihub/seelog v0.0.0-20170130134532-f561c5e57575 // indirect
    github.com/garyburd/redigo v1.6.0 // indirect
    github.com/golang/glog v0.0.0-20160126235308-23def4e6c14b // indirect
    github.com/youtube/vitess v2.1.1+incompatible // indirect
    golang.org/x/net v0.0.0-20200202094626-16171245cfb2 // indirect
)
```
看一下语法:
```text
go mod edit -replace=old[@v]=new[@v]
```
我们执行这个试下:
```text
go mod edit -replace=github.com/benmanns/goworker@v0.1.3=github.com/kebingzao/goworker
```
发现报错了:

![png](26.png)

可以看到报错了。这个是说明 `github.com/kebingzao/goworker` 这个必须的是本地下载的资源才行。所以我们要下载一下：
```text
go get github.com/kebingzao/goworker
```

![png](27.png)

可以看到已经下载下来了。 而且 `go.mod` 也变了
```text
module goworker

go 1.13

require (
    github.com/benmanns/goworker v0.1.3
    github.com/cihub/seelog v0.0.0-20170130134532-f561c5e57575 // indirect
    github.com/garyburd/redigo v1.6.0 // indirect
    github.com/golang/glog v0.0.0-20160126235308-23def4e6c14b // indirect
    github.com/kebingzao/goworker v0.1.2 // indirect   <----这一行
    github.com/youtube/vitess v2.1.1+incompatible // indirect
    golang.org/x/net v0.0.0-20200202094626-16171245cfb2 // indirect
)
```
多了这一行，而且版本是我们 `fork` 的版本，就是 `0.1.2` 这个旧版，但是其实这个第三方包并没有被我们这个项目所依赖，我们可以用 
```text
go mod why  -m  github.com/kebingzao/goworker
```

![png](28.png)

可以看到这个模块其实并没有被依赖。这时候如果我们执行 `go mod tidy` 指令的话，`go.mod` 和 `go.sum` 就会把不需要的依赖记录清理掉。 但是因为等下我们要替换，所以就先不执行了。

接下来继续替换:

![png](29.png)

还是报这个错误，很显然后面要跟上具体的版本, 所以改成:
```text
go mod edit -replace=github.com/benmanns/goworker@v0.1.3=github.com/kebingzao/goworker@v0.1.2
```

![png](30.png)

这样子就替换成功了。 我们可以看到 `go.mod` 也变了:
```text
module goworker

go 1.13

require (
    github.com/benmanns/goworker v0.1.3
    github.com/cihub/seelog v0.0.0-20170130134532-f561c5e57575 // indirect
    github.com/garyburd/redigo v1.6.0 // indirect
    github.com/golang/glog v0.0.0-20160126235308-23def4e6c14b // indirect
    github.com/kebingzao/goworker v0.1.2 // indirect
    github.com/youtube/vitess v2.1.1+incompatible // indirect
    golang.org/x/net v0.0.0-20200202094626-16171245cfb2 // indirect
)

replace github.com/benmanns/goworker v0.1.3 => github.com/kebingzao/goworker v0.1.2
```
可以看到多了一行 replace 记录。也就是虽然代码没有变。但是引入的 `goworker` 库已经变成我 `fork` 的这个 `0.1.2` 版本了。 这时候如果 `build` 失败。说明确实是用的我 `fork` 的库。因为 `0.1.2` 版本构建的时候，会报错:

![png](31.png)

可以看到 build 的时候，报错了，但是这个错误非常奇怪：
```text
go: github.com/kebingzao/goworker@v0.1.2 used for two different module paths (github.com/benmanns/goworker and github.com/kebingzao/goworker)
```
看起来是因为 `go.mod` 有两个不同的 `kebingzao/goworker` 的路径导致的。所以我用 `go mod tidy` 清理了一下。 所以 `go.mod` 就变成:
```text
module goworker

go 1.13

require (
    github.com/benmanns/goworker v0.1.3
    github.com/cihub/seelog v0.0.0-20170130134532-f561c5e57575 // indirect
    github.com/garyburd/redigo v1.6.0 // indirect
    github.com/golang/glog v0.0.0-20160126235308-23def4e6c14b // indirect
    github.com/youtube/vitess v2.1.1+incompatible // indirect
    golang.org/x/net v0.0.0-20200202094626-16171245cfb2 // indirect
)

replace github.com/benmanns/goworker v0.1.3 => github.com/kebingzao/goworker v0.1.2
```
可以看到原先在 `require` 中的这一条  
```text
github.com/kebingzao/goworker v0.1.2 // indirect
```
已经清掉了。接下来再进行 `go build`，就可以看到，这下子真的报参数错误。所以确实用的是我 `fork` 的 `0.1.2` 版本。这样就没问题了。 所以以上的方式总结一下就是:
这里有几点要注意：
- `replace`应该在引入新的依赖后立即执行，以免`go tools`自动更新`mod`文件时使用了`old package`导致可能的失败
- `package`后面的`version`不可省略。（`edit`所有操作都需要版本`tag`）


基于以上原因，我们替换一个package的步骤应该是这样的：
1. 首先`go get new-package`（如果你知道package的版本tag，那么这一步其实可以省略，如果想使用最新的版本而不想确认版本号，则需要这一步）
2. 然后查看`go.mod`，手动复制`new-package`的版本号（如果你知道版本号，则跳过，这一步十分得不人性化，也许以后会改进）
3. 接着`go mod tidy`或者`go build`或者使用其他的`go tools`，他们会去获取`new-package`然后替换掉`old-package`
4. 最后，在你的代码里直接使用`old-package`的名字，`golang`会自动识别出`replace`，然后实际你的程序将会使用`new-package`，替换成功

如果我不想要 `replace` 了，那么怎么办？？ 一个简单的方式，就是直接在 `go.mod` 将这个 `replace` 的引用这一行去掉。 然后重新执行 `go build` 看下

![png](32.png)

可以看到这种方式是可行的。 而且编译是没问题的。不过要注意一点的是，我们只去掉了 `go.mod` 里面的 `replace` ， 但是 `go.sum` 里面还保留这个 `github.com/kebingzao/goworker v0.1.2` 这个版本的校验， 所以我们还要再执行一下 `go mod tidy` 来清掉这些信息。

## 总结
基本上关于 `Go Module` 的使用就这样了。不过我后面还遇到一个新的情况，就是在 docker 里面使用 Go Module 下载内部库遇到的问题。具体看: {% post_link docker-git-ssh %}


