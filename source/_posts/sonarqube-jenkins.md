---
title: 使用 sonarqube 在 gitlab + Jenkins 的构建流程之前添加代码质量检测
date: 2020-12-21 11:38:56
tags: 
- sonarqube
- Jenkins
categories: 实用工具集
---
## 前言
之前在 {% post_link sonarqube-use %} 我们已经实现在 windows 7 在使用 SonarQube 针对项目进行代码质量检测了。 接下来我们将结合到项目中原有的构建流程中，在 Jenkins 进行构建的时候，执行构建脚本之前，先通过 SonarQube 进行代码质量检测。 所以之前的流程是这样子的:
1. 用户提交分支或者打tag，触发 Jenkins 设置的 hook 机制
2. Jenkins 收到 hook， 拉取对应分支下来
3. 最后执行构建脚本 (比如如果是go项目就进行打包，然后将二进制分发到对应的服务器， 最后重启服务，如果是php项目就将修改后的代码文件，同步到线上的服务)
4. 如果有其他 hook，比如钉钉通知， 会在开始打包的时候，和打包结束的时候，分别推送一条到对应的群组中

![1](1.png) 

<!--more-->
加入了 SonarQube 的构建流程就会变成
1. 用户提交分支或者打tag，触发 Jenkins 设置的 hook 机制
2. Jenkins 收到 hook， 拉取对应分支下来
3. <font color=red><b>分支拉取下来之后，通过 sonar scanner 进行代码质量检测，然后生成报告到 sonar 后台</b></font>
4. 最后执行构建脚本 (比如如果是go项目就进行打包，然后将二进制分发到对应的服务器， 最后重启服务，如果是php项目就将修改后的代码文件，同步到线上的服务)
5. 如果有其他 hook，比如钉钉通知， 会在开始打包的时候，和打包结束的时候，分别推送一条到对应的群组中
6.  <font color=red><b>开发人员可以到 sonar 后台看检测报告</b></font>

多了第 3 点和第 6 点。

## 实操
还是一样，为了更真实的演练这个效果，我会在本地环境 window 7 重现这个过程。 重新本地装 Jenkins， 不过 gitlab 内部代码库还是直接用内网环境的。 SonarQube 就是沿用 {% post_link sonarqube-use %} 这一个，本地已经装了。

因为只是重点实现 Jenkins 构建的时候，通过 SonarQube 进行质量检测， 所以上面的流程有几个环节，我不会处理，因为这几个环节跟我本文要展示的重点没啥关系，而且也很简单，可以自己配置:
1. gitlab hook 触发构建的方式不会配，正常情况下我们都是直接通过 gitlab hook 的方式来触发 Jenkins 构建，一般不会到 Jenkins 后台手动点击构建按钮(除非是项目要上线到生产环境), 但是因为我 Jenkins 是部署在本地的， 地址是 localhost 的，线上的 gitlab 配置 hook，没办法到我本地来，所以后续测试的时候，一律采用手动在 Jenkins 后台进行主动构建的方式来处理
2. 执行脚本， 这个流程也忽略， 因为跟代码质量检测没啥关系， 而且不同的项目所对应的脚本也不一样。
3. 钉钉通知，这个当然也不需要

