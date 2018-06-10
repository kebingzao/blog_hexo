---
title: git 回退已经merge过但是没有提交到远端仓库的分支
date: 2018-05-03 20:20:24
tags: git
categories: git 操作
---
之前在协作的时候，有发生过一种情况：
有个同事有在他的分支上做了一个新的接口。但是我用dev合并的时候，发现这个分支是基于master分支，导致代码是比较旧的。
这时候，merge 后的代码还没有提交到远端仓库。
所以我是直接抛弃掉这个merge操作。
命令就是: 

{% codeblock lang:git %}
git reset --hard HEAD~1
{% endcodeblock %}

这时候就会回到了dev最新的分支了，并且没有任何的commit了