---
title: 使用 Logan 来做前端日志系统
date: 2021-08-19 14:18:05
tags: logan
categories: 实用工具集
---
## 前言
前段时间，业务要上线一个 web 的业务，他是使用 webrtc 的技术去远程控制手机的。 但是使用 webrtc 连接过程中， 会涉及到大量的信息交互， 比如接口交互， signal 交互， ice 交互， sdp 交互。 有时候出现连接失败的情况， 需要开发人员去排查到底卡在哪一个环节。 但是 web 不像其他客户端，可以让用户提供某个目录下的某一个日志文件。 他在浏览器跑的， 一般关掉浏览器， log 也就没有了。 而且我们也不希望线上的产品，在浏览器的 console 控制台上输出一堆的 log， 显得很不专业。 而且也没有哪个用户会在运行的时候，打开 console 控制台的。

因为将这些连接失败的 log 上抛到服务端是必要的，虽然可以借助 indexDB 或者 localstorage 之类的 浏览器存储来暂时存放。 但是如果一整套都要自己做的话，真的太累了。 所以就萌生了直接找第三方开源解决方案的想法。 刚好有找到一个 美团点评开源的前端日志系统: Logan

## Logan
Logan 开源的是一整套日志体系，包括日志的收集存储，上报分析以及可视化展示。它提供了五个组件，包括端上日志收集存储 、iOS SDK、Android SDK、Web SDK，后端日志存储分析 Server，日志分析平台 LoganSite。并且提供了一个 Flutter 插件Flutter 插件。

