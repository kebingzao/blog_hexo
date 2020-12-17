---
title: github建站系列(9) -- 写文章的时候，插入图片
date: 2018-05-07 16:37:24
tags: github
categories: github建站系列
---
## 前言
虽然可以写文章了，但是怎么插入图片呢，或者说怎么插入本地项目的图片呢? 后面找了一下，有篇文章 [hexo生成博文插入图片](https://blog.csdn.net/sugar_rainbow/article/details/57415705) 讲的还可以。所以就按照他的方式试了一下。

## 操作
还是一样几个步骤
1. 把主页配置文件 `_config.yml` 里的 `post_asset_folder:` 这个选项设置为 `true`
2. 在你的 hexo 目录下执行这样一句话 `npm install hexo-asset-image --save`，这是下载安装一个可以上传本地图片的插件
<!--more-->

```text
F:\code\github\blog_hexo>npm install hexo-asset-image --save
npm WARN optional SKIPPING OPTIONAL DEPENDENCY: fsevents@1.2.3 (node_modules\fsevents):
npm WARN notsup SKIPPING OPTIONAL DEPENDENCY: Unsupported platform for fsevents@1.2.3: wanted {"os":"darwin","arch":"any"} (current: {"os":"win32","arch":"x64"})

+ hexo-asset-image@0.0.3
added 13 packages in 15.65s
```
3. 等待一小段时间后，再运行`hexo new "xxxx"`来生成md博文时，/source/_posts文件夹内除了 xxxx.md 文件还有一个同名的文件夹
```text
F:\code\github\blog_hexo>hexo n ie-png-fliter
INFO  Created: F:\code\github\blog_hexo\source\_posts\ie-png-fliter.md
```

接下来把图片放进去

![1](1.png)

然后这样调用就可以了

```text
![png](ie-png-fliter/angry.jpg)
```

这样就有效果了。

![1](2.png)

## 总结
图片可以插入了， 接下来我将为 blog 增加搜索功能

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

