---
title: 安装 gvm 来管理多个go版本
date: 2019-02-14 11:25:00
tags: golang
categories: golang相关
---
## 前言
现在项目中的多个go项目还是用的是比较低的版本。所以接下来的计划是打算将这些项目的go版本逐个升级到新版本上来。所以在这个过程中一定会有至少两个不同版本的go并存在主机上。
之前在做 nodejs 的时候，就是用 nvm 来管理和切换 nodejs 的多个版本的， 刚好go也找到一个 gvm， 所以打算先在自己的服务器上先实践一下用 gvm 来安装并管理 go 的多个版本。
<!--more-->
我的服务器是CentOS7,并且之前已经有安装一个go版本了：版本是 **1.8.3**， GOPATH 是 **/root/go**
```html
[root@VM_156_200_centos ~]# go version
go version go1.8.3 linux/amd64

[root@VM_156_200_centos ~]# go env
GOARCH="amd64"
GOBIN=""
GOEXE=""
GOHOSTARCH="amd64"
GOHOSTOS="linux"
GOOS="linux"
GOPATH="/root/go"
GORACE=""
GOROOT="/usr/lib/golang"
GOTOOLDIR="/usr/lib/golang/pkg/tool/linux_amd64"
GCCGO="gccgo"
CC="gcc"
GOGCCFLAGS="-fPIC -m64 -pthread -fmessage-length=0 -fdebug-prefix-map=/tmp/go-build133654575=/tmp/go-build -gno-record-gcc-switches"
CXX="g++"
CGO_ENABLED="1"
PKG_CONFIG="pkg-config"
CGO_CFLAGS="-g -O2"
CGO_CPPFLAGS=""
CGO_CXXFLAGS="-g -O2"
CGO_FFLAGS="-g -O2"
CGO_LDFLAGS="-g -O2"
```
## 安装 gvm 
```html
[root@VM_156_200_centos ~]# bash < <(curl -s -S -L https://raw.githubusercontent.com/moovweb/gvm/master/binscripts/gvm-installer)
Cloning from https://github.com/moovweb/gvm.git to /root/.gvm
Created profile for existing install of Go at "/usr/lib/golang"
Installed GVM v1.0.22

Please restart your terminal session or to get started right away run
`source /root/.gvm/scripts/gvm`
```
这样就安装好了，接下来重启 terminal， 然后看看有没有安装成功
```html
[root@VM_156_200_centos ~]# gvm version
WARNING:
Could not find bison

  linux: apt-get install bison

ERROR: Missing requirements.
```
提示少装了 bison，我们装上：
```html
[root@VM_156_200_centos ~]# yum install bison
Loaded plugins: fastestmirror, langpacks
Repository epel is listed more than once in the configuration
epel                                                                                                                                                                                                                              | 4.7 kB  00:00:00     
extras           

。。。

Dependency Installed:
  m4.x86_64 0:1.4.16-10.el7                                                                                                                                                                                                                              

Complete!
```
然后再试下：
```html
[root@VM_156_200_centos ~]# gvm version
Go Version Manager v1.0.22 installed at /root/.gvm
[root@VM_156_200_centos ~]# gvm help
Usage: gvm [command]

Description:
  GVM is the Go Version Manager

Commands:
  version    - print the gvm version number
  get        - gets the latest code (for debugging)
  use        - select a go version to use (--default to set permanently)
  diff       - view changes to Go root
  help       - display this usage text
  implode    - completely remove gvm
  install    - install go versions
  uninstall  - uninstall go versions
  cross      - install go cross compilers
  linkthis   - link this directory into GOPATH
  list       - list installed go versions
  listall    - list available versions
  alias      - manage go version aliases
  pkgset     - manage go packages sets
  pkgenv     - edit the environment for a package set
```
这样子 gvm 就安装好了。看一下当前是否有安装好的版本：（肯定没有的）
```html
[root@VM_156_200_centos ~]# gvm list

gvm gos (installed)

   system
```
## 使用 gvm 安装 go 版本
目前还没有安装的版本，接下来装个**go 1.10** 版本试试：
```html
[root@VM_156_200_centos ~]# gvm install 1.10
Downloading Go source...
ERROR: Couldn't download Go source. Check the logs /root/.gvm/logs/go-download.log
[root@VM_156_200_centos ~]# cat /root/.gvm/logs/go-download.log
Cloning into '/root/.gvm/archive/go'...
fatal: unable to access 'https://go.googlesource.com/go/': Failed connect to go.googlesource.com:443; Connection timed out

```
下载失败，要翻墙才行, 所以就去打开了代理： {% post_link centos7-ss-proxy %}， 然后再试下：
```html
[root@VM_156_200_centos ~]# gvm install 1.10
Downloading Go source...
Updating Go source...
ERROR: Unrecognized Go version
```
结果显示找不到该版本？？？ 我们查下到底有哪些版本（这个指令要翻墙才行）
```html
[root@VM_156_200_centos ~]# gvm listall

gvm gos (available)

   go1
   go1.0.1
   go1.0.2
   go1.0.3
   go1.1
   go1.1rc2
   go1.1rc3
   go1.1.1
   go1.1.2
   go1.2
   go1.2rc2
   go1.2rc3
   go1.2rc4
   go1.2rc5
   go1.2.1
   go1.2.2
   go1.3
   go1.3beta1
   go1.3beta2
   go1.3rc1
   go1.3rc2
   go1.3.1
   go1.3.2
   go1.3.3
   go1.4
   go1.4beta1
   go1.4rc1
   go1.4rc2
   go1.4.1
   go1.4.2
   go1.4.3
   go1.5
   go1.5beta1
   go1.5beta2
   go1.5beta3
   go1.5rc1
   go1.5.1
   go1.5.2
   go1.5.3
   go1.5.4
   go1.6
   go1.6beta1
   go1.6beta2
   go1.6rc1
   go1.6rc2
   go1.6.1
   go1.6.2
   go1.6.3
   go1.6.4
   go1.7
   go1.7beta1
   go1.7beta2
   go1.7rc1
   go1.7rc2
   go1.7rc3
   go1.7rc4
   go1.7rc5
   go1.7rc6
   go1.7.1
   go1.7.2
   go1.7.3
   go1.7.4
   go1.7.5
   go1.7.6
   go1.8
   go1.8beta1
   go1.8beta2
   go1.8rc1
   go1.8rc2
   go1.8rc3
   go1.8.1
   go1.8.2
   go1.8.3
   go1.8.4
   go1.8.5
   go1.8.5rc4
   go1.8.5rc5
   go1.8.6
   go1.8.7
   go1.9
   go1.9beta1
   go1.9beta2
   go1.9rc1
   go1.9rc2
   go1.9.1
   go1.9.2
   go1.9.3
   go1.9.4
   go1.9.5
   go1.9.6
   go1.9.7
   go1.10
   go1.10beta1
   go1.10beta2
   go1.10rc1
   go1.10rc2
   go1.10.1
   go1.10.2
   go1.10.3
   go1.10.4
   go1.10.5
   go1.10.6
   go1.10.7
   go1.10.8
   go1.11
   go1.11beta1
   go1.11beta2
   go1.11beta3
   go1.11rc1
   go1.11rc2
   go1.11.1
   go1.11.2
   go1.11.3
   go1.11.4
   go1.11.5
   go1.12beta1
   go1.12beta2
   go1.12rc1
   release.r56
   release.r57
   release.r58
   release.r59
   release.r60
   release.r57.1
   release.r57.2
   release.r58.1
   release.r58.2
   release.r60.1
   release.r60.2
   release.r60.3
```
版本还是很多的，所以我就挑了一个 1.10.8 版本来装
```html
[root@VM_156_200_centos ~]# gvm install go1.10.8
Installing go1.10.8...
* Compiling...
ERROR: Failed to compile. Check the logs at /root/.gvm/logs/go-go1.10.8-compile.log
ERROR: Failed to use installed version
```
结果编译的时候失败了。查看了一下编译的 log：
```html
[root@VM_156_200_centos ~]# cat /root/.gvm/logs/go-go1.10.8-compile.log
Building Go cmd/dist using /usr/lib/golang.
Building Go toolchain1 using /usr/lib/golang.
Building Go bootstrap cmd/go (go_bootstrap) using Go toolchain1.
Building Go toolchain2 using go_bootstrap and Go toolchain1.
Building Go toolchain3 using go_bootstrap and Go toolchain2.
go build cmd/compile/internal/ssa: /root/.gvm/gos/go1.10.8/pkg/tool/linux_amd64/compile: signal: killed
go tool dist: FAILED: /root/.gvm/gos/go1.10.8/pkg/tool/linux_amd64/go_bootstrap install -gcflags=all= -ldflags=all= -a -i cmd/asm cmd/cgo cmd/compile cmd/link: exit status 1
```
我擦，看不出什么原因， 我重新换个版本试试： 换了一个 1.8 就可以了
```html
[root@VM_156_200_centos ~]# gvm install go1.8
Installing go1.8...
* Compiling...
go1.8 successfully installed!
```
查看了一下当前安装的版本：
```html
[root@VM_156_200_centos ~]# gvm list

gvm gos (installed)

   go1.8
   system
```
这时候就有一个 **go 1.8** 的版本了, 这时候再装一个版本 **go1.11**
```html
[root@VM_156_200_centos ~]# gvm install go1.11
Installing go1.11...
* Compiling...
ERROR: Failed to compile. Check the logs at /root/.gvm/logs/go-go1.11-compile.log
ERROR: Failed to use installed version
```


