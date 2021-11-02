---
title: 记一次 go module 因为内部公共包使用 replace 导致当前程序拉取公共包最新版本失败的情况
date: 2021-11-02 13:12:12
tags: 
- golang
categories: golang相关
---
## 前言
事情是这样子的， 我们现在 golang 版本是 `1.13`, 并且采用了 Go Module 作为第三方包管理工具: {% post_link go-module %}, 正常情况下我们有一个内部的公共库，比如叫 igong， 存放一些公共抽象出来的方法， 供各个程序去调用。

有一次，我正常给 igong 包打 tag 之后，tag 版本号为 `v1.8.7`, 这时候我们的具体程序去更新这个 igong 包的时候， 却报错了:

```text
F:\example_code\gopush-air>go get -u gitlab.example.com/example-utils/iGong
go: downloading gitlab.example.com/example-utils/iGong v1.8.7
go: extracting gitlab.example.com/example-utils/iGong v1.8.7
go get: gitlab.example.com/example-utils/iGong@v1.8.7 requires
        github.com/qiniu/bytes@v0.0.0-00010101000000-000000000000: invalid version: git fetch -f https://github.com/qiniu/bytes refs/heads/*:refs/heads/* refs/tags/*:refs/tags/* in F:\example_code\go\pkg\mod\cache\vcs\9343a7564b
b508f7f517b255feef5381fea307ce6fb5426727e018b089363563: exit status 128:
        fatal: could not read Username for 'https://github.com': terminal prompts disabled

```

发现在更新这个 igong 包版本的时候， 获取某一个依赖的时候， 一直报错。 好像一直去下载一个不存在的资源 `https://github.com/qiniu/bytes`， 这个包在 github 上已经找不到了。

后面去 igong 包看了一下，发现 igong 项目的 go.mod 确实是有间接引用这个包，不过之前是因为这个包已经不在 github 上面了， 所以就用 replace 将下载地址引入到我们自己的内部私有库，让其可以下载并应用:
```text
replace github.com/qiniu/bytes => gitlab.example.com/qiniu/bytes v0.0.0-20191012100200-92558a444c07
```

但是问题就出现在这里， 如果作为第三方被引用的包 igong 里面还有 replace 语法去对一些已失效的包的下载地址进行替换的话， 那么在这个第三方 igong 被其他程序导入的时候， 这时候 go module 进行下载的时候， 好像不会再去 参照 igong 的 go.mod 的 replace 语法。 而是只按照 igong go.mod 的依赖去进行下载， 所以就会导致下载失败的情况。 我不知道 golang 更高一点的版本有没有修复， 但是在我使用的 `1.13` 版本，确实存在这个 bug， 所以解决方法就是将 igong 的 go.mod 的这个 replace 语句，也一样拷贝一份到我们的当前程序，这样子就可以了。 当然最好的解决方式，就是内部公共包最好不要使用 replace 语句

举个例子:

```text
当前程序 A , 引用 内部公共库 B， 内部公共库的 go.mod 使用了 replace 语句去替换一些下载链接失效的依赖包 C ->replace-> D， 最后其实下载的是 D 包

这时候 A 去更新 B 的版本的时候， 会去下载 C 包， 但是 C 包已经不存在了。 所以就报错了， 所以这时候 B go.mod 的这一句 replace 
C ->replace-> D
也要添加到 A 的 go.mod， 这样子 A 引用 B 的时候，才能正常更新 B 的版本
```