相关资讯这边不再说，本文也不是科普文，而是实操文，具体资料可以看:
- [Logan](https://github.com/Meituan-Dianping/Logan/blob/master/README-zh.md)
- [Logan Web SDK](https://github.com/Meituan-Dianping/Logan/blob/master/Logan/WebSDK/README.ch.md)
- [美团开源 Logan Web：前端日志在 Web 端的实现](https://tech.meituan.com/2020/01/09/meituan-logan.html)

<!--more-->

废话少说，开始安装吧

## 安装
一整套要全部安装成功，包含三部分
1. Server 端
2. 展示日志的 UI 端
3. 前端调用的 sdk

### 部署 Server 端
接下来我们开始部署 Server 端，然后我们的服务器是 CentOS 7。

这个 Server 端虽然 github 项目上有源码，但是他其实是一个 java 程序，而且还是一个 maven 项目的 java 程序， 所以部署的话要装相关的 java 环境，包含:
- java
- tomcat (server 部署)
- maven (打成 war 包)

![](1.png)

#### 安装 java
因为我们的服务器上没有 java 环境，所以我们先安装 java
```text
[root@VM-0-13-centos ~]# java -version
-bash: java: command not found

[root@VM-0-13-centos ~]# yum install java
Loaded plugins: fastestmirror, langpacks
...

[root@VM-0-13-centos ~]# java -version
openjdk version "1.8.0_302"
OpenJDK Runtime Environment (build 1.8.0_302-b08)
OpenJDK 64-Bit Server VM (build 25.302-b08, mixed mode)
```
这样子就安装好了。

#### 安装 Tomcat
因为之前也没有安装过 Tomcat， 所以这次也要装，可以参照: [apache tomcat on centos](https://www.ionos.com/digitalguide/server/configuration/apache-tomcat-on-centos/)
```text
[root@VM-0-13-centos ~]# yum install tomcat
Loaded plugins: fastestmirror, langpacks
...
```
这样子就装好了。 接下来启动
```text
[root@VM-0-13-centos usr]# sudo systemctl start tomcat
```
然后测试一下 `8080` 端口是否有启动
```text
[root@VM-0-13-centos usr]# curl 127.0.0.1:8080
```
虽然有启动，但是没有 web 页面响应， 这个是因为 我们这时候装的 Tomcat 只有一个服务， 并没有包含 webapp 和 admin 管理那一块的。 所以这一块也要另外安装才行

所以先停止
```text
[root@VM-0-13-centos usr]# sudo systemctl stop tomcat
```
然后安装
```text
[root@VM-0-13-centos usr]# yum install tomcat-webapps tomcat-admin-webapps tomcat-docs-webapp tomcat-javadoc
```
这时候 重新启动，就可以看到页面有返回了
```text
[root@VM-0-13-centos usr]# sudo systemctl start tomcat
[root@VM-0-13-centos usr]# curl 127.0.0.1:8080

<!DOCTYPE html>


<html lang="en">
    <head>
        <title>Apache Tomcat/7.0.76</title>
        ...
```
然后在腾讯云安全组那边 对 `8080` 端口，进行开放，然后外网访问试试 

> 因为后面要用 nginx 进行转发代理，所以 Tomcat 的 8080 端口其实不需要在外网开放, 本文这么做主要为了更直观的看到 Tomcat 部署的页面情况

![](2.png)

这时候就可以看到了。

最后将 Tomcat 设置为 开机自启动

```text
[root@VM-0-13-centos usr]# sudo systemctl enable tomcat
Created symlink from /etc/systemd/system/multi-user.target.wants/tomcat.service to /usr/lib/systemd/system/tomcat.service.
```

#### 部署程序
接下来我们将这个 Logan 的 server 项目 部署到 Tomcat 中

找到 webapps 目录下
```text
[root@VM-0-13-centos tomcat]# cd /var/lib/tomcat/webapps/
[root@VM-0-13-centos webapps]# ll
total 24
drwxr-xr-x 14 root   root   4096 Aug 18 16:06 docs
drwxr-xr-x  8 tomcat tomcat 4096 Aug 18 16:06 examples
drwxr-xr-x  5 root   tomcat 4096 Aug 18 16:06 host-manager
drwxr-xr-x  5 root   tomcat 4096 Aug 18 16:06 manager
drwxr-xr-x  3 tomcat tomcat 4096 Aug 18 16:06 ROOT
drwxr-xr-x  5 tomcat tomcat 4096 Aug 18 16:06 sample
```

这个 webapps 目录下的 ROOT，其实就是站点根目录，也就是只要保留 ROOT 目录即可，其他都可以删掉 (事实上后面这个 webapps 会直接清空的)

接下来我们将 Server 的代码打成 war 包，就是
```text
https://github.com/Meituan-Dianping/Logan/tree/master/Logan/Server
```
这个目录下的代码 (刚开始不需要改任何的代码， 先运行起来再说)

![](3.png)

#### 踩过的坑，不能用 jar 打 war 包
这边在那时候的实作中，有走了一个弯路，就是知道是要打成 war 包， 但是那时候不知道要用 maven 打， 而是直接用 jar 打, 而且因为 jar 命令不存在，还要装一个依赖
```text
[root@VM-0-13-centos ROOT]# yum install -y java-1.8.0-openjdk-devel
```
然后这时候就可以用 jar 打 war 包了:
```text
[root@VM-0-13-centos ROOT]# jar -cvf ROOT.war ./*
added manifest
adding: pom.xml(in = 7781) (out= 1150)(deflated 85%)
adding: README(in = 4471) (out= 1401)(deflated 68%)
adding: Server.zip(in = 63860) (out= 47227)(deflated 26%)
adding: src/(in = 0) (out= 0)(stored 0%)
```
在代码目录下， 成功将 war 包打出来了， 并命名为 ROOT.war, 接下来就是将 Tomcat 的 webapps 目录清空掉，然后将 ROOT.war 放到 webapps 目录下，然后重启 Tomcat， 这时候 Tomcat 就会将 ROOT.war 解压成 ROOT 目录了。 当然结果是这样子是不行的， 直接 404 了

![](4.png)

#### 安装 maven，并且用 maven 打 war 包
后面发现这个是一个 maven 项目，所以打 war 包，还得用 maven 来打。 所以还是得装 maven:
```text
[root@VM-0-13-centos ~]# mvn -v
-bash: mvn: command not found

[root@VM-0-13-centos local]# yum install maven
Loaded plugins: fastestmirror, langpacks
...

[root@VM-0-13-centos local]# mvn -v
Apache Maven 3.0.5 (Red Hat 3.0.5-17)
Maven home: /usr/share/maven
Java version: 1.8.0_302, vendor: Red Hat, Inc.
Java home: /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.302.b08-0.el7_9.x86_64/jre
Default locale: en_US, platform encoding: UTF-8
OS name: "linux", version: "3.10.0-1160.31.1.el7.x86_64", arch: "amd64", family: "unix"
```

这样子就装好了， 接下来就是执行 `mvn package` 指令在代码目录下打包了 (第一次打的时候，会安装很多依赖，所以会稍微久一点):
```text
[root@VM-0-13-centos Server]# mvn package
[INFO] Scanning for projects...
[WARNING]
[WARNING] Some problems were encountered while building the effective model for com.meituan.logan:logan-web:war:1.0-SNAPSHOT
...
[INFO] Assembling webapp[logan-web] in [/root/Server/target/logan-web-beta-1.0-SNAPSHOT]
[INFO] Processing war project
[INFO] Webapp assembled in[76 msecs]
[INFO] Building war: /root/Server/target/logan-web-beta-1.0-SNAPSHOT.war
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 3.657s
[INFO] Finished at: Wed Aug 18 17:27:49 CST 2021
[INFO] Final Memory: 19M/417M
[INFO] ------------------------------------------------------------------------
```
最后就可以得到 `/root/Server/target/logan-web-beta-1.0-SNAPSHOT.war` 这个 war 包了。

接下来还是一样，将 webapps 清空，然后将这个 war 拷贝到这个目录下，并且重命名为 ROOT.war, 然后重启 Tomcat:
```text
[root@VM-0-13-centos webapps]# rm -rf ./*
[root@VM-0-13-centos webapps]# ll
total 0 

[root@VM-0-13-centos webapps]# cp /root/Server/target/logan-web-beta-1.0-SNAPSHOT.war ./ROOT.war
[root@VM-0-13-centos webapps]# ll
total 26028
-rw-r--r-- 1 root root 26651559 Aug 18 17:31 ROOT.war

[root@VM-0-13-centos webapps]# systemctl restart tomcat

[root@VM-0-13-centos webapps]# ll
total 26032
drwxr-xr-x 4 tomcat tomcat     4096 Aug 18 17:31 ROOT
-rw-r--r-- 1 root   root   26651559 Aug 18 17:31 ROOT.war

[root@VM-0-13-centos ROOT]# ll
total 12
-rw-r--r-- 1 tomcat tomcat   16 Aug 16 15:37 index.jsp
drwxr-xr-x 3 tomcat tomcat 4096 Aug 18 17:31 META-INF
drwxr-xr-x 4 tomcat tomcat 4096 Aug 18 17:31 WEB-INF
```
这时候就可以看到 `index.jsp` 页面了。访问一下，就可以看到生效了

![](5.png)

请求一个业务接口，发现也不会报错:

![](6.png)

#### 创建库表并且配置正确的数据库链接
Server 部署没问题之后，接下来就是建表了， 直接按照教程，先把这 4 个表建起来:
```text
https://github.com/Meituan-Dianping/Logan/tree/master/Logan/Server
```

![](7.png)

建完之后，要修改对应的连接串，这个就要改到这个 java 代码，在 `db.properties`:

![](8.png)

> 因为改到业务代码了， 如果要应用的话，那么就得重新打 war 包，然后重新部署 Tomcat，并重启 Tomcat

#### 设置 CORS 跨域
因为这个服务是要给前端请求的， 所以会涉及到 跨域问题， 他本身在 java 代码中，就有这个 跨域的配置了:
```text
Server\src\main\java\com\meituan\logan\web\filter\CORSFilter.java
```
原先的代码是这样子的:
```text
    private final List<String> allowedOrigins = Arrays.asList("http://localhost:3000", "http://localhost:5001");
```
因为后面我们前后端都会有域名，而且域名是不一样的，所以这边就要改成我们前端的域名，当做跨域白名单处理:
```text
    private final List<String> allowedOrigins = Arrays.asList("https://test-logan.foo.com", "https://logan.foo.com");
```

这样子就可以了，但是还会有一个问题，加上我后面其他 foo.com 的子域名也要用这个服务呢，那不是又得添加一次白名单， 所以更好的方法，直接去判断 origin 字段是否含有 `foo.com` 就可以了。

原先代码是:
```text
    response.setHeader("Access-Control-Allow-Origin", allowedOrigins.contains(origin) ? origin : "");
```
就是判断 origin 是否在白名单之类， 但是后面我们改成直接判断是否含有这个 `foo.com` 二级域名即可
```text
    response.setHeader("Access-Control-Allow-Origin", (origin== null||origin== "") ? "" : (origin.contains("foo.com") ? origin : ""));
```
这样子更好，只要是这个域名下的子域名，就都可以跨域。

> 因为改到业务代码了， 如果要应用的话，那么就得重新打 war 包，然后重新部署 Tomcat，并重启 Tomcat

#### nginx 代理转发
因为 Tomcat 默认是 8080 端口，不好看，最好还是默认的 80， 但是 80 在这台服务器上已经被 nginx 拿走了， 而且 Tomcat 服务如果要单独设置 域名 和 ssl 设置，又比较麻烦， 所以直接本机用 nginx 进行代理转发就行了。快捷方便， 所以转发代理的配置如下:
```text
upstream api_server {
		server 127.0.0.1:8080 fail_timeout=0;
}

server {
	listen 80;
	listen 443 ssl http2;
	server_name test-logan-server.foo.com;
	include ssl/ssl_foo.com.conf;
  access_log /var/log/nginx/logan.foo.com.access.log main;
	error_log /var/log/nginx/logan.foo.com.error.log warn;	
	 
	location / {
		proxy_pass  http://api_server;
    proxy_set_header Host $host;
    proxy_redirect off;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	}

}
```

![](9.png)

这样子 Server 就部署好了。

### 部署 LoganSite 端
这个是前端的 日志展示页面， 其实是一个用 react 框架写的静态站点， 部署文档:
```text
https://github.com/Meituan-Dianping/Logan/blob/master/README-zh.md#logansite
```
我们直接看部署到线上的方式，就是 将 `LoganSite/src/common/api.js` 中第四行改成 server 的地址即可:
```text
//const BASE_URL = process.defineEnv.API_BASE_URL;
const BASE_URL = "https://test-logan-server.foo.com";
```
这样子就可以直接打包了:
```text
$ cd $LOGAN_SITE
$ npm install
$ npm run build
```

![](10.png)

然后将 build 的静态文件，拷贝到服务器上，用 nginx 部署即可, 配上域名 和 ssl
```text
server {
	listen 80;
	listen 443 ssl http2;
	server_name test-logan.foo.com;
	
	include ssl/ssl_foo.com.conf;
  access_log /var/log/nginx/logan.foo.com.access.log main;
	error_log /var/log/nginx/logan.foo.com.error.log warn;	
	 
 	root /usr/local/src/Logan/Logan/LoganSite/build;
	index index.html index.htm index.php;

	location / {
		try_files $uri $uri/ /index.html;
	}

}
```
这样子就好了

![](11.png)


### 接入 WEB SDK 测试
Server 和 LoganSite 都部署好了，接下来就是接入 WEB SDK 看下是否有抛送， 这个也是直接有 demo 的，在:
```text
https://github.com/Meituan-Dianping/Logan/tree/master/Example/Logan-WebSDK
```
直接跑这个项目就行了，不过要改一下 `src/test.js` 中抛送的这个 url `reportUrl`， 改成我们 Server 的接口:
```html
var Logan = require('logan-web');
Logan.initConfig({

});
function timeFormat2Day(date) {
    var Y = date.getFullYear();
    var M = date.getMonth() + 1;
    var D = date.getDate();
    return Y + '-' + (M < 10 ? '0' + M : M) + '-' + (D < 10 ? '0' + D : D);
}
document.getElementById('log').onclick = log;
document.getElementById('logWithEncryption').onclick = logWithEncryption;
document.getElementById('report').onclick = report;

function log() {
    Logan.log('Hello World!132456', 1);
}

function logWithEncryption() {
    Logan.logWithEncryption('Hello World!', 2);
}

function report() {
    var now = new Date();
    var sevenDaysAgo = new Date(+now - 6 * 24 * 3600 * 1000);
    Logan.report({
        reportUrl: 'https://test-logan-server.foo.com/logan/web/upload.json',
        deviceId: 'test-logan-web-2',
        fromDayString: timeFormat2Day(sevenDaysAgo),
        toDayString: timeFormat2Day(now),
        webSource: 'browser',
        environment: navigator.userAgent,
        customInfo: JSON.stringify({ userId: 123456789, biz: 'Live Better' })
    });
}

```
其他都不用改，然后重新打包，安装依赖之后，执行:
```text
npm run demo
```
这时候就会用 webpack 打包了:
```text
  "scripts": {
    "demo": "rm -rf demo/js && webpack"
  },
```
不过如果是 windows 的话，那么是不支持 `rm` 指令的，可以直接去掉 rm 指令:
```text
  "scripts": {
    "demo": "webpack"
  },
```
这样子就可以了，不过为了防止打包的旧文件一直存在，打包之前最好将 demo 文件的 js 目录删除。 打完包之后， 我们还需要创建一个 webserver， 可以用 npm 装一个简单的 [http-server](https://www.npmjs.com/package/http-server), 然后直接在 demo 目录下启动即可:
```text
F:\foo_code\github\Logan\Example\Logan-WebSDK\demo>http-server ./ -p 8001
Starting up http-server, serving ./
Available on:
  http://192.168.40.51:8001
  http://127.0.0.1:8001
Hit CTRL-C to stop the server

```
这时候就可以访问 test.html 文件了，不过要注意，因为有跨域限制，所以要本地 host 一个 `*.foo.com` 的域名，才能正常请求:

![](12.png)

点击 Log 按钮，就是记录信息到 indexDB， 然后 点击 report 按钮，就是抛送到服务端

![](13.png)

我们可以在 indexDB 那边看到数据，这时候 LoganSite 也有数据了。

![](14.png)

## 总结
整个框架算是搭起来了， 接下来就看好不好用，以及怎么用了， 因为我看了一下，他有任务编号，也有日志编号。 所以用的话，也不能无脑用，还是有技巧的。




