---
title: Docker Volume - 目录挂载以及文件共享
date: 2019-02-25 13:52:02
tags: docker
categories: docker相关
---
## 前言
Docker中的数据可以存储在类似于虚拟机磁盘的介质中，在Docker中称为数据卷（Data Volume）。数据卷可以用来存储Docker应用的数据，也可以用来在Docker容器间进行数据共享。数据卷呈现给Docker容器的形式就是一个目录，支持多个容器间共享，修改也不会影响镜像。使用Docker的数据卷，类似在系统中使用 mount 挂载一个文件系统。
## Docker文件系统工作情况
想要了解Docker Volume，首先我们需要知道Docker的文件系统是如何工作的。Docker镜像是由多个文件系统（只读层）叠加而成。当我们启动一个容器的时候，Docker会加载只读镜像层并在其上（镜像栈顶部）添加一个读写层。如果运行中的容器修改了现有的一个已经存在的文件，那该文件将会从读写层下面的只读层复制到读写层，该文件的只读版本仍然存在，只是已经被读写层中该文件的副本所隐藏。当删除Docker容器，并通过该镜像重新启动时，之前的更改将会丢失。在Docker中，只读层及在顶部的读写层的组合被称为Union File System（联合文件系统）。
<!--more-->
为了能够保存（持久化）数据以及共享容器间的数据，Docker提出了Volume的概念。简单来说，Volume就是目录或者文件，它可以绕过默认的联合文件系统，而以正常的文件或者目录的形式存在于宿主机上。
最常用的其实就是**目录挂载**和**文件共享**的使用。
## 挂载本地目录
Docker容器启动的时候，如果要挂载宿主机的一个目录，可以用-v参数指定，这个其实也是创建一个数据卷，只不过是把一个本地主机的目录当做数据卷挂载在容器上。
譬如我要启动一个 CentOS 容器，宿主机的 /hostTest 目录挂载到容器的 /conainterTest 目录，可通过以下方式指定：
```html
docker run -it  -v /hostTest:/conainterTest --name centos-demo-1  centos 
```
这样在容器启动后，容器内会自动创建 /conainterTest 的目录。通过这种方式，我们可以明确一点，即-v参数中，冒号":"前面的目录是宿主机目录，后面的目录是容器内目录。
### 基础操作
下面我们来验证一下： 我服务器上的docker 版本都是 **18.09.2**
```html
[root@VM_156_200_centos ~]# docker version
Client:
 Version:           18.09.2
```
首先执行命令：
```html
[root@VM_156_200_centos /]# docker run -it  -v /hostTest:/conainterTest --name centos-demo-1  centos 
[root@a8e50a72519e /]# cd conainterTest/
[root@a8e50a72519e conainterTest]# echo "123" > 123
[root@a8e50a72519e conainterTest]# ls
123
[root@a8e50a72519e conainterTest]# exit
exit
```
run 一个 centos 的镜像，如果本地没有的话，会去执行**docker pull centos**下载，并生成一个 centos-demo-1 的容器，并进入交互模式
这时候就可以看到容器里面已经有生成**/conainterTest**这个目录了，接下来我们在这个目录下创建了一个 123 的文件，然后退出容器。
回到宿主机之后，可以看到宿主机对应的目录已经有**/hostTest**这个目录了，并且也有 123 这个文件。
```html
[root@VM_156_200_centos /]# cd /hostTest/
[root@VM_156_200_centos hostTest]# ls
123
[root@VM_156_200_centos hostTest]# echo "456" > 456
[root@VM_156_200_centos hostTest]# ls
123  456
```
而且我们还再新建了一个 456 文件，看看会不会同步到容器中，接下来启动容器查看一下：
```html
[root@VM_156_200_centos hostTest]# docker start -ai  centos-demo-1
[root@a8e50a72519e /]# cd conainterTest/
[root@a8e50a72519e conainterTest]# ls
123  456
[root@a8e50a72519e conainterTest]# cat 456
456
```
可以看到这个是有的，所以这两个目录其实就是共享的。接下来具体分析一下这个命令的几种情况。
### 容器目录不可以为相对路径
具体指令如下：
```html
[root@VM_156_200_centos /]# docker run -it  -v /hostTest:conainterTest --name centos-demo-1  centos 
docker: Error response from daemon: invalid volume specification: '/hostTest:conainterTest': invalid mount config for type "bind": invalid mount path: 'conainterTest' mount path must be absolute.
See 'docker run --help'.
```
直接报错，提示 conainterTest 不是一个绝对路径，所谓的绝对路径，必须以下斜线“/”开头。
### 宿主机目录如果不存在，则会自动生成
先把宿主机的目录删掉：然后在run容器：
ps： 因为有为容器命名，但是因为相同名字的容器创建会报错，所以这边我有一个隐形操作，就是每次操作，都会先执行：
```html
[root@VM_156_200_centos /]# docker rm centos-demo-1 
centos-demo-1
```
当然这个不是重点，所以后面我不会再把这个操作再说明，默认每次run的时候，如果名字相同，那么我就会先删掉同名的容器。
```html
[root@VM_156_200_centos /]# rm -rf  hostTest
[root@VM_156_200_centos /]# docker run -it  -v /hostTest:/conainterTest --name centos-demo-1  centos 
[root@e57a25c8f8e5 /]# ls
anaconda-post.log  bin  conainterTest  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```
查看宿主机，发现新增了 **/hostTest** 这个目录
```html
[root@VM_156_200_centos /]# cd hostTest/
[root@VM_156_200_centos hostTest]# pwd
/hostTest
```
### 宿主机的目录如果为相对路径呢？
这次，我们换个目录名 hostTest1 试试:
```html
[root@VM_156_200_centos /]# docker run -it  -v hostTest1:/conainterTest --name centos-demo-1  centos 
```
接下来去 / 查找有没有增加 hostTest1 目录：
```html
[root@VM_156_200_centos /]# ll / | grep 'hostTest1'
```
发现找不到？？ 那么 hostTest1 在哪里创建呢？ 通过**docker inspect**命令，查看容器“Mounts”那一部分，我们可以得到这个问题的答案。
```html
[root@VM_156_200_centos hostTest]# docker inspect centos-demo-1
[
    {
    ...
        "Mounts": [
            {
                "Type": "volume",
                "Name": "hostTest1",
                "Source": "/var/lib/docker/volumes/hostTest1/_data",
                "Destination": "/conainterTest",
                "Driver": "local",
                "Mode": "z",
                "RW": true,
                "Propagation": ""
            }
        ],
    ...
    }
]
```
可以看出，容器内的 /conainterTest 目录挂载的是宿主机上的 /var/lib/docker/volumes/hostTest1/_data 目录。
原来，所谓的相对路径指的是 **/var/lib/docker/volumes/** ，与宿主机的当前目录无关。
### 只是-v指定一个目录的情况？
先启动一个容器：
```html
[root@VM_156_200_centos volumes]# docker run -it  -v /test2 --name centos-demo-1  centos 
[root@64e9d535b48c /]# ls
anaconda-post.log  bin  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  test2  tmp  usr  var
```
发现 /test2 在容器中有存在，那么宿主机的目录在哪里呢？？ 用 **docker inspect** 查看一下：
```html
[root@VM_156_200_centos hostTest]# docker inspect centos-demo-1
[
    {
    ...
        "Mounts": [
            {
                "Type": "volume",
                "Name": "43c602bf16cf87452499988cd4dbfad834e40e2f01e93d960ec01a557f40bc58",
                "Source": "/var/lib/docker/volumes/43c602bf16cf87452499988cd4dbfad834e40e2f01e93d960ec01a557f40bc58/_data",
                "Destination": "/test2",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            }
        ],
    ...
    }
]
```
可以看出，同上例中的结果类似，只不过，它不是相对路径的目录名，而是随机生成的一个目录名。
### 如果在容器内修改了目录的属主和属组，那么对应的挂载点是否会修改呢？
首先开启一个容器,并查看容器内 /conainterTest/ 的属性
```html
[root@VM_156_200_centos volumes]# docker run -it  -v /hostTest:/conainterTest --name centos-demo-1  centos 
[root@25d6613bcb81 /]# ll -d /conainterTest/
drwxr-xr-x 2 root root 4096 Feb 26 03:24 /conainterTest/
```
接下来查看宿主机内 /hostTest 目录的属性：
```html
[root@VM_156_200_centos volumes]# ll -d /hostTest/
drwxr-xr-x 2 root root 4096 Feb 26 11:24 /hostTest/
```
可以看到都是 root。接下来我们在容器内新建用户，修改 /conainterTest 的属主和属组：
```html
[root@VM_156_200_centos volumes]# docker start -ai  centos-demo-1
[root@25d6613bcb81 /]# useradd kbz
[root@25d6613bcb81 /]# chown -R kbz.kbz /conainterTest/
[root@25d6613bcb81 /]# ll -d /conainterTest/
drwxr-xr-x 2 kbz kbz 4096 Feb 26 03:24 /conainterTest/
```
可以看到已经将属主和属组都变成 kbz 了。接下来查看宿主机 /hostTest 的属主和属组是否会改变？？
```html
[root@VM_156_200_centos volumes]# ll -d /hostTest/
drwxr-xr-x 2 privoxy privoxy 4096 Feb 26 11:24 /hostTest/
```
发现改变了，但是并不是 kbz，而是一个 privoxy ？？
原来，这个与UID有关系，UID，即“用户标识号”，是一个整数，系统内部用它来标识用户。一般情况下它与用户名是一一对应的。首先查看容器内victor对应的UID是多少：
```html
[root@VM_156_200_centos volumes]# docker start -ai  centos-demo-1
[root@25d6613bcb81 /]# cat /etc/passwd | grep kbz
kbz:x:1000:1000::/home/kbz:/bin/bash
```
kbz 的 UID 为 1000，那么宿主机内 1000 对应的用户是谁呢？
```html
[root@VM_156_200_centos volumes]# cat /etc/passwd | grep 1000
privoxy:x:1000:1000::/home/privoxy:/bin/bash
```
果然是 privoxy， 那么就说的通了。
### 容器销毁了，在宿主机上新建的挂载目录是否会消失
在这里，主要验证两种情况：
- 指定了宿主机目录，即  -v /hostTest:/conainterTest。
- 没有指定宿主机目录，即 -v /conainterTest

首先的第一种情况：
```html
[root@VM_156_200_centos /]# rm -rf /hostTest/  
[root@VM_156_200_centos /]# ll | grep hostTest
[root@VM_156_200_centos /]# docker run -it  -v /hostTest:/conainterTest --name centos-demo-1  centos 
[root@a225da0f4576 /]# exit
exit
[root@VM_156_200_centos /]# docker rm centos-demo-1 
centos-demo-1
[root@VM_156_200_centos /]# ll | grep hostTest
drwxr-xr-x    2 root root     4096 Feb 26 14:10 hostTest
```
我们先把宿主机的 hostTest 目录删掉，然后进行挂载，再把容器删掉，最后发现挂载时候建的 hostTest 还存在。
可以看出，即便容器销毁了，新建的挂载目录不会消失。进一步也可验证，如果宿主机目录的属主和属组发生了变化，容器销毁后，宿主机目录的属主和属组不会恢复到挂载之前的状态。

第二种情况：
通过上面的验证知道，如果没有指定宿主机的目录，则容器会在 /var/lib/docker/volumes/ 随机配置一个目录，那么我们看看这种情况下的容器销毁是否会导致相应目录的删除：
首先启动容器：
```html
[root@VM_156_200_centos /]# docker run -it  -v conainterTest --name centos-demo-1  centos 
```
然后通过 docker inspect 来查看挂载的宿主机的目录：
```html
[root@VM_156_200_centos ~]# docker inspect centos-demo-1
[
    {
    ...
        "Mounts": [
            {
                "Type": "volume",
                "Name": "225e07eb178cc49ee6a4bd95d82430fdf77af717fc4924cb0d201a3f2f162683",
                "Source": "/var/lib/docker/volumes/225e07eb178cc49ee6a4bd95d82430fdf77af717fc4924cb0d201a3f2f162683/_data",
                "Destination": "conainterTest",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            }
        ],
    ...
    }
]
```
对应的是/var/lib/docker/volumes/22...83/_data目录, 去查看一下这个目录是否存在：
```html
[root@VM_156_200_centos /]# ll /var/lib/docker/volumes/225e07eb178cc49ee6a4bd95d82430fdf77af717fc4924cb0d201a3f2f162683/
total 4
drwxr-xr-x 2 root root 4096 Feb 26 14:14 _data
```
发现该目录依然存在。而且即使重启了docker服务，该目录依旧存在。
### 挂载宿主机已存在目录后，在容器内对其进行操作，报“Permission denied”
可通过两种方式解决：
- 关闭selinux。
临时关闭：# setenforce 0
永久关闭：修改/etc/sysconfig/selinux文件，将SELINUX的值设置为disabled。

- 以特权方式启动容器 
指定--privileged参数
```html
docker run -it --privileged=true -v /test:/soft centos /bin/bash
```
### 将挂载的路径的权限设置为只读
挂载的路径权限默认为读写。如果指定为只读可以用：**ro**
```html
[root@VM_156_200_centos /]# docker run -it  -v /hostTest:/conainterTest:ro --name centos-demo-1  centos 
[root@c9c2a58bbfef /]# cd conainterTest/
[root@c9c2a58bbfef conainterTest]# echo "12" > 12
bash: 12: Read-only file system
```
这时候，如果在容器的挂载目录下进行读写操作的话，就会报错。
## 挂载文件
不仅可以挂载目录，还可以挂载文件。如果挂载目录可以理解为系统的目录映射的话，那么挂载文件，也可以理解为文件映射。
语法跟挂载目录差不多。<font color=red>不过有一点要注意的是，挂载宿主机文件的时候，该文件一定要存在，如果该文件不存在，就会跟上面例子说的一样，docker会自动帮你创建，但是这时候它创建的时候，就是目录了，而不是文件了</font>。
接下来尝试一下：
```html
[root@VM_156_200_centos ~]# vim web.list
[root@VM_156_200_centos ~]# cat web.list
1111111111
2222222222
```
我创建了一个 web.list 的文件，并且写入了两行，这时候就开始挂载这个文件了：
```html
[root@VM_156_200_centos ~]# docker run -it  -v /root/web.list:/root/web.list --name centos-demo-1  centos 
[root@6c91b4f6b765 /]# cat /root/web.list 
1111111111
2222222222
[root@6c91b4f6b765 /]# echo "333333" >> /root/web.list 
[root@6c91b4f6b765 /]# cat /root/web.list 
1111111111
2222222222
333333
[root@6c91b4f6b765 /]# exit
exit
```
可以看到挂载成功了，并且在容器内又在 web.list 写入新的一行,并退出容器
```html
[root@VM_156_200_centos ~]# cat /root/web.list 
1111111111
2222222222
333333
[root@VM_156_200_centos ~]# echo "444444" >> /root/web.list 
[root@VM_156_200_centos ~]# docker start -ai  centos-demo-1
[root@6c91b4f6b765 /]# cat /root/web.list 
1111111111
2222222222
333333
444444
```
可以看到在宿主机，之前容器新写的一行也同步过来了，而且之后宿主机也新加了一行，之后进入容器也一样有。
挂载文件跟上述挂载目录一样，也是默认为读写的，如果设置为只读的话，那么在容器里面就不能修改了，只能在宿主机修改：
```html
[root@VM_156_200_centos ~]# docker run -it  -v /root/web.list:/root/web.list:ro --name centos-demo-1  centos 
[root@2a7e3a4ed710 /]# echo "5555" >> /root/web.list 
bash: /root/web.list: Read-only file system
[root@2a7e3a4ed710 /]# exit
exit
[root@VM_156_200_centos ~]# echo "666" >> /root/web.list 
[root@VM_156_200_centos ~]# cat web.list 
1111111111
2222222222
333333
444444
666
```
可以看到在容器内编辑是会报错的，而在宿主机内就不会。
## 挂载数据卷容器
上述的挂载目录和挂载文件，只是 Docker Volume 的使用方式。
而我们用的方式就是在docker run命令后面跟上-v参数即可创建一个数据卷，然后把本地目录或者文件当做数据卷挂载到容器中。当然也可以跟多个-v参数来创建多个数据卷。
因此我们完全可以创建一个数据卷的容器，后面专门用来提供其他容器来挂载。
首先先创建一个专门用来提供数据的本地目录，然后挂载到一个普通的容器，而这个容器就是其他容器要挂载的数据卷容器，这种方式主要就是用来做多个容器的共享数据存在的。
```html
[root@VM_156_200_centos ~]# mkdir go-data
[root@VM_156_200_centos go-data]# cp -r  /root/go/src/goworker/ ./
[root@VM_156_200_centos go-data]# ls
goworker
```
go-data 这个目录存放一些 golang 语言的项目，然后我们把它做成一个数据卷容器，并指向容器的 /go/src 目录：
```html
[root@VM_156_200_centos go-data]# docker run -it  -v /root/go-data/:/go/src/ --name centos-go-data  centos 
[root@10d5c4f11e77 /]# cd /go/src
[root@10d5c4f11e77 src]# ls
goworker
[root@10d5c4f11e77 src]# exit
exit
```
这样这个就生成了一个名为 centos-go-data 的数据卷容器了，容器里面 /go/src 目录含有一个叫做 goworker 的 go 程序。刚好我服务器上之前有一个 go-1.10 环境的镜像。
接下来我们将这个镜像 run 起来，并且挂载 centos-go-data 这个数据卷容器（通过 --volumes-from 可以挂载数据卷容器），看看原来 /go/src 里面应该为空的目录，会不会有这个数据卷容器里面的go程序代码？
```html
[root@VM_156_200_centos go-data]# docker run -it --volumes-from centos-go-data --name golang-volume-demo kbz/golang-1.10
root@412ab80da79e:/app# cd /go/src 
root@412ab80da79e:/go/src# ls
goworker
root@412ab80da79e:/go/src# cd goworker/
root@412ab80da79e:/go/src/goworker# ls
hello_worker.go  worker  worker.go
```
可以看到原本应该有空的 src 目录，竟然有代码目录存在了，说明这个数据卷挂载成功了。接下来将这个 goworker 添加一个文件，看会不会同步到挂载的那个本地目录：
```html
root@412ab80da79e:/go/src/goworker# echo "123" > 123
root@412ab80da79e:/go/src/goworker# ls
123  hello_worker.go  worker  worker.go
root@412ab80da79e:/go/src/goworker# exit
exit
[root@VM_156_200_centos go-data]# cd /root/go-data/goworker/
[root@VM_156_200_centos goworker]# ls
123  hello_worker.go  worker  worker.go
```
发现本地目录也出现刚才在容器中出现的 123 文件了，说明还是共享成功了。 这时候我们再 run 一个新的 golang 环境的容器，然后也挂载这个数据卷看看数据会不会也有这个 123 文件？
```html
[root@VM_156_200_centos goworker]# docker run -it --volumes-from centos-go-data --name golang-volume-demo-2 kbz/golang-1.10
root@25d7a8fd5da1:/app# cd /go/src/goworker/
root@25d7a8fd5da1:/go/src/goworker# ls
123  hello_worker.go  worker  worker.go
```
发现新的容器也有这个 123 文件。
其实很多时候这种数据卷容器都是用来做持久化的，举个例子，比如我有一些golang的测试代码，做成了一个数据卷容器，然后我本地有好几个不同go环境的容器，每一个容器在 run 的时候，都会挂载这个数据卷容器。但是我们不希望在go环境的容器里面去修改这个数据卷容器里面的数据。所以刚开始我们在生成这个数据卷容器的时候，就要把权限设置为只读才行，这样这个数据卷就变成一个公共的挂载数据卷，别的容器只能挂载使用里面的数据，但是却不能做修改：
```html
[root@VM_156_200_centos goworker]# docker rm centos-go-data
centos-go-data
[root@VM_156_200_centos goworker]# docker run -it  -v /root/go-data/:/go/src/:ro --name centos-go-data  centos 
[root@851a09a40cb8 /]# exit
exit
[root@VM_156_200_centos goworker]# docker run -it --volumes-from centos-go-data --name golang-volume-demo-3 kbz/golang-1.10
root@c986955f5ee0:/app# cd /go/src/goworker/
root@c986955f5ee0:/go/src/goworker# ls
123  hello_worker.go  worker  worker.go
root@c986955f5ee0:/go/src/goworker# echo "456" > 456
bash: 456: Read-only file system
```
一旦进行修改，就会报错。 
而且这种数据卷容器可以挂好几个的，就跟用 -v 挂载多个本地目录一样：
```html
docker run --name data -v /opt/data1:/var/www/data1 -v /opt/data2:/var/www/data2:ro -it docker.io/ubuntu
```
这边稍微说一下，没有加-t和-i参数，所以这个容器创建好之后是没有进行交互模式的。其中 -i 表示以“交互模式”运行容器。 -t 表示打开一个连接到容器的终端。
```html
[root@VM_156_200_centos js-data]# docker run -it --volumes-from centos-go-data --volumes-from centos-js-data  --name golang-volume-demo-4 kbz/golang-1.10
root@94e9d5868711:/app# cd /go/src/ && ls
goworker
root@94e9d5868711:/go/src# cd /js && ls
test.js
```
这样就挂载了两个数据卷容器。
使用数据容器的两个注意点：
- 不要运行数据容器，这纯粹是在浪费资源。
- 不要为了数据容器而使用“最小的镜像”，如busybox或scratch，只使用数据库镜像本身就可以了。你已经拥有该镜像，所以并不需要占用额外的空间。

## 创建数据卷的其他方式
除了以上用 -v 来创建挂载数据卷之外，docker 还可以用以下方式来创建数据卷:
### 直接用docker volume 来管理
Docker 新版本中引入了 docker volume 命令来管理 Docker volume：
比如我创建一个名为 js-data 的数据卷：
```html
[root@VM_156_200_centos js-data]# docker volume create --name js-data
js-data
```
这样就创建成功了，可以通过 **docker volume ls** 来查看当前的数据卷：
```html
[root@VM_156_200_centos js-data]# docker volume ls
DRIVER              VOLUME NAME
local               0e01f8abe89c9956609eed063c623536112525fb32cf4c0363255a953aa2d35d
...
local               js-data
```
然后我们可以通过**docker volume inspect xxx**来查看这个数据卷所对应的本地目录是哪个？
```html
[root@VM_156_200_centos js-data]# docker volume inspect js-data
[
    {
        "CreatedAt": "2019-02-26T16:12:28+08:00",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/js-data/_data",
        "Name": "js-data",
        "Options": {},
        "Scope": "local"
    }
]
```
可以看到对应的本地目录就是 **/var/lib/docker/volumes/js-data/_data** 这个目录。当然目前这个目录是空的：
```html
[root@VM_156_200_centos js-data]# ll /var/lib/docker/volumes/js-data/_data
total 0
```
接下来就可以将这个数据卷挂载上去：
```html
[root@VM_156_200_centos js-data]# docker run -it -v js-data:/js  --name golang-volume-demo-3 kbz/golang-1.10
root@c55671066dec:/app# cd /js
root@c55671066dec:/js# ls
root@c55671066dec:/js# echo "123" > 123
root@c55671066dec:/js# ls
123
root@c55671066dec:/js# exit
exit
```
语法差不了多少，也是 -v 来指定，不过这时候“:”前面已经不是宿主机的目录了，而是换成 js-data 这个数据卷了，当然这个目录是空的。
然后我们给他添加了一个新的文件 123，看看宿主机目录会不会也有？
```html
[root@VM_156_200_centos js-data]# ll /var/lib/docker/volumes/js-data/_data
total 4
-rw-r--r-- 1 root root 4 Feb 26 16:29 123
[root@VM_156_200_centos js-data]# cat /var/lib/docker/volumes/js-data/_data/123
123
```
事实上肯定是有的。 而且这种方式其实就是上面那种**当宿主机的目录是相对目录的处理方式**，其实在docker处理过程中，如果宿主机目录是相对的，这时候就会去判断本地是否存在这个volume，如果不存在就创建一个(所以其实我们不太需要去显示的创建一个 volume)，所以我们还可以看到文章前面当**当宿主机的目录是相对目录的处理方式**那时候的相对目录的 hostTest1，其实已经是一个 volume了：
```html
[root@VM_156_200_centos js-data]# docker volume inspect hostTest1
[
    {
        "CreatedAt": "2019-02-26T11:30:34+08:00",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/hostTest1/_data",
        "Name": "hostTest1",
        "Options": null,
        "Scope": "local"
    }
]
```
但是有一点要注意的话，如果要指定具体的宿主机目录的话，那就不能用这种方式，“:” 前面的宿主机目录路径还是得用绝对路径。
### 通过dockerfile创建数据卷
除了用docker volume命令显示或者用 -v 宿主机相对目录隐示的创建数据卷之外，还可以在dockerfile设置volume数据卷。
ps： 注意上述**[5 挂载数据卷容器]**创建的数据卷容器,其实本质上是一个容器，即 Docker Container，而不是 Docker Volume。 Container 是可以在 **docker ps -a** 可以找到，而在 **docker volume ls** 这里面是找不到的：
```html
[root@VM_156_200_centos js-data]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                           PORTS                                                                        NAMES
a559685d18ae        centos              "/bin/bash"              About an hour ago   Exited (0) About an hour ago                                                                                  centos-js-data
851a09a40cb8        centos              "/bin/bash"              About an hour ago   Exited (0) About an hour ago                                                                                  centos-go-data
```
在 dockerfile 设置 volume 数据卷，举个例子：
```html
[root@VM_156_200_centos ~]# mkdir docker-volume-centos
[root@VM_156_200_centos ~]# cd docker-volume-centos/
[root@VM_156_200_centos docker-volume-centos]# vim dockerfile
[root@VM_156_200_centos docker-volume-centos]# cat dockerfile 
FROM centos
VOLUME /js-data
```
可以看到这个 dockerfile 挂载了一个容器内的 js-data。 接下来 build 一下：
```html
[root@VM_156_200_centos docker-volume-centos]# docker build --rm -t volume-centos .
```
然后 run 起来：
```html
[root@VM_156_200_centos docker-volume-centos]# docker run -it --name volume-centos-demo  volume-centos
[root@eeb532993285 /]# cd /js-data
[root@eeb532993285 js-data]# ls
[root@eeb532993285 js-data]# echo "456" > 456
[root@eeb532993285 js-data]# ls
456
[root@eeb532993285 js-data]# exit
exit
```
可以看到跑起来之后，容器里面已经有一个 js-data， 但是这个目录是空的，我们先在这个目录创建一个 456 的文件。然后退出容器。
接下来我们看看这个共享目录对应的宿主机的目录是哪个：
```html
[root@VM_156_200_centos docker-volume-centos]# docker inspect volume-centos-demo
[
    {
    ...
        "Mounts": [
            {
                "Type": "volume",
                "Name": "b07ab9cb039ffa90cc0ea7186716aa861cb9bcda11b1925bffca7a6539ee1304",
                "Source": "/var/lib/docker/volumes/b07ab9cb039ffa90cc0ea7186716aa861cb9bcda11b1925bffca7a6539ee1304/_data",
                "Destination": "/js-data",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            }
        ],
    ...
    }
]
```
可以看到宿主目录是这个：/var/lib/docker/volumes/b07...304/_data，接下来看看这个目录是否有 456 这个文件：
```html
[root@VM_156_200_centos docker-volume-centos]# ll /var/lib/docker/volumes/b07ab9cb039ffa90cc0ea7186716aa861cb9bcda11b1925bffca7a6539ee1304/_data
total 4
-rw-r--r-- 1 root root 4 Feb 26 16:56 456
[root@VM_156_200_centos docker-volume-centos]# cat /var/lib/docker/volumes/b07ab9cb039ffa90cc0ea7186716aa861cb9bcda11b1925bffca7a6539ee1304/_data/456 
456
```
是有的。 通过 dockerfile 设置 volume 也可以设置多个，不过这种方式都是相对目录的方式，而且文件名还随机，相当于上述的**-v 只指定一个的情况**：
```html
[root@VM_156_200_centos volumes]# docker run -it  -v /test2 --name centos-demo-1  centos 
```
## Volume 的删除和清理
如果是属于 volume，那么直接调用**docker volume rm xxx** 就可以删除了：
```html
[root@VM_156_200_centos docker-volume-centos]# docker volume ls
DRIVER              VOLUME NAME
...
local               js-data
[root@VM_156_200_centos docker-volume-centos]# docker volume rm js-data
js-data
```
如果是挂载的数据卷容器，就调用**docker rm xxx** 当做普通容器删除即可：
```html
[root@VM_156_200_centos docker-volume-centos]# docker rm centos-js-data 
centos-js-data
```
虽然容器删除了，但是宿主机挂载的本地目录的资料还是存在的。
## dockerfile volume 的权限和许可
这是个很有意思的现象，就是如果dockerfile里面的volume命令前后对共享目录进行操作的话，是会有差别的，举个例子：
```html
[root@VM_156_200_centos docker-volume-centos]# cat dockerfile
FROM centos
RUN useradd foo
VOLUME /data
RUN touch /data/x
RUN chown -R foo:foo /data
```
这个是一个dockerfile，我们的预期目录是，当run起来之后，会有 /data/x 这个文件。 但是实际上没有：
```html
[root@VM_156_200_centos docker-volume-centos]# docker build --rm -t kbz/volume-test .
[root@VM_156_200_centos docker-volume-centos]# docker run -it --name volue-test-demo kbz/volume-test
[root@3d037af9a5ca /]# cd /data
[root@3d037af9a5ca data]# ls
```
可以看到，虽然有 /data 这个目录，但是在目录里面并没有 x 这个文件？？？ 这个是怎回事呢？？？
这个就涉及到dockerfile的运行规则，在Dockerfile中的每个命令都会创建一个新的用于运行指定命令的容器，并将容器提交到镜像，每一步都是在前一步的基础上构建。
因此在Dockerfile中 **ENV FOO=bar**等同于：
```html
cid=$(docker run -e FOO=bar <image>)
docker commit $cid
```
那么针对我们本例的dockerfile，他的执行顺序应该是(真实过程可能不是这样的，但是应该差不多)：
```html
cid=$(docker run centos useradd foo)
image_id=$(docker commit $cid)
cid=$(docker run -v /data centos)
image_id=$(docker commit $cid)
cid=$(docker run $image_id touch /data/x)
image_id=$(docker commit $cid)
cid=$(docker run $image_id chown -R foo:foo /data)
docker commit $(cid) volue-test-demo
```
那这个过程就很明显了，因为每一行都会启动一个新的容器，因此每次都会有一个新的 volume，也就是每次run的新容器， /data 都是新的。
所以其实我们的这个 touch 操作，就是在当前的 volume 的 data 目录下操作的，而不是实际的容器或镜像的文件系统内。所以当我们重新再run的时候，这时候新的 volume 又是一个空的 data 目录。
所以如果要改的话，应该要把这个 touch 的操作，先存在镜像的文件系统之内，然后最后在挂载目录：
```html
[root@VM_156_200_centos docker-volume-centos]# vim dockerfile
[root@VM_156_200_centos docker-volume-centos]# cat dockerfile 
FROM centos
RUN useradd foo
RUN touch /data/x
RUN chown -R foo:foo /data
VOLUME /data
```
这个是改完后的 dockerfile
```html
[root@VM_156_200_centos docker-volume-centos]# docker build --rm -t kbz/volume-test-2 .
[root@VM_156_200_centos docker-volume-centos]# docker run -it --name volue-test-demo-2 kbz/volume-test-2
[root@2850978389ac /]# cd /data 
[root@2850978389ac data]# ls
x
```
这时候就有 x 文件了。 而且对应的宿主目录也有这个 x 文件了：
```html
[root@VM_156_200_centos docker-volume-centos]# cd /var/lib/docker/volumes/d4c83377d137104a13893b31e42c57e3635e8e59c14027463361b814c86c6ad7/_data
[root@VM_156_200_centos _data]# ls
x
```

---

参考资料：
[深入理解Docker Volume（一）](http://dockone.io/article/128)
[深入理解Docker Volume（二）](http://dockone.io/article/129)
[关于Docker目录挂载的总结](https://www.cnblogs.com/ivictor/p/4834864.html)
[Docker容器学习梳理--Volume数据卷使用](https://www.cnblogs.com/kevingrace/p/6238195.html)


