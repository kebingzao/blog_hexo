---
title: Gitlab 强制推送提示 "You are not allowed to force push code to a protected branch on this project."
date: 2019-04-18 16:21:53
tags: git
categories: git 操作
---
## 前言
之前有出现过一个情况，就是 master 分支有改了一些东西，但是发现这些东西有问题，这时候要回滚到 master 的某一个 commit 版本：
```html
git rest --hard xxxx
```
这时候就将这个分支回滚到之前的某一个 commit 了。 但是这时候直接 
```html
git push origin
```
是不行的。 会显示远端的版本比当前版本高，所以只能用 **-f** 来强制 push 上去。
```html
git push -u origin master -f
```
但是发现还是报错：
<!--more-->
```html
You are not allowed to force push code to a protected branch on this project
```
信息提示我无法强制 push 代码到一个受保护的分支？？ 哪怕我已经是 master 了，还是强推不了？？ 
## 解决
后面去 gitlab 的该项目的配置项看了一下(在 Settings 的 Repository 设置项的 Protected Branches)， 原来这个项目中有对 master 分支做了 protected 保护， 不允许强制推送，这个好像是项目创建的时候就默认的设置：
![1](1.png)
所以如果要强制推送到 master 的话，这边要先取消掉 protected 分支。所以点击  Unprotect 按钮：
![1](2.png)
这时候就没有保护分支了， 然后这时候强制推送就成功了。 然后为了安全，我们再重新设置为保护分支
![1](3.png)
这样就恢复成原来的样子了。
## 后记
虽然在我的 pc 上已经将这个 master 分支回滚了。 但是后面发现在我的另一个同事的 IDE 上，master 分支 git pull 之后，竟然还是旧的代码，而不是之前回滚之后的那个版本？？
```html
F:\airdroid_code\go\src\auto_pack>git pull
Already up to date.
```
而且已经没办法再 pull 任何东西了，看了一下 gitlab ，发现 master 分支的代码确实是回滚之后的代码了。所以我怀疑有可能是 master 分支的 commit 版本太新了， 比当前线上的 master 的 commit 版本还新，所以才会 pull 不到任何东西。
所以解决方法，就是先把这个分支回滚到某一个更旧的 commit ，然后再 git pull ，这样才有东西：
```html
F:\airdroid_code\go\src\auto_pack>git reset --hard 3dfc6e8d
HEAD is now at 3dfc6e8 readme

F:\airdroid_code\go\src\auto_pack>git pull
Updating 3dfc6e8..ad9e3f2
Fast-forward
main.go   | 18 +++++++++++++++---
model.go  |  9 ++++++++-
upload.go |  5 ++++-
3 files changed, 27 insertions(+), 5 deletions(-)
```
果然这样就可以了。
所以结论就是，本地的分支 commit 如果比线上回滚后的分支 commit 还要新的话，那么 git pull 是拉不到任何东西的，所以就要先回滚到比线上的分支还旧的 commit ，然后再 git pull。