## 安装 Jenkins
到 Jenkins 的下载页面选择 [windows 版本](https://www.jenkins.io/download/thank-you-downloading-windows-installer-stable/), 最新版本是 `2.263.1`, 下载是文件 `Jenkins.msi`, 双击点击安装即可

![1](2.png) 

![1](3.png) 

去这个地方拷贝一下这个密码，然后贴进去

![1](4.png) 

这时候进入到新手入门， 直接安装他推荐的插件

![1](5.png) 

安装完了之后，就创建一个 admin 账号

![1](6.png) 

最后确认一下

![1](7.png) 

这样子就安装成功了。 首次进入的界面是这样子的

![1](8.png) 

## 安装和配置 SonarQube 插件
接下来我们安装一下 SonarQube 插件， 找到 插件管理， 找到 SonarQube 这个插件

![1](9.png) 

选择下载并重启

![1](10.png) 

这时候就会进入到 下载中心， 然后下载完之后， 勾选重启的勾选框即可

![1](11.png) 

重启完，可以在这边找到

![1](12.png) 

接下来我们需要配置一下 SonarQube， 在 SonarQube 后台中生成一个Token（PS：用token代替输入用户名和密码）， 到 security 页面。 选择 Administrator 用户，查看 token

![1](14.png) 

可以看到之前跑那三个项目的时候， 已经有生成 3 个 token， 我这边再生成一个专门用来做 Jenkins 的  `017ff87ea20c68b1c0f9ea1be799f56325356f62`， 后面会用到

![1](15.png) 

既然 token 生成了，那么接下来就在Jenkins中配置连接 SonarQube 服务器的地址，这里用到的token就是刚才在 SonarQube 中创建的那个token, Jenkins 系统设置这边

![1](16.png) 

找到 SonarQube servers 配置这边

![1](17.png)

然后填入信息。

![1](18.png)

这边要添加一个 token 凭证 ， 如果点击没有下拉选项。 就先保存，然后再点 add， 然后要创建一个 secret text 的凭证， 文档有写
{% blockquote Jenkins https://docs.sonarqube.org/latest/analysis/scan/sonarscanner-for-jenkins/ %}
Installation
1. Install the SonarScanner for Jenkins via the Jenkins Update Center.
2. Configure your SonarQube server(s):
  1. Log into Jenkins as an administrator and go to Manage Jenkins > Configure System.
  2. Scroll down to the SonarQube configuration section, click Add SonarQube, and add the values you're prompted for.
  3. The server authentication token should be created as a 'Secret Text' credential.
{% endblockquote %}
然后 secret 就是刚才那个 token

![1](19.png)

最后点击 add 之后，这时候就可以选了

![1](20.png)

最后点击 apply 和 save， 这样子就配置好了。

最后，还需要配置全局工具配置

![1](21.png)

找到 `SonarQube Scanner`, 填入刚才创建的那个 scanner 名词，并且勾选 `Install automatically`

![1](22.png)

最后点击 apply 和 save 按钮， 这样子整个 SonarQube 就配置好了。

## 创建一个项目测试
接下来我们创建一个任务，测试一下构建有没有触发代码质量检测

![1](23.png)

接下来开始进行配置， gitlab 内部项目， 我们用 http 的方式来拉， 并且要先创建一个证书授权，要输 gitlab 的用户名和密码，这样子才有权限拉取项目。

![1](24.png)

如果填入的时候，没有报错的话，那应该就没问题。 如果是选择 ssh 的方式的话，就不能用用户名和密码的方式，不然就会报错

![1](25.png)

分支的话，为了不跟代码上面的分支有冲突， 我自己基于 master 分支， 重新拉了一个 分支，叫 test ，这样子就不会影响到线上已有的构建了。

![1](26.png)

接下来继续处理 build environment， 选择构建之前使用 SonarQube 进行代码检测

![1](27.png)

这边有个 warning， 不要管没事。

接下来继续到构建步骤，选择 `Execute SonarQube Scanner`
 
![1](28.png)

然后进行设置

![1](29.png)

这一串内容是在参考资料中，拷的， 然后稍微调整一下
```text
sonar.projectKey=my_demo
sonar.projectName=my_demo
sonar.projectVersion=1.0

sonar.language=java
sonar.sourceEncoding=UTF-8

sonar.sources=$WORKSPACE
sonar.java.binaries=$WORKSPACE
```

先点击 apply， 然后再 save。 至此， 就配置完毕了。 

其实对于标准的流程的，这边的配置少了两步，一步就是配置 gitlab hook：

![1](30.png)

一个是设置构建执行脚本

![1](31.png)

不过缺少这两步，对我们来说不会影响我们的测试目的。 感兴趣的，可以参考后面的参考文档，自己测试，不难。

接下来我们直接点击 `Build Now`, 试试看能不能正常构建

![1](32.png)

> 注意，我们并没有装  gitlab 的 webhook 通知插件，所以这边只能手动拉取。 没办法通过 push test 分支 来自动触发

![1](33.png)

可以看到代码拉下来了，并且成功应用 sonar scanner 进行代码质量检测了， 不过最后好像报错了:
```text
ERROR: Invalid value of sonar.sources for my_demo

INFO: ------------------------------------------------------------------------
INFO: EXECUTION FAILURE
INFO: ------------------------------------------------------------------------
INFO: Total time: 25.289s
INFO: Final Memory: 5M/27M
INFO: ------------------------------------------------------------------------
ERROR: Error during SonarScanner execution
ERROR: The folder 'C:Windowssystem32configsystemprofileAppDataLocalJenkins.jenkinsworkspacemy_demo' does not exist for 'my_demo' (base directory = C:\Windows\system32\config\systemprofile\AppData\Local\Jenkins\.jenkins\workspace\my_demo)
ERROR:
ERROR: Re-run SonarScanner using the -X switch to enable full debug logging.
WARN: Unable to locate 'report-task.txt' in the workspace. Did the SonarScanner succeed?
WARN: Unable to locate 'report-task.txt' in the workspace. Did the SonarScanner succeed?
ERROR: SonarQube scanner exited with non-zero code: 1
Finished: FAILURE
```
看起来好像是找不到项目目录， 可是我去查了一下，上面的目录是存在的， 后面检查了一下，上面的步骤有没有跟目录有关系的配置，后面发现可能是这个抄别人的配置可能有问题:
```text
sonar.sources=$WORKSPACE
sonar.java.binaries=$WORKSPACE
```
所以就把这两个配置去掉，只保留上面5个
```text
sonar.projectKey=my_demo
sonar.projectName=my_demo
sonar.projectVersion=1.0

sonar.language=java
sonar.sourceEncoding=UTF-8
```
结果这样子就可以了

![1](34.png)

接下来回到 SonarQube，终于有看到分析数据了。

![1](35.png)

接下来我在 test 分支稍微修改格式再push 一次， 然后再 build now 一次

![1](36.png)

可以看到有在继续分析了。 也是成功的，可以知道他是基于上一次的分析，只对修改的代码进行检查的，因为这一次我没有修改代码，所以分析结果就全部都是 0 了。

## 总结
通过本地搭建的 Jenkins 和 SonarQube， 再加上线上的 gitlab 内部库，我们可以实现在代码构建的时候，预先通过 SonarQube 进行代码质量检测。

---
参考资料:
- [Jenkins 集成 SonarQube Scanner](https://www.cnblogs.com/cjsblog/archive/2019/04/20/10740840.html)
- [SonarSource/sonar-scanning-examples](https://github.com/SonarSource/sonar-scanning-examples/blob/master/sonarqube-scanner/sonar-project.properties)
- [sonarscanner-for-jenkins](https://docs.sonarqube.org/latest/analysis/scan/sonarscanner-for-jenkins/)
- [Gitlab利用Webhook实现Push代码后的jenkins自动构建](https://www.cnblogs.com/kevingrace/p/6479813.html)




