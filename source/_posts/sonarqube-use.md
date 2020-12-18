---
title: windows 环境下使用 SonarQube 来检测工程代码质量
date: 2020-12-18 10:26:42
tags: sonarqube
categories: 实用工具集
---
## 前言
做过开发的都知道，虽然都是可以 work，但是代码质量的好坏， 直接会导致这个项目的可维护性和可扩展性。 虽然我们的项目的代码在上线之前都会经过测试和 code review。但是随着项目数量越来越多，team leader 在进行 code review 的时候，往往只能注重代码的可用性，很多代码质量上的问题都被忽略了。 所以项目组决定在进行代码 patch 提交之前，在 Jenkins 构建之前，先通过代码质量检测工具 SonarQube 来检查一下代码质量。 这样子可以保证在最后入 master 分支的时候， code review 阶段可以尽可能去看这些格式优雅的代码。

## SonarQube 简介
[SonarQube](https://www.sonarqube.org/) 是一款用于代码质量管理的开源工具，它主要用于管理源代码的质量。 通过插件形式，可以支持众多计算机语言，比如 java, C#, go，C/C++, PL/SQL, Cobol, JavaScrip, Groovy 等。sonar可以通过PMD,CheckStyle,Findbugs等等代码规则检测工具来检测你的代码，帮助你发现代码的漏洞，Bug，单元测试覆盖率等信息。

Sonar 不仅提供了对 IDE 的支持，可以在 Eclipse和 IntelliJ IDEA 这些工具里联机查看结果；同时 Sonar 还对大量的持续集成工具提供了接口支持，可以很方便地在持续集成中使用 Sonar。 
<!--more-->
## SonarQube 安装
因为我的环境是 windows 7，所以接下来的整个安装环境都是在 windows 7 来完成的。 接下来的安装过程是从 0 到 1 的，然后遇到问题，一步一步踩坑解决的。 而不是等我最后安装使用成功了，再以上帝视角来描述。 因为那样子会忽略很多安装过程中的问题，导致对一些想使用 SonarQube 的新手来说会不够友好。

### 1. 下载 SonarQube
首先到 [官网下载页面](https://www.sonarqube.org/downloads/) 下载免费版，也就是 Community 版本， 因为我们的主要项目语言，主要是 JavaScript， PHP, Golang，刚好它都有支持

![1](1.png) 

截止到 `2020-12-18`, 最新版本是 8.6, 所以我们直接下载， 下载完之后，直接解压， 进入到 bin 目录， 接下来选择 `windows  x86` 下的 bat 文件

![1](2.png) 

双击这个 `StartSonar.bat` 文件，进行安装

![1](3.png) 

从 log 上来看，发现需要 JVM 的支持，所以我们需要去下载一个， 而且要 jdk 11+ 的版本。

### 2. 下载并安装 jdk 11
接下来到官网 [官网 Java SE 下载页面](https://www.oracle.com/java/technologies/oracle-java-archive-downloads.html), 然后直接选择一个 Java SE 11 的版本

![1](4.png) 

![1](5.png) 

这边注意一个细节，现在下载 Java SE 的话，要登录账号才行，如果没有的话，就去注册一个就行了。 下载完成之后，就开始进行安装

![1](6.png) 

安装好了，直接验证一下，是否有自动写到环境变量中:

![1](7.png) 

看起来是有了， 所以就不再需要再去配置环境变量了。

接下来重新执行 `StartSonar.bat` 文件，这时候就可以看到有加载到 JVM 了
> 我的电脑比较特殊，还是会加载失败，不过我重启电脑之后，就可以加载到了，不过一般的电脑应该不需要重启，就可以生效了

![1](8.png) 

首次启动可能会久一点，耐心等待一下，一直到看到 `SonarQube is up` 的字样，说明服务已经启动了。

![1](9.png) 

### 3. 登入 web 界面
既然服务已经启动了，这时候访问本地的 localhost:9000 就可以看到界面了

![1](10.png) 

需要登录，默认的用户名和密码就是admin，admin。 首次登录之后，会需要你重新修改密码。 第一次登录进去之后，界面是这样子

![1](11.png) 

默认语言是英语， 不过有汉化包， 我们下载一下 汉化包

### 4. 下载汉化包
具体步骤如截图:

![1](12.png) 

点击安装。 最后重启一下 server

![1](13.png) 

等一会儿让你重新登录， 过一会儿就变中文界面了。 但是神奇的是，我重新登录之后，界面还是没有变化，还是英文？ 懵逼了？ 啥姿势不对？

后面查了一下，我的版本是 8.6 的，但是汉化包最新版本只支持到 8.5 的版本，所以我下载到的汉化包版本也是 8.5 的，暂时还没有 8.6 的汉化包

![1](14.png) 

原来版本太新也有问题。 没事，用英语界面也是棒棒的。
> 值得一提的是，隔了一天，我看到 release 版本变成支持 8.6 的版本了，本来以为可以下到 8.6 的汉化包了，结果查询了一下这个插件包，还是只有 8.5 的包，看来这两个姿势也不统一啊，估计还来不及上架，后面再等等吧

## 实操验证
### 1. 新建项目
既然安装好了，接下来就拿几个项目实际遛一遛才知道是否有效果。因为我们的主要项目语言就是 JavaScript， PHP, Golang。 所以先拿一个 PHP 项目测一下。

首先先创建一个项目，名称随便取

![1](15.png) 

然后接下来会让你生成一个 token， 后面扫描代码做校验的， 这个也是随便填没事，反正后面扫码也是直接贴指令，没有谁会去记这一串鬼东西。

![1](16.png) 

接下来会让我们选择扫描项目的语种，PHP 是在 other 类别里面 (JavaScript 和 Golang 也同样是这个类别)， 然后平台选择 windows

![1](17.png) 

### 2. 下载扫描组件
因为是第一次，所以得先下载一个扫描工具，点击 download 按钮，会跳转到一个下载页面，选择 `windows 64-bit 版本即可`

![1](18.png) 

点击下载， 然后下载完之后，解压， 将 这个目录的 bin 放到环境变量

![1](19.png) 

![1](20.png) 

### 3. 在项目目录下执行扫码命令
扫描组件安装完之后，接下来就 copy 下面的那一长串代码，然后放到对应的项目 (本次是 php 项目) 的根目录下执行

![1](21.png) 

不过好像报错了:
```text
F:\airdroid_code\lumen-pay>sonar-scanner.bat -D"sonar.projectKey=lumen-pay" -D"sonar.sources=." -D"sonar.host.url=http://localhost:9000" -D"sonar.login=54822b97fdce00358a0747dc84b88a169d2e61a3"
INFO: Scanner configuration file: D:\sonar-scanner-4.5.0.2216-windows\bin\..\conf\sonar-scanner.properties
INFO: Project root configuration file: NONE
INFO: SonarScanner 4.5.0.2216
INFO: Java 11.0.3 AdoptOpenJDK (64-bit)
INFO: Windows 7 6.1 amd64
INFO: User cache: C:\Users\admin\.sonar\cache
INFO: Scanner configuration file: D:\sonar-scanner-4.5.0.2216-windows\bin\..\conf\sonar-scanner.properties
INFO: Project root configuration file: NONE
INFO: Analyzing on SonarQube server 8.6.0
INFO: Default locale: "zh_CN", source code encoding: "GBK" (analysis is platform dependent)
INFO: Load global settings
INFO: Load global settings (done) | time=86ms
INFO: Server id: BF41A1F2-AXZv_C5uGbK98_9nLrOT
INFO: User cache: C:\Users\admin\.sonar\cache
INFO: Load/download plugins
INFO: Load plugins index
INFO: Load plugins index (done) | time=43ms
INFO: Plugin [l10nzh] defines 'l10nen' as base plugin. This metadata can be removed from manifest of l10n plugins since version 5.2.
INFO: Load/download plugins (done) | time=104ms
INFO: Process project properties
INFO: Process project properties (done) | time=11ms
INFO: Execute project builders
INFO: Execute project builders (done) | time=2ms
INFO: Project key: lumen-pay
INFO: Base dir: F:\airdroid_code\lumen-pay
INFO: Working dir: F:\airdroid_code\lumen-pay\.scannerwork
INFO: Load project settings for component key: 'lumen-pay'
INFO: Load project settings for component key: 'lumen-pay' (done) | time=24ms
INFO: Load quality profiles
INFO: Load quality profiles (done) | time=54ms
INFO: Load active rules
INFO: ------------------------------------------------------------------------
INFO: EXECUTION FAILURE
INFO: ------------------------------------------------------------------------
INFO: Total time: 2.198s
INFO: Final Memory: 5M/24M
INFO: ------------------------------------------------------------------------
ERROR: Error during SonarScanner execution
java.lang.IllegalStateException: Unable to load component class org.sonar.scanner.report.ActiveRulesPublisher
        at org.sonar.core.platform.ComponentContainer$ExtendedDefaultPicoContainer.getComponent(ComponentContainer.java:66)
        at org.picocontainer.DefaultPicoContainer.getComponent(DefaultPicoContainer.java:621)
        ...
        ... 50 more
ERROR:
ERROR: Re-run SonarScanner using the -X switch to enable full debug logging.
```
后面网上查了一下，解决方法是
```text
Delete the directory data/es in your SonarQube installation.
Restart SonarQube.
```
先把 data 下面的 es7 这个目录删掉

![1](22.png)

然后再重启 SonarQube 服务 (terminal 窗体执行 `ctrl + c` 退出， 然后重新执行 .bat 文件即可)。 这一次终于跑成功了
```text
18:28:38.133 INFO: Note that you will be able to access the updated dashboard once the server has processed the submitted analysis report
18:28:38.134 INFO: More about the report processing at http://localhost:9000/api/ce/task?id=AXZwPpQRUZLk_accn7bS
18:28:38.136 DEBUG: Post-jobs :
18:28:38.143 INFO: Analysis total time: 2:06.523 s
18:28:38.146 INFO: ------------------------------------------------------------------------
18:28:38.147 INFO: EXECUTION SUCCESS
18:28:38.147 INFO: ------------------------------------------------------------------------
18:28:38.147 INFO: Total time: 2:08.729s
18:28:38.211 INFO: Final Memory: 7M/30M
18:28:38.214 INFO: ------------------------------------------------------------------------
```
这时候刷新页面，就有东西了

![1](23.png)

看起来比想象中。好一点。 是 passed， 那就应该还好。  接下来就是针对这些有问题的地方，去进行调整和优化。 回到首页，就可以看到

![1](24.png)

而且他还有安全警告的功能， 比如那三个安全警告，点进去看的时候

![1](25.png)

发现有两个是 md5 的调用， 还有一个是 伪随机参数，都是不够安全的因素(这种方式都可以被 hash 碰撞)，所以就标出来。

![1](26.png)

至于这个要不要改，得看所对应的业务的安全性和影响性，如果是影响比较大的，那么最好改一下。

### 4. 扫描生成的缓存文件
在扫描的时候，会在两个地方生成缓存文件。因为它每一次扫描都不是全部代码再扫描一次，而是基于上次的扫描范围，对新的修改的代码再进行扫描分析，所以肯定会有一些缓存文件的。

项目里面就会有一个 `.scannerwork` 的目录, 里面存放一些扫描的 report 报告

![1](27.png)

同时在 `C:\Users\admin\.sonar\cache` 也会有缓存

![1](28.png)

### 5. 扫码 golang 项目
接下来我们跑一下 golang 项目的扫描检测，一样创建一个 golang 项目的 project， 然后生成 token 之后，直接将下面的 指令 拷贝到项目目录下，开始执行
```text
INFO: ANALYSIS SUCCESSFUL, you can browse http://localhost:9000/dashboard?id=golang-resque
INFO: Note that you will be able to access the updated dashboard once the server has processed the submitted analysis report
INFO: More about the report processing at http://localhost:9000/api/ce/task?id=AXZwXu71UZLk_accn7bg
INFO: Analysis total time: 26.953 s
INFO: ------------------------------------------------------------------------
INFO: EXECUTION SUCCESS
INFO: ------------------------------------------------------------------------
INFO: Total time: 28.692s
INFO: Final Memory: 13M/54M
INFO: ------------------------------------------------------------------------
```
这次的结果如下:

![1](29.png)

也是 passed， 而且可以看到扫描结果还不错，除了一个 unreachable 的代码问题，其他几乎没啥问题，这个也是因为本身 golang 就要求错误啥的都要自己在写的时候就捕获，所以相对应的问题也比较少。

### 6. 扫描 JavaScript 项目
接下来针对 JavaScript， 也搞一个项目检测一下， 而且是 vue 框架项目。 一样的流程，扫描一下。

因为代码量比较大，所以会比较久，而且这一次的检测结果竟然是失败的， 为了担心有问题，我还特意跑了两次， 结果两次都是 failed， 因为有两个 E 级别的东西， 一个是 bug ，一个是安全隐患

![1](30.png)

然后我点了 bug 列表看了一下，有些确实有点问题，比如 重复定义， 还有 unreachable code，但是还有一些其实语法是没问题的，只是它没办法识别，比如这个标签

![1](31.png)

![1](32.png)

他就识别不了， 其实 v-deep 的语法是 vue 中 style scope 深度访问新方式， 不过 SonarQube 没有识别出来。

另外关于安全缺陷的扫码，识别也有点问题， 

![1](33.png)

这个明明是一个 password 字段的定义， 但是它却识别为密码硬编码。 只能说识别不够智能。 当然也有可能是 js 的语法变化太快了，它的语法分析库跟不上。

## 数据库配置的问题
之前在实操的时候，也有看到网上相关的一些博文，大部分都是有讲到要配置数据库的，但是我一整套流程操作下来都没有涉及到数据库的配置问题，甚至连 SonarQube 的配置文件 `sonar.properties` 都没有改过配置。 不过我后面抱着这个疑惑还是去翻了一下这个配置文件，发现这一行注释就解开了我的疑惑：
```text
# DATABASE
#
# IMPORTANT:
# - The embedded H2 database is used by default. It is recommended for tests but not for
#   production use. Supported databases are Oracle, PostgreSQL and Microsoft SQLServer.
# - Changes to database connection URL (sonar.jdbc.url) can affect SonarSource licensed products.
```
他有一个内置的 H2 的数据库是默认的，我们直接用默认的即可，不需要再额外设置数据库配置。

不过他的界面上，还有一个 warning 提示是针对这个内置数据库的

![1](35.png)

那就是用了这个内置的数据库，只能用来做常用分析，不支持扩展，同时也不支持新版本的升级和迁移。 所以如果后面要放到构建流程去，那么有可能得自己再配一个数据库，比如 mysql 之类的。

## 总结
这次通过 SonarQube 总共分析了三个不同语种的项目， 其中两个通过，一个失败。

![1](34.png)

其中 php 和 golang 的语法分析规则还是挺可靠的。 JavaScript 的话，如果是原生的可能会好一点， 但是现在大家都不用原生 JavaScript 写代码了， 大部分都用框架， 而且现在 JavaScript 框架层出不穷，迭代贼快， 新语法也很多， 所以语法分析规则跟不上也很正常。 所以 JavaScript 的代码质量检测会稍微差一点， 但是一些通用的写法问题，比如变量定义，重复定义，unreachable code 这种检测肯定是没问题的。

总的来说 SonarQube 用来检测项目代码质量还是有意义的。接下来就可以考虑在 gitlab + Jenkins + SonarQube 的模式，在构建的时候，先进行代码质量检测的机制。

---
参考资料
- [个人学习系列 - SonarQube的简单使用](https://segmentfault.com/a/1190000020719584?utm_source=tag-newest)
- [项目工程代码质量检测神器——SonarQube 的用法](https://www.imooc.com/article/279446?block_id=tuijian_wz)