刚开始也是编译失败的，那时候去网上查了一下，好像要先装一个 **mercurial** 这个软件，这是一个轻量级分布式版本控制系统，那时候就死马当作活马医，先装再说：
```html
[root@VM_156_200_centos ~]# yum install mercurial
```
安装成功之后，然后再试一下，结果竟然编译成功了，我擦，简直不要太惊讶
```html
[root@VM_156_200_centos ~]# gvm install go1.11
Installing go1.11...
* Compiling...
go1.11 successfully installed!
```
这时候就有两个go 版本了
```html
[root@VM_156_200_centos ~]# gvm list

gvm gos (installed)

   go1.11
   go1.8
   system
```
但是这时候执行 **go version** 还是之前在本机上装的版本, 因为我们还没有指定 gvm 安装的 go 版本
```html
[root@VM_156_200_centos ~]# go version
go version go1.8.3 linux/amd64
```
因为我们还没有 use gvm 的版本。 通过 go env ，我们发现原来版本的 GOPATH 是在 /root/go 这个目录
```html
[root@VM_156_200_centos go]# go env | grep 'GOPATH'
GOPATH="/root/go"
```
## 将go版本切换到gvm的版本
假设我们现在要用 gvm 的 go 1.11 版本，那么我们就要先 use 一下：
```html
[root@VM_156_200_centos go]# gvm use go1.11
Now using version go1.11
```
这时候发现 版本和 GOPATH， goroot 都改过来了:
```html
[root@VM_156_200_centos go1.11]# go version
go version go1.11 linux/amd64

[root@VM_156_200_centos go]# go env
GOARCH="amd64"
GOBIN=""
GOCACHE="/root/.cache/go-build"
GOEXE=""
GOFLAGS=""
GOHOSTARCH="amd64"
GOHOSTOS="linux"
GOOS="linux"
GOPATH="/root/.gvm/pkgsets/go1.11/global"
GOPROXY=""
GORACE=""
GOROOT="/root/.gvm/gos/go1.11"
GOTMPDIR=""
GOTOOLDIR="/root/.gvm/gos/go1.11/pkg/tool/linux_amd64"
GCCGO="gccgo"
CC="gcc"
CXX="g++"
CGO_ENABLED="1"
GOMOD=""
CGO_CFLAGS="-g -O2"
CGO_CPPFLAGS=""
CGO_CXXFLAGS="-g -O2"
CGO_FFLAGS="-g -O2"
CGO_LDFLAGS="-g -O2"
PKG_CONFIG="pkg-config"
GOGCCFLAGS="-fPIC -m64 -pthread -fmessage-length=0 -fdebug-prefix-map=/tmp/go-build771272779=/tmp/go-build -gno-record-gcc-switches"
```
这时候就会看到 GOPATH 就会变成 **/root/.gvm/pkgsets/go1.11/global**  这个目录了，这个目录太深了，我们肯定不用了，我们要自己建一个 GOPATH 目录。
首先创建一个新的目录 **/root/go1.11** 这个目录，
```html
[root@VM_156_200_centos ~]# mkdir go1.11
```
然后进入到这个目录，执行:
```html
[root@VM_156_200_centos ~]# cd go1.11/
[root@VM_156_200_centos go1.11]# gvm pkgset create --local
[root@VM_156_200_centos go1.11]# gvm pkgset use --local
Now using version go1.11 in local package set
Local GOPATH is now /root/go1.11/.gvm_local
[root@VM_156_200_centos go1.11]# go env
GOARCH="amd64"
GOBIN=""
GOCACHE="/root/.cache/go-build"
GOEXE=""
GOFLAGS=""
GOHOSTARCH="amd64"
GOHOSTOS="linux"
GOOS="linux"
GOPATH="/root/go1.11:/root/go1.11/.gvm_local/pkgsets/go1.11/local:/root/.gvm/pkgsets/go1.11/global"
GOPROXY=""
GORACE=""
GOROOT="/root/.gvm/gos/go1.11"
GOTMPDIR=""
GOTOOLDIR="/root/.gvm/gos/go1.11/pkg/tool/linux_amd64"
GCCGO="gccgo"
CC="gcc"
CXX="g++"
CGO_ENABLED="1"
GOMOD=""
CGO_CFLAGS="-g -O2"
CGO_CPPFLAGS=""
CGO_CXXFLAGS="-g -O2"
CGO_FFLAGS="-g -O2"
CGO_LDFLAGS="-g -O2"
PKG_CONFIG="pkg-config"
GOGCCFLAGS="-fPIC -m64 -pthread -fmessage-length=0 -fdebug-prefix-map=/tmp/go-build178886880=/tmp/go-build -gno-record-gcc-switches"
```
使用 **gvm pkgset create --local** 命令，将目录设为一个 local 的 pkgset，然后通过**gvm pkgset use --local**来使用它，这时，当前的环境变量的GOPATH为:
```html
/root/go1.11:/root/go1.11/.gvm_local/pkgsets/go1.11/local:/root/.gvm/pkgsets/go1.11/global
```
其实就变成了 **/root/go1.11** 这个目录了
接下来就很简单，就在这个目录下，创建 src 目录即可
```html
[root@VM_156_200_centos go1.11]# mkdir src
```
src 里面就是放代码，我把原先在 **/root/go** 里面的项目代码，都拷到 **/root/go1.11** 这个目录:
```html
[root@VM_156_200_centos src]# cp -r ../../go/src/go_learn_demo  ./go_learn_demo
[root@VM_156_200_centos src]# cp -r ../../go/src/goworker  ./goworker
```
然后看看 能不能跑起来：
```html
[root@VM_156_200_centos src]# cd go_learn_demo/
[root@VM_156_200_centos go_learn_demo]# ll
total 4
drwxr-xr-x 2 root root 4096 Feb 13 17:16 flag
[root@VM_156_200_centos go_learn_demo]# cd flag/
[root@VM_156_200_centos flag]# ll
total 1640
-rw-r--r-- 1 root root     280 Feb 13 17:16 example1.go
-rw-r--r-- 1 root root    1035 Feb 13 17:16 example2.go
[root@VM_156_200_centos flag]# go run example1.go
port is  9088
```
这个简单的程序跑起来没问题，接下来换另一个程序 goworker：
```html
[root@VM_156_200_centos src]# cd goworker/
[root@VM_156_200_centos goworker]# go run *.go
hello_worker.go:5:2: cannot find package "github.com/benmanns/goworker" in any of:
    /root/.gvm/gos/go1.11/src/github.com/benmanns/goworker (from $GOROOT)
    /root/go1.11/src/github.com/benmanns/goworker (from $GOPATH)
    /root/go1.11/.gvm_local/pkgsets/go1.11/local/src/github.com/benmanns/goworker
    /root/.gvm/pkgsets/go1.11/global/src/github.com/benmanns/goworker
```
有个依赖，所以我们要安装一下，退到 src，然后执行 go get：
```html
[root@VM_156_200_centos goworker]# cd ..
[root@VM_156_200_centos src]# go get github.com/benmanns/goworker
[root@VM_156_200_centos src]# ll
total 20
drwxr-xr-x 5 root root 4096 Feb 13 17:19 github.com
drwxr-xr-x 3 root root 4096 Feb 13 17:19 golang.org
drwxr-xr-x 3 root root 4096 Feb 13 17:16 go_learn_demo
drwxr-xr-x 2 root root 4096 Feb 13 17:16 goworker
drwxr-xr-x 3 root root 4096 Feb 13 17:19 vitess.io
```
可以看到已经安装成功了，然后在执行一下：
```html
[root@VM_156_200_centos src]# cd goworker/
[root@VM_156_200_centos goworker]# go run *.go
===========start=========
Error: you must specify at least one queue
```
可以看到执行成功。因此切换没问题。这时候看 gvm，就会指向 1.11 版本中：
```html
[root@VM_156_200_centos src]# gvm list

gvm gos (installed)

=> go1.11
   go1.8
   system
```
接下来就按照同样的操作，切换到 gvm 的另一个版本 1.8， 然后将 GOPATH 设置为 **/root/go1.8**
```html
[root@VM_156_200_centos src]# gvm use go1.8
Now using version go1.8

[root@VM_156_200_centos src]# go version
go version go1.8 linux/amd64

[root@VM_156_200_centos src]# go env
GOARCH="amd64"
GOBIN=""
GOEXE=""
GOHOSTARCH="amd64"
GOHOSTOS="linux"
GOOS="linux"
GOPATH="/root/.gvm/pkgsets/go1.8/global"
GORACE=""
GOROOT="/root/.gvm/gos/go1.8"
GOTOOLDIR="/root/.gvm/gos/go1.8/pkg/tool/linux_amd64"
GCCGO="gccgo"
CC="gcc"
GOGCCFLAGS="-fPIC -m64 -pthread -fmessage-length=0 -fdebug-prefix-map=/tmp/go-build434499559=/tmp/go-build -gno-record-gcc-switches"
CXX="g++"
CGO_ENABLED="1"
PKG_CONFIG="pkg-config"
CGO_CFLAGS="-g -O2"
CGO_CPPFLAGS=""
CGO_CXXFLAGS="-g -O2"
CGO_FFLAGS="-g -O2"
CGO_LDFLAGS="-g -O2"

[root@VM_156_200_centos src]# cd /root && mkdir /root/go1.8
[root@VM_156_200_centos ~]# cd go1.8/
[root@VM_156_200_centos go1.8]# gvm pkgset create --local
[root@VM_156_200_centos go1.8]# gvm pkgset use --local
Now using version go1.8 in local package set
Local GOPATH is now /root/go1.8/.gvm_local
[root@VM_156_200_centos go1.8]# go env
GOARCH="amd64"
GOBIN=""
GOEXE=""
GOHOSTARCH="amd64"
GOHOSTOS="linux"
GOOS="linux"
GOPATH="/root/go1.8:/root/go1.8/.gvm_local/pkgsets/go1.8/local:/root/.gvm/pkgsets/go1.8/global"
GORACE=""
GOROOT="/root/.gvm/gos/go1.8"
GOTOOLDIR="/root/.gvm/gos/go1.8/pkg/tool/linux_amd64"
GCCGO="gccgo"
CC="gcc"
GOGCCFLAGS="-fPIC -m64 -pthread -fmessage-length=0 -fdebug-prefix-map=/tmp/go-build402513367=/tmp/go-build -gno-record-gcc-switches"
CXX="g++"
CGO_ENABLED="1"
PKG_CONFIG="pkg-config"
CGO_CFLAGS="-g -O2"
CGO_CPPFLAGS=""
CGO_CXXFLAGS="-g -O2"
CGO_FFLAGS="-g -O2"
CGO_LDFLAGS="-g -O2"
[root@VM_156_200_centos go1.8]# mkdir src
[root@VM_156_200_centos go1.8]# cd src
[root@VM_156_200_centos src]# cp -r ../../go/src/go_learn_demo  ./go_learn_demo
[root@VM_156_200_centos src]# cp -r ../../go/src/goworker  ./goworker
[root@VM_156_200_centos src]# cd goworker/
[root@VM_156_200_centos goworker]# go run *.go
hello_worker.go:5:2: cannot find package "github.com/benmanns/goworker" in any of:
    /root/.gvm/gos/go1.8/src/github.com/benmanns/goworker (from $GOROOT)
    /root/go1.8/src/github.com/benmanns/goworker (from $GOPATH)
    /root/go1.8/.gvm_local/pkgsets/go1.8/local/src/github.com/benmanns/goworker
    /root/.gvm/pkgsets/go1.8/global/src/github.com/benmanns/goworker
[root@VM_156_200_centos goworker]# cd ..

[root@VM_156_200_centos src]# go get github.com/benmanns/goworker
[root@VM_156_200_centos src]# ll
total 20
drwxr-xr-x 5 root root 4096 Feb 13 17:32 github.com
drwxr-xr-x 3 root root 4096 Feb 13 17:32 golang.org
drwxr-xr-x 3 root root 4096 Feb 13 17:31 go_learn_demo
drwxr-xr-x 2 root root 4096 Feb 13 17:31 goworker
drwxr-xr-x 3 root root 4096 Feb 13 17:32 vitess.io

[root@VM_156_200_centos goworker]# go run *.go
===========start=========
Error: you must specify at least one queue


[root@VM_156_200_centos goworker]# gvm list

gvm gos (installed)

   go1.11
=> go1.8
   system
```
这样子 gvm 的 go 1.8 也切换好了，接下来我们做个试验，看是不是当前跑的就是 go 1.8 的版本，也就是 GOPATH 是go 1.8 的版本。
首先我们把 go1.8 的 GOPATH 目录 是 src 里面的 github.com 目录删掉 (这个其实就是 goworker 程序的依赖包)
```html
[root@VM_156_200_centos src]# cd ../../go1.8/src
[root@VM_156_200_centos src]# ll
total 20
drwxr-xr-x 5 root root 4096 Feb 13 17:32 github.com
drwxr-xr-x 3 root root 4096 Feb 13 17:32 golang.org
drwxr-xr-x 3 root root 4096 Feb 13 17:31 go_learn_demo
drwxr-xr-x 2 root root 4096 Feb 13 17:31 goworker
drwxr-xr-x 3 root root 4096 Feb 13 17:32 vitess.io
[root@VM_156_200_centos src]# rm -rf github.com/
[root@VM_156_200_centos src]# ll
total 16
drwxr-xr-x 3 root root 4096 Feb 13 17:32 golang.org
drwxr-xr-x 3 root root 4096 Feb 13 17:31 go_learn_demo
drwxr-xr-x 2 root root 4096 Feb 13 17:31 goworker
drwxr-xr-x 3 root root 4096 Feb 13 17:32 vitess.io
```
然后到 go1.11 目录里面去执行 goworker 程序，看看会不会报找不到 go 1.8 的依赖包，如果有报这个错误，说明是对的。
注意，本身 go1.11 src 里面有 github.com 这个依赖包，但是我们现在切换的是 go 1.8 版本，他是没有这个依赖包的，所以应该要报错才对。
```html
[root@VM_156_200_centos src]# cd ../../go1.11/src
[root@VM_156_200_centos src]# pwd
/root/go1.11/src
[root@VM_156_200_centos src]# ll
total 20
drwxr-xr-x 5 root root 4096 Feb 13 17:19 github.com
drwxr-xr-x 3 root root 4096 Feb 13 17:19 golang.org
drwxr-xr-x 3 root root 4096 Feb 13 17:16 go_learn_demo
drwxr-xr-x 2 root root 4096 Feb 13 17:35 goworker
drwxr-xr-x 3 root root 4096 Feb 13 17:19 vitess.io
[root@VM_156_200_centos src]# cd goworker/
[root@VM_156_200_centos goworker]# go run *.go
hello_worker.go:5:2: cannot find package "github.com/benmanns/goworker" in any of:
    /root/.gvm/gos/go1.8/src/github.com/benmanns/goworker (from $GOROOT)
    /root/go1.8/src/github.com/benmanns/goworker (from $GOPATH)
    /root/go1.8/.gvm_local/pkgsets/go1.8/local/src/github.com/benmanns/goworker
    /root/.gvm/pkgsets/go1.8/global/src/github.com/benmanns/goworker
```
错误信息正是找不到 go 1.8 GOPATH 路径的依赖包。 说明当前的go版本确实是 1.8 版本。 接下来我们将版本切换到 go1.11 版本，因为这个是有依赖包的，所以应该会执行成功才对：
```html
[root@VM_156_200_centos goworker]# gvm use go1.11
Now using version go1.11
[root@VM_156_200_centos goworker]# gvm list

gvm gos (installed)

=> go1.11
   go1.8
   system

[root@VM_156_200_centos goworker]# go run *.go
hello_worker.go:5:2: cannot find package "github.com/benmanns/goworker" in any of:
    /root/.gvm/gos/go1.11/src/github.com/benmanns/goworker (from $GOROOT)
    /root/.gvm/pkgsets/go1.11/global/src/github.com/benmanns/goworker (from $GOPATH)
```
报错了？？ 发现程序找的 GOPATH 路径竟然还是原来的  **/root/.gvm/gos/go1.11/**， 查看了一下go env，发现确实又变回来了
```html
[root@VM_156_200_centos goworker]# go env | grep GOPATH
GOPATH="/root/.gvm/pkgsets/go1.11/global"
```
这个意味着我们之前设置的 GOPATH 的指向又被重置了， 因此我们又得重新到 go1.11 目录去做 pkgset 的 create 和 use， 这时候不用 create， 但是得重新 use 才行：
```html
[root@VM_156_200_centos goworker]# cd /root/go1.11 && gvm pkgset use --local
Now using version go1.11 in local package set
Local GOPATH is now /root/go1.11/.gvm_local
```
这样子就重新切回来了。
```html
[root@VM_156_200_centos go1.11]# go env | grep GOPATH
GOPATH="/root/go1.11:/root/go1.11/.gvm_local/pkgsets/go1.11/local:/root/.gvm/pkgsets/go1.11/global"
```
程序也就可以执行了。
```html
[root@VM_156_200_centos go1.11]# go run src/goworker/*.go
===========start=========
Error: you must specify at least one queue
```
但是这样很麻烦，这样意味着，我每次切换 gvm 版本，还要重新指定这个版本的 GOPATH 目录才行，不然他目录还会指向原来的那个 global 目录才行。
后面看了一下这个指令：
```html
[root@VM_156_200_centos go1.11]# gvm pkgset list

gvm go package sets (go1.11)

=>L /root/go1.11
    global
    local
    root
```
很神奇的是，我可以通过 **gvm pkgset use global** 给他重新再切换成默认的 global ，但是却没有可以切回到 /root/go1.11 这个目录的方式，只能到这个目录里面，再执行 **use --local** 才行。
这个得后面想想办法，看看有没有更快的方式，可以解决这个问题。
后面有在网上有找到一个方式：[GVM - Go 的多版本管理工具，使用介绍](https://dryyun.com/2018/11/28/how-to-use-gvm/), 就是安装 gvm 成功的时候，会在用户目录下的 .bashrc 文件（也就是 .gvm 目录的上级目录）增加一行：
```html
[root@VM_156_200_centos ~]# cat .bashrc
# .bashrc
。。。

[[ -s "/root/.gvm/scripts/gvm" ]] && source "/root/.gvm/scripts/gvm"
```
作者是说在这句话后面增加相应的环境变量就可以覆盖了：
```html
GOPATH=”xxxx”
GOBIN=”$GOPATH/bin”
PATH==”$PATH:$GOBIN”
```
但是这边的 GOPATH 也是写死的啊，如果我切换多个 go 版本，还是的再切换后一一指定啊？？？ 这个只能后面再找方法解决了
## gvm 命令解释
附上  gvm 的命令解释：
```html
Usage: gvm [command]

Description:
GVM is the Go Version Manager

Commands:
    version - print the gvm version number
    - 打印 GVM 的版本
    
    get - gets the latest code (for debugging)
    - 获取 GVM 最新的代码
    
    use - select a go version to use (--default to set permanently)
    - 当前终端环境使用某个 go 版本，加上 --default 代表所有新打开的终端环境都使用这个版本
    - 查看帮助，`gvm use -h`
    
    diff - view changes to Go root
    - ？？？
    
    help - display this usage text
    - 显示帮助信息
    
    implode - completely remove gvm
    - 彻底删除 gvm 和安装的所有 go 版本和包
    - 如果命令没用，那么删除 `rm -rf ~/.gvm` 目录，去掉 .bashrc 或者 .zshrc 的相关内容即可
    
    install - install go versions
    - 安装某个 go 的版本
    - 可以加上 tag， gvm install [tag]，参考 https://github.com/golang/go/tags ，安装一些非稳定版本
    - 查看帮助，`gvm install - `
    
    uninstall - uninstall go versions
    - 卸载某个 go 版本
    
    cross - install go cross compilers
    - 安装交叉编译器
    - gvm cross [os] [arch]，os = linux/darwin/windows，arch = amd64/386/arm
    
    linkthis - link this directory into GOPATH
    - 链接指定目录到 GOPATH 路径
    - 以个人使用来说，只要正确设置 GOPATH 就行，这个命令基本用不到，可以往下看 GOPATH 设置部分
    - 查看帮助，gvm linkthis -h
    - 吐槽，是不是缺了 unlink 命令。。
    
    list - list installed go versions
    - 列出安装的 Go 版本
    
    listall - list available versions
    - 列出可用的 Go 版本
    - 使用 `--all `，列出所有的 tags
    - 查看帮助，gvm listall help
    
    alias - manage go version aliases
    - 管理 Go 版本别名
    - gvm alias list ，列出所有别名
    - gvm alias create [alias name] [go version name]，创建别名
    - gvm alias delete [alias name] ，删除别名
    - 个人感觉也基本用不到
    
    pkgset - manage go packages sets
    - gvm pkgset [create/use/delete/list/empty] [pkgset name]
    - 管理 GOPATHs 环境变量
    - 会在 `~/.gvm/environments` 目录下创建相应的文件
    - 吐槽，没有类似的 unuse 命令
    
    pkgenv - edit the environment for a package set
    - 编辑 pkgset 的环境变量
    - gvm pkgenv [pkgset name]
```
所以如果要卸掉 gvm 和清掉所有的go版本和相关的依赖包， 就用这个：  
```html
gvm implode
```
## 结语
使用 gvm 虽然可以管理和切换多个go版本，但是一次只能有一个go版本存在，所以只适合在个人的开发环境用，跟 nvm 是一样的。如果是在构建Jenkins服务器上， 有时候是会有并行构建多个不同版本go的程序，这时候 gvm 是满足不了。 所以最好还是用docker来装载不同版本的go，然后再进行程序的构建。

---

参考资料： [GVM - Go 的多版本管理工具，使用介绍](https://dryyun.com/2018/11/28/how-to-use-gvm/)


