---
title: CentOS 7 安装 GoCV
date: 2021-05-27 17:43:17
tags: 
- golang
- opencv
- cmake
categories: golang相关
---
## 前言
前段时间， 业务上有需要用到图像识别的功能， 用的是开源的 OpenCV, 刚好我们的服务端语言 golang 有支持 OpenCV 这个开源库的库: [GoCV](https://gocv.io/)

所以就在服务器上进行安装, 服务器环境就是 CentOS 7

## 操作之前
既然是 golang 的库，当前要先安装 golang:
```text
yum install golang
```
安装后的版本是:
```text
[root@VM-16-29-centos ~]# go version
go version go1.15.5 linux/amd64
```

## 开始安装
官方有 linux 的安装教程的: [linux install](https://gocv.io/getting-started/linux/)

首先安装 GoCV 的包:
```text
go get -u -d gocv.io/x/gocv
```
然后到目标目录:
```text
cd $GOPATH/src/gocv.io/x/gocv
```
然后在该目录下执行安装就行了:
```text
make install
```
如果一切顺利的话，最后打印应该是版本号:
```text
gocv version: 0.26.0
opencv lib version: 4.5.1
```
所以就完啦 ??? 那写个毛线 blog， 没见过这么水字数的。 所以不出意外， `make install` 应该是报错了:
<!--more-->
```text
CMake Error at CMakeLists.txt:27 (cmake_minimum_required):
  CMake 3.5.1 or higher is required.  You are running version 2.8.12.2


-- Configuring incomplete, errors occurred!
make[1]: Entering directory `/tmp/opencv/opencv-4.5.2/build'
make[1]: *** No targets specified and no makefile found.  Stop.
make[1]: Leaving directory `/tmp/opencv/opencv-4.5.2/build'
make[1]: Entering directory `/tmp/opencv/opencv-4.5.2/build'
make[1]: *** No rule to make target `preinstall'.  Stop.
make[1]: Leaving directory `/tmp/opencv/opencv-4.5.2/build'
/tmp/opencv/opencv-4.5.2
cd /tmp/opencv/opencv-4.5.2/build
sudo make install
sudo ldconfig
cd -
make: *** No rule to make target `install'.  Stop.
/root/go/src/gocv.io/x/gocv
go clean --cache
rm -rf /tmp/opencv
go run ./cmd/version/main.go
# pkg-config --cflags  -- opencv4
Package opencv4 was not found in the pkg-config search path.
Perhaps you should add the directory containing `opencv4.pc'
to the PKG_CONFIG_PATH environment variable
No package 'opencv4' found
pkg-config: exit status 1
make: *** [verify] Error 2
```

一堆的报错，让我无从下手啊， 先从第一个开始解决吧， Cmake 版本太低了
```text
[root@VM-16-29-centos gocv]# cmake -version
cmake version 2.8.12.2
```
先升级到 3.x 版本吧。

## 升级 Cmake
按照这一篇教程来: [记一次 Centos7 cmake 版本升级（由 v2.8.12.2 升级至 v3.14.5）](https://blog.csdn.net/llwy1428/article/details/95473542)

### 1. 先下载并解压
```text
[root@VM-16-29-centos ~]# mkdir /opt/cmake
[root@VM-16-29-centos ~]# cd /opt/cmake/
[root@VM-16-29-centos cmake]# wget https://cmake.org/files/v3.14/cmake-3.14.5.tar.gz
[root@VM-16-29-centos cmake]# tar zxvf cmake-3.14.5.tar.gz
```

### 2. 先将旧的删掉
```text
[root@VM-16-29-centos cmake]# yum remove cmake -y
```
> 旧的那个是刚才执行 `make install` 安装的

### 3. 编译
```text
[root@VM-16-29-centos cmake]# cd cmake-3.14.5/
[root@VM-16-29-centos cmake-3.14.5]# ./configure --prefix=/usr/local/cmake
... 这边执行配置还挺久的
---------------------------------------------
CMake has bootstrapped.  Now run gmake.
```

### 4. 安装
```text
[root@VM-16-29-centos cmake-3.14.5]# make && make install
```

### 5. 创建连接
```text
[root@VM-16-29-centos cmake-3.14.5]# ln -s /usr/local/cmake/bin/cmake /usr/bin/cmake
```

### 6. 查看
```text
[root@VM-16-29-centos cmake-3.14.5]# cmake -version
cmake version 3.14.5

CMake suite maintained and supported by Kitware (kitware.com/cmake).
```

## 再次安装
既然 Cmake 升级好了，那么接下来继续执行 `make install`
```text
[root@VM-16-29-centos cmake-3.14.5]# cd /root/go/src/gocv.io/x/gocv/
[root@VM-16-29-centos gocv]# make install
...
CMake Error at CMakeLists.txt:27 (cmake_minimum_required):
  CMake 3.5.1 or higher is required.  You are running version 2.8.12.2
```
结果还是一样的错误， 又是版本太低， 不是已经升级上来了吗， recheck 一下:
```text
[root@VM-16-24-centos gocv]# cmake --version
cmake version 2.8.12.2
```
神奇的事情发生了， 还是旧版本。 可是刚刚明明是新版本的啊， 所以我怀疑是不是又被覆盖了。

因为是 `make install` 是快速安装，我们没办法去判断是哪一步有问题。 幸好官方也有提供逐步安装的方案。 那我们用逐步安装的方式试一下。

## 逐步安装

### 1. 安装依赖包 make deps
```text
[root@VM-16-24-centos gocv]# make deps
sudo yum -y install pkgconfig cmake curl wget git gtk2-devel libpng-devel libjpeg-devel libtiff-devel tbb tbb-devel libdc1394-devel unzip gcc-c++
Loaded plugins: fastestmirror, langpacks
Loading mirror speeds from cached hostfile
Package 1:pkgconfig-0.27.1-4.el7.x86_64 already installed and latest version
Package cmake-2.8.12.2-2.el7.x86_64 already installed and latest version
```
发现真相了， 安装依赖包的时候，竟然又去装 `cmake-2.8.12.2` 版本的 cmake。 所以我们应该在 build 的时候， 将 cmake 重新升级上来。

### 2. 下载 opencv 包
```text
[root@VM-16-29-centos gocv]# make download
```

### 3. 重新升级 cmake
这时候不需要再走配置了，直接删掉原来的:
```text
[root@VM-16-29-centos cmake]# yum remove cmake -y
```
然后直接安装:
```text
[root@VM-16-29-centos cmake-3.14.5]# make && make install
```
创建连接:
```text
[root@VM-16-29-centos cmake-3.14.5]# ln -s /usr/local/cmake/bin/cmake /usr/bin/cmake
```
升级成功:
```text
[root@VM-16-29-centos cmake-3.14.5]# cmake -version
cmake version 3.14.5
```

### 4. 开始编译
```text
[root@VM-16-29-centos gocv]# make build
```
记得重新回到 gocv 目录， 同时这一步的时间也会耗时久一点

### 5. 安装
```text
[root@VM-16-29-centos gocv]# make sudo_install
```

好这样子就安装完了。

## 验证
接下来验证一下， 这个库有一个 version 的脚本，可以验证是否有成功安装:
```text
[root@VM-16-24-centos gocv]# go run ./cmd/version/main.go
# pkg-config --cflags  -- opencv4
Package opencv4 was not found in the pkg-config search path.
Perhaps you should add the directory containing `opencv4.pc'
to the PKG_CONFIG_PATH environment variable
No package 'opencv4' found
pkg-config: exit status 1
```
发现还是报错了，好像是要配置一个环境变量 `PKG_CONFIG_PATH`, 查了一下资料，发现这边有说: [Centos7 GoCV安装实操](https://blog.csdn.net/weixin_40592935/article/details/104880710)

其实就是在 `/root/.bashrc` 文件下面添加这两行, 作为环境变量
```text
PKG_CONFIG_PATH=$PKG_CONFIG_PATH:/usr/local/lib64/pkgconfig
export PKG_CONFIG_PATH
```
```text
[root@VM-16-24-centos gocv]# vim /root/.bashrc
[root@VM-16-24-centos gocv]# source /root/.bashrc
[root@VM-16-24-centos lib64]# cat /root/.bashrc
...
PKG_CONFIG_PATH=$PKG_CONFIG_PATH:/usr/local/lib64/pkgconfig
export PKG_CONFIG_PATH
```

然后再试一下:
```text
[root@VM-16-29-centos gocv]# go run ./cmd/version/main.go
/tmp/go-build268425498/b001/exe/main: error while loading shared libraries: libopencv_stitching.so.4.5: cannot open shared object file: No such file or directory
exit status 127
```

发现变成另一个错误了。 变成 `libopencv_stitching.so` 这个文件好像找不到

但是我找了一下, 文件是存在的， 在 `/usr/local/lib64` 这个目录下
```text
lrwxrwxrwx 1 root root      26 May 27 15:41 libopencv_stitching.so -> libopencv_stitching.so.4.5
lrwxrwxrwx 1 root root      28 May 27 15:41 libopencv_stitching.so.4.5 -> libopencv_stitching.so.4.5.2
-rwxr-xr-x 1 root root  826992 May 27 15:38 libopencv_stitching.so.4.5.2
```

不过怎么会找不到呢， 再查了一下， 这篇文章有解: [Linux多版本opencv配置与“cannot open shared object file: No such file or directory”问题解决](https://blog.csdn.net/innocent_sheld/article/details/80509651)

按照他的做法，是因为编译器自己找不到安装后的 opencv 库的路径，所以需要告诉系统到指定的地方链接， 所以我们编辑 `/etc/ld.so.conf.d/opencv.conf` 配置文件，补上这个路径 `/usr/local/lib64` 即可 (这个文件默认不存在，所以要新建)
```text
[root@VM-16-29-centos ld.so.conf.d] touch opencv.conf
[root@VM-16-24-centos gocv]# cat /etc/ld.so.conf.d/opencv.conf
/usr/local/lib64
```
然后重新链接一下:
```text
[root@VM-16-24-centos gocv]# sudo ldconfig -v
...
/usr/local/lib64:
    libopencv_saliency.so.4.5 -> libopencv_saliency.so.4.5.2
    libopencv_img_hash.so.4.5 -> libopencv_img_hash.so.4.5.2
    libopencv_wechat_qrcode.so.4.5 -> libopencv_wechat_qrcode.so.4.5.2
    libopencv_plot.so.4.5 -> libopencv_plot.so.4.5.2
```
可以看到已经有链接这个目录了。

最后再执行一下:

```text
[root@VM-16-24-centos gocv]# go run ./cmd/version/main.go
gocv version: 0.27.0
opencv lib version: 4.5.2
```

而这个 脚本 其实就是简单的打印出 gocv 和 opencv 库的版本:
```javascript
package main

import (
	"fmt"
	"gocv.io/x/gocv"
)

func main() {
	fmt.Printf("gocv version: %s\n", gocv.Version())
	fmt.Printf("opencv lib version: %s\n", gocv.OpenCVVersion())
}
```

这下子终于成功了。

## 总结
万丈高楼平地起， 就算官网文档放在你面前，但是整个安装过程还是有很多的坑啊， 不过好在环境终于配好了， 接下来就去撸代码吧。 gocv 这边还有一些很有用的例子可以参考
- https://gocv.io/writing-code/more-examples/






