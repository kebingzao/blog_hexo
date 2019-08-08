---
title: 使用git log来区分两个分支的差异
date: 2019-08-08 14:36:46
tags: git
categories: git 操作
---
## 场景
有时候我们在开发的时候，尤其是多个人协同开发的时候，有时候某一个功能在 develop 分支，然后这时候要合并到 master 分支。这时候我们就会想要知道当前的 develop 分支和 master 分支到底有没有差异。如果有差异，并且差异就只是你要合并的内容，那就直接 merge 到 master 就行了，但是如果差异有很多的 commit，并且这些 commit 还包含其他人的提交，并且这些提交当下还不能合并到 master， 那么这时候是不能直接 merge 的。 这时候要么就把你的 commit 一个一个用 cherry-pick 拷贝过去，要么就单独对这几个 commit 做成一个 patch，然后再把 patch 合并到 master。
## 对比两个分支的不同
```html
git log develop ^master
```
<!--more-->
通过以上的命令，可以查看在 develop 分支上，但没有在 master 上的 commit。当然这个只是本地的，如果要比对线上的，那么就加 origin：
```html
git log origin/develop ^origin/master
```
这个是在有差异的 commit 的情况下
```html
admin@admin-PC MINGW64 /f/airdroid_code/id-airdroid-com (master)
$ git log develop ^master
commit 8e5d29cecf81d5a559a1f7c75900c1091e8fdc68 (HEAD -> develop, origin/develop)
Author: kbz <xxx@gmail.com>
Date:   Thu Aug 8 14:23:30 2019 +0800

    修改 日志 备份配置，p20 保留 session

```
如果没有差异的话，其实就是空
```html
admin@admin-PC MINGW64 /f/airdroid_code/id-airdroid-com (master)
$ git log origin/develop ^origin/master

```
## 备忘
之前用这个指令的时候，发现在 IDEA 这个 IDE 自带的 Terminal 窗口会有问题。明明 develop 和 master 完全一样，但是就是会有一堆的 commit 出来。后面用 git 自带的 shell terminal 窗口，才没问题。


