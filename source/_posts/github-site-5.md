---
title: github建站系列(5) -- 重新再换一个好看一点的模板
date: 2018-05-01 16:37:24
tags: github
categories: github建站系列
---
## 前言
终于过了两年了，还是没有写文章。不过今天终于觉悟了，要写文章了，结果打开界面一看，尼玛，界面太丑了。之前是 16年搭的， 尼玛现在是 18年了，有一些主题模板早就out了。

所以重新找了一个比较新的模板 [NEXT](https://github.com/iissnan/hexo-theme-next)

还有就是之前用的 node 的版本还是 4.2.2， 现在换成 9.5.0 版本，所以 hexo 要重新安装一下， 具体操作如下：
## 操作
因为后面有分为两个项目来存放，所以
### 1. 首先先把 kebingzao.github.io 和 blog_hexo 这两个项目都清空掉。
<!--more-->
### 2. 重新安装 hexo
```text
D:\wamp\www\checkTurnServer>npm install -g hexo-cli
C:\Program Files\nodejs\hexo -> C:\Program Files\nodejs\node_modules\hexo-cli\bin\hexo
npm WARN optional SKIPPING OPTIONAL DEPENDENCY: fsevents@1.2.3 (node_modules\hexo-cli\node_modules\fsevents):
npm WARN notsup SKIPPING OPTIONAL DEPENDENCY: Unsupported platform for fsevents@1.2.3: wanted {"os":"darwin","arch":"any"} (current: {"os":"win32","arch":"x64"})

+ hexo-cli@1.1.0
added 103 packages in 36.908s
```
### 3. 在 blog_hexo 所在的目录初始化
> 使用 hexo init <folder>

注意一个细节，就是执行 `hexo init .` 的时候， 目录一定要清空， 不然会报错
### 4. 生成静态页面 `hexo g`
### 5. 启动本地服务，进行文章预览调试 `hexo s`
```text
admin@admin-PC MINGW64 /f/airdroid_code/github/blog_hexo
$ hexo s
INFO  Start processing

INFO  Hexo is running at http://localhost:4000/. Press Ctrl+C to stop.
```
这时候可以看到页面了

![1](1.png)

### 6. 更换主题
我们换这个好看的主题： https://github.com/iissnan/hexo-theme-next, 安装主题的方法就是一句git命令：
```text
$ git clone --branch v5.1.2 https://github.com/iissnan/hexo-theme-next themes/next
Cloning into 'themes/next'...
remote: Counting objects: 12033, done.
```
安装完之后，就修改 根目录下的 _config.yml 将主题改成新下的这个
```text
## Themes: https://hexo.io/themes/
theme: next
```
后面如果我们要更新最新的 next 这个主题的版本，直接进入到 theme/next 这个目录然后，执行
```text
$ cd themes/next
$ git pull
```
即可

接下来重启一下，服务，就可以看到 界面已经改了

![1](2.png)

### 7 _config.yml 配置修改
接下来当然要在根目录下的 _config.yml 文件里面对站点的一些信息进行修改。 包括一些标题啊
```text
# Site
title: Zach Ke's Blog
subtitle:
description:
keywords:
author: Zach Ke
language:
timezone:

# URL
## If your site is put in a subdirectory, set url as 'http://yoursite.com/child' and root as '/child/'
url: http://kebingzao.com
root: /
permalink: :year/:month/:day/:title/
permalink_defaults:
```
然后 最后执行 hexo server -g 重新跑  （-g 说明要先执行 hexo g，再执行 hexo s）
```text
admin@admin-PC MINGW64 /f/airdroid_code/github/blog_hexo
$ hexo s -g
INFO  Start processing

INFO  Hexo is running at http://localhost:4000/. Press Ctrl+C to stop.
```
### 8 更新到线上去
因为已经编译到静态文件了，因此可以直接push到github上面去，用 `hexo deploy` 来 push 到远程， 首先要安装 `hexo-deployer-git`
```text
npm install hexo-deployer-git --save
```
_config.yml 修改配置:
```text
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repo: git@github.com:kebingzao/kebingzao.github.io.git
```
这时候可以看到已经push上去了。
```text
admin@admin-PC MINGW64 /f/airdroid_code/github/blog_hexo
$ hexo deploy
INFO  Deploying: git
INFO  Setting up Git deployment...
Initialized empty Git repository in F:/airdroid_code/github/blog_hexo/.deploy_git/.git/

[master (root-commit) 6352912] First commit
```
这时候就会将 blog_hexo 项目的 public 里面的内容 push 到 kebingzao.github.io 这个资源库了, 接下来只要将 blog_hexo 的新代码重新提交即可, 然后将 kebingzao.github.io 的 readme 和 cname 文件补上去就行了， 但是这样发现会有一个问题：

就是如果只是将 readme 和 cname 文件上传到 kebingzao.github.io 这个项目的话。但是这样一来，我每次进行 `hexo deploy` 的时候，就会把 kebingzao.github.io 这个项目的文件全部覆盖。因此下次这两个文件又没了。 所以解决的办法就是将 readme 和 cname 文件上传到 `blog_hexo` 项目的 public 里面。 然后调用 `hexo deploy` 之前，先调用  `hexo g` 生成静态资源。 最后随着 deploy 一起放到 kebingzao.github.io 这个资源库。 这样子这两个文件就会跟着一起上传。

<font color=red>不过这边要注意一个细节，就是不要用 `hexo clean`，不然 public 目录会被清掉重置, 里面的 readme 和 cname 文件就会没有了。 如果真的迫不得已用了 `hexo clean`， 那么这两个文件还要手动补到 public 上，然后再重新生成静态资源，最后再上传</font>

## 总结
终于换了一个好看的主题了，是不是可以写文章了。

---
github 建个人站点系列文章:
{% post_link github-site-1 %}
{% post_link github-site-2 %}
{% post_link github-site-3 %}
{% post_link github-site-4 %}
{% post_link github-site-5 %}
{% post_link github-site-6 %}
{% post_link github-site-7 %}
{% post_link github-site-8 %}
{% post_link github-site-9 %}
{% post_link github-site-10 %}
{% post_link github-site-11 %}
{% post_link github-site-12 %}
{% post_link github-site-13 %}
{% post_link github-site-14 %}
{% post_link github-site-15 %}
{% post_link github-site-16 %}
{% post_link github-site-17 %}





