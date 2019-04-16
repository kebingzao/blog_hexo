---
title: gitlab 生成并使用 ssh keys
date: 2019-04-16 19:52:02
tags: git
categories: git 操作
---
## 前言
我们代码托管用的是 gitlab，但是发现在 IDE 中，在 push 的时候，需要输入 gitlab 的密码？
![1](1.png)
<!--more-->
提示 ssh key 不存在，因此要重新生成 ssh key， 他这边有教程：
![1](2.png)
使用 git 工具打开，然后一直输入这个命令：
```html
ssh-keygen -t rsa -C "youremail@mail.com"
```
然后一直按回车，最后就会生成两个文件。
![1](3.png)
找到该文件，用记事本打开第二个文件，然后全选复制里面的内容：
![1](4.png)
然后打开 gitlab 的 profile 设置：
![1](5.png)
添加一个 SSH key：
![1](6.png)
最后点击生成：
![1](7.png)
一台电脑，只能生成一个profile文件，可以把旧的去掉，这样旧电脑就不能用这个了，这样就可以了。

