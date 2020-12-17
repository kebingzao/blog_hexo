---
title: github建站系列(3) -- 使用hexo创建个人blog主页
date: 2016-05-19 15:37:24
tags: github
categories: github建站系列
---
## 前言
通过 {% post_link github-site-2 %} 可以看到个人blog的主页已经建起来了，接下来就是搞一个比较好的模板。

这时候有两个选择，一个是 jekyll ， 一个是 hexo， 其中 jekyll 是用ruby，而 hexo 用的 nodejs。而且无论是从速度还是简易程度，hexo 都比 jekyll 好。因此这次是用 hexo 来搭建blog。

## 安装
hexo 需要 node环境和 git 环境， 直接全局安装
```text
npm install -g hexo
```
这样就全局装成功了 （建议翻墙，因为第一次装的时候没有翻墙，结果失败）
<!--more-->
## 在所在的目录初始化
> 使用 hexo init <folder>

因为 git bash 的窗口已经在 kebingzao.github.io 这个项目中了，因此这时候不需要再指定目录，而是直接在当前目录中建
 （这时候可以把之前的那个旧的index删掉了）: `hexo init .`

> 这个过程会比较久，因为下载的资源比较多（包括需要的node_modules），所以可能要两三分钟

![1](1.png)

这样就成功初始化项目了。

## 生成静态页面
```text
hexo generate
```
也可以用 `hexo g`

![1](2.png)

## 启动本地服务，进行文章预览调试
```text
hexo server
```
也可以用 `hexo s`

![1](3.png)

默认打开 4000 端口。 http://localhost:4000/

![1](4.png)

这样基本框架就搭好了。

## 更换主题
安装主题的方法就是一句git命令：
```text
git clone https://github.com/wuchong/jacman.git themes/jacman
```

![1](5.png)

这时候还要再改 配置文件里面的 theme 参数。 修改当前根目录下的config.yml配置文件中的theme属性，将其设置为jacman。
```text
theme: jacman
```

![1](6.png)

接下来重新生成静态文件和开启服务
```text
hexo g
hexo s
```
然后刷新一下：

![1](7.png)

发现已经变回来了，说明主题更新成功

## _config.yml 配置修改
接下来当然要在根目录下的 _config.yml 文件里面对站点的一些信息进行修改。 包括一些标题啊，作者的头像啊之类的, 其中作者的头像要在 themes/source/img 里面改，即替换 author.jpg

![1](8.png)

最后执行 `hexo server -g` 重新跑  （`-g` 说明要先执行 `hexo g`，再执行 `hexo s`）

![1](9.png)

可以看到标题改过来了, 底下的头像也改过来了

![1](10.png)

## 更新到线上去
因为已经编译到静态文件了，因此可以直接 push 到 github 上面去， 也可以用 hexo deploy 来部署，不过这个需要配置。 
```text
https://hexo.io/zh-cn/docs/deployment.html
```
先用第一种先试试，发现上传上去了，根本不行。因此只能用 hexo deploy 来 push 到远程

首先要安装 `hexo-deployer-git`
```text
npm install hexo-deployer-git --save
```
接下来在 _config.yml 修改配置

![1](11.png)

然后执行命令 hexo deploy

![1](12.png)

这时候可以看到已经push上去了。

![1](13.png)

可以看到，它是把 kebingzao.github.io 的 `.deploy_git` 里面的文件传上去，而之前存在的文件都不在了。

![1](14.png)

这时候查看 http://kebingzao.github.io/

![1](15.png)

发现界面倒是出来了。但是好像哪里不对，因为之前配的 _config.yml 都无效了， 而且底下的图片，还是原来的那个。 发现原项目的文件变成静态编译后的文件了，直接被覆盖了。

所以只用一个项目是处理不了的，会相互覆盖， 于是我把这个重新分成两个项目。
- 一个是 <b>git@github.com:kebingzao/kebingzao.github.io.git</b> 这个就是 `hexo d` 上传的那个项目， 即blog主页的代码。
- 一个是 <b>git@github.com:kebingzao/blog_hexo.git</b>，这个就是使用 hexo 的代码。

两者的关系，就是在 `blog_hexo` 项目使用 `hexo d` 部署的时候，就会把生成的 `.deploy_git` 里面的文件上传到 <b>git@github.com:kebingzao/kebingzao.github.io.git</b> 这个项目。 相当于一个是源代码，一个是静态编译后的代码。 也就是说我们后续都不需要去管 <b>git@github.com:kebingzao/kebingzao.github.io.git</b> 这个库的代码， 只要编写 hexo 的代码就行了, 然后编写完，本地没问题之后，通过 `hexo d` 去更新 blog 项目的代码。

这边需要注意一点的是， 要在 `blog_hexo` 的 `_config.yml` 配置 deploy 地址为 `kebingzao.github.io` 这个项目。

![1](16.png)

而且这边需要注意的一个细节是。 有时候会出现在本地跑的好好的。但是一旦使用 `hexo d` 上传到线上。就是旧的代码。 这时候就要使用 `hexo clean` 清理一下缓存，然后再使用 `hexo s -g`, 最后再 `hexo d` 部署到线上去，这时候就是最新的了。

![1](17.png)

![1](18.png)

这时候可以看到，已经是最新的了

## 总结
blog 主页搞好了，接下来绑定域名

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

