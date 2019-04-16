---
title: 同一台 PC 使用 SSH key 管理 gitlab 和 github 的提交
date: 2019-04-16 20:07:00
tags: git
categories: git 操作
---
## 前言
通过 {% post_link gitlab-ssh %} 我们就在 gitlab 上用 ssh key 去进行命令行提交代码了。但是有时候会有一些个人项目也在这台PC上，但是这时候传的就不是项目的内部私有代码库gitlab了。而是 github。
那么要怎么在同一台 PC 同时对 gitlab 和 github 进行不同项目的代码 git 操作呢，比如 push 之类的？
<!--more-->
## 生成 github 的 ssh key
如果直接用 **ssh-keygen** 生成 github 的 ssh key的时候，会发现覆盖 gitlab 之前生成的 ssh key，因此在创建新的 ssh key 的时候，要重新指定一个新的文件名：
![1](1.png)
创建完成之后
![1](2.png)
多了两个，这时候要创建一个 config 文件来分别指配：
![1](3.png)
内容是：
```html
# gitlab
Host gitlab.airdroid.com
    HostName gitlab.airdroid.com
    PreferredAuthentications publickey
    IdentityFile C:/Users/admin/.ssh/id_rsa

# github
Host github.com
    HostName github.com
    PreferredAuthentications publickey
    IdentityFile C:/Users/admin/.ssh/id_rsa_github
```
这样就会分别指派过去了。因为之前 gitlab 已经把 ssh key 配好了，所以就不用管了。接下来只要把 id_rsa_github.pub 里面的内容 放到 github 的 ssh key 里面就可以了(就是本地config里面指定的是私钥，github ssh key那边填的是公钥)。当完成之后，就可以尝试连接了：
```html
ssh -T git@github.com
```
![1](4.png)
这个时候，要注意一点，就是要允许请求连接。 同时也可以试试看连接 gitlab.airdroid.com
![1](5.png)
这样子 github 就配好了。接下来就是上传项目到 github 上面去了。
注意，这时候，如果该项目还不是git项目，就要先转化为git 项目：
```html
git init > git add --all  > git commit -m 'first commit'
```
接下来将git项目传到 github (该项目要已经建立了), 接下来执行 
```html
git remote add origin git@github.com:kebingzao/node-test-browserify.git
```
但是出错:
![1](6.png)
原来是之前测试的时候，已经添加过一次了，先把旧的删掉，再执行一次就可以了。
![1](7.png)
接下来就push上去。
```html
git push -u origin master
```
![1](8.png)
结果出错了，原因就是 本地库和远程库的资源不一致导致的，因为在github新建项目的时候，多了一个readme文件，这是本地库没有的。因此要先把远程库的资源先拉下来，再推送:
```html
git pull origin master
```
![1](9.png)
这样就没有问题了。打开github 上看一下， 东西已经上去了

