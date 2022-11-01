---
title: bug 追踪系统 Sentry (1) -- 单机安装
date: 2022-10-28 18:14:16
tags: Sentry
categories: 实用工具集
---
## 前言
现在的前端项目，如果一旦发布上线，代码一般都会进行混淆、压缩甚至加密，如果线上没有bug跟踪系统，客户端一旦报错，前端就无法及时感知，这个时候就需要使用人员上报，一层层上报到技术这边，技术如果要调试或者获得更具体的信息，就没有办法了，大部分情况下只有一张图片，但光靠一张图片，要追查bug产生的原因，有的时候是蛮困难的，等真正查清楚了，黄花菜都凉了。

所以其实是需要一个可以记录 bug 或者抛送异常数据的一个系统的。 其实这一块业内是有一些解决方案，除了一些系统之外， 各大云服务厂商其实也有类似的产品，比如
1. [frontjs](https://www.frontjs.com/pricing)
2. [fundebug](https://www.fundebug.com/)
3. [trackjs](https://trackjs.com/)
4. [instabug](https://instabug.com/)
5. [rollbar](https://rollbar.com/)
6. [sentry](https://sentry.io)

但是规模大了，基本上都要商用，都需要钱， 再加上之前也有用过美团开源的 Logan: {% post_link use-web-logan %}, 基本上体验了一阵子下来，也有一些问题没有解决，比如:
1. 没有多项目管理
2. 搜索功能鸡肋
3. 日子列表缺少筛选功能，希望能加上分类筛选，例如用户信息、设备型号、浏览器环境、或者添加自定义字段进行筛选。
4. 没有分页，无法查看20条以后的数据
5. 日志丢失内容
6. 希望可以自定义日志类型与颜色
7. 在日志详情页面上看不到环境等信息
8. 日志条目无法显示颜色，例如时间用其他颜色显示
9. 日志条目详情希望可以解析json

<!--more-->
就不吐槽了，反正后面也是沦为鸡肋了。只能再次寻找，后面发现 sentry 这个系统虽然也有商用版本， 但是核心的功能，其实都有开源的， 可以自己搭建， 再加上身边的业务人士也有用过，貌似评价还行， 所以就试用看看。

## 单机搭建
关于自己搭建的话，官网是有推荐教程的，就是用 docker-composer 来搭建: 
- [Self-Hosted Sentry nightly](https://github.com/getsentry/self-hosted)
- [Self-Hosted Sentry](https://develop.sentry.dev/self-hosted/)

其实这个系统就是一堆的微服务搭起来的，架构的话，大概是这样子:

![](1.png)

这边不细讲架构，因为我自己也没用熟，反正只需要知道这玩意用 docker-compose 一整套部署之后，光在跑的容器就有 30+ 个。 废话少说，接下来马上进入动手环境

### 1. 硬件配置要求
别看是单机部署，但是对环境是有要求的:
- Docker 19.03.6+
- Compose 1.28.0+
- 4 CPU Cores
- 8 GB RAM
- 20 GB Free Disk Space

所以我搞了一台 4核8G 的 centos 7 的服务器裸机准备来用。

### 2. 前置环境服务
因为是用 docker-compose 来部署的，所以先装个 19 版本的 docker， 具体指令如下:
```text
[root@VM-64-9-centos ~]# sudo yum install -y yum-utils
[root@VM-64-9-centos ~]# sudo yum-config-manager \
> --add-repo \
> https://download.docker.com/linux/centos/docker-ce.repo
[root@VM-64-9-centos ~]# sudo yum install docker-ce docker-ce-cli containerd.io
[root@VM-64-9-centos ~]# sudo systemctl start docker
[root@VM-64-9-centos ~]# docker -v
Docker version 20.10.20, build 9fdeb9c
```

接下来安装 1.28.0 版本以上的 docker-compose， 具体指令如下:
```text
[root@VM-64-9-centos ~]# curl -L "https://github.com/docker/compose/releases/download/1.29.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
[root@VM-64-9-centos ~]# chmod +x /usr/local/bin/docker-compose
[root@VM-64-9-centos ~]# ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
[root@VM-64-9-centos ~]# docker-compose --version
docker-compose version 1.29.0, build 07737305
```

### 3. 安装程序
接下来就要安装程序了， 从 github 将项目下载下来: [Self-Hosted Sentry nightly](https://github.com/getsentry/self-hosted), 然后进入目录，执行 `./install.sh` 文件
```text
[root@VM-64-9-centos self-hosted-master]# ./install.sh --no-report-self-hosted-issues --skip-user-prompt
```
我这边用了两个指令:
- `--no-report-self-hosted-issues` 不往 sentry 官方回报任何的 issue 咨询
- `--skip-user-prompt` 安装过程中，需要用户交互的场景，先跳过 (比如创建管理员这一步就会在安装过程中被跳过)

然后接下来就开始跑了，这个过程还是比较久的，因为下载的镜像比较多，如果是在国内的话，可能要科学上网

安装的 log 中有几个细节提一下:
1. 安装过程中会以当前 docker-compose 所在目录为相对目录来生成一些配置文件，比如这些 log:

```text
▶ Ensuring files from examples ...
Creating ../sentry/sentry.conf.py...
Creating ../sentry/config.yml...
Creating ../symbolicator/config.yml...
```
```text
▶ Ensuring Relay credentials ...
Creating ../relay/config.yml...
Relay credentials written to ../relay/credentials.json.
```
```text
▶ Generating secret key ...
Secret key written to ../sentry/config.yml
```
2. 会有创建管理员的交互，不过被我们跳过去, 但是后面有指令可以操作这一步

```text
Did not prompt for user creation. Run the following command to create one
yourself (recommended):

  docker-compose run --rm web createuser
```

### 4. 启动容器
```text
[root@VM-64-9-centos self-hosted-master]#  docker-compose up -d
```
看一下启动的情况，确实有 31 个容器在跑

![](3.png)

不过可以看到有一个 容器有挂掉:
```text
sentry-self-hosted_geoipupdate_1                                /usr/bin/geoipupdate -d /s ...   Exit 1
```

然后查了一下 安装的 log，确实有报了这个 warning:
```text
08:23:53 [WARNING] sentry.utils.geo: Error opening GeoIP database: /geoip/GeoLite2-City.mmdb
08:23:53 [WARNING] sentry.utils.geo: Error opening GeoIP database in Rust: /geoip/GeoLite2-City.mmdb
```
然后最后安装之后的报错信息也有:
```text
▶ Setting up GeoIP integration ...
Setting up IP address geolocation ...
Installing (empty) IP address geolocation database ... done.
IP address geolocation is not configured for updates.
See https://develop.sentry.dev/self-hosted/geolocation/ for instructions.
Error setting up IP address geolocation.
```

看样子是找不到这个 ip地址库 (`/geoip/GeoLite2-City.mmdb`), 看了一下教程，是需要在[这边](https://www.maxmind.com/en/geolite2/signup), 注册一个免费的账号的，然后放在 `geoip/GeoIP.conf`

看了一下，对应的 geoip， 目前是只有两个 empty 的空文件:
```text
[root@VM-64-9-centos geoip]# ll
total 8
-rw-r--r-- 1 root root 1055 Oct 25 19:17 GeoLite2-City.mmdb
-rw-r--r-- 1 root root 1055 Oct 22 07:00 GeoLite2-City.mmdb.empty
```

所以可以申请这个账号，然后新建一个这个 `geoip/GeoIP.conf`, 然后填入对应的账号信息:
```text
AccountID 012345
LicenseKey foobarbazbuz
EditionIDs GeoLite2-City
```
然后接下来重启一下整个 docker-compose 实例:
```text
docker-compose restart
```
让容器 `relay` 和 `web` 可以读到这个配置，然后下载对应的 `GeoLite2-City.mmdb` 文件

当然还有一种更简单的方式，就是直接替换 docker-compose 所在目录下的 `/geoip/GeoLite2-City.mmdb` 文件就行了，替换成从 maxmind 下载下来的文件:
```text
[root@VM-64-9-centos geoip]# ll
total 68032
-rw-r--r-- 1 root root 69657215 Oct 11 08:00 GeoLite2-City.mmdb
-rw-r--r-- 1 root root     1055 Oct 22 07:00 GeoLite2-City.mmdb.empty
```
然后一样重启 docker-compose，这样子就会生效了。 
> 需要注意的是，在启动后很快看到 `sentry_self_hosted_geoipupdate_1` 容器退出是正常的，因为更新地理定位数据库是一次性的批处理过程，而不是长时间运行的工作。 所以运行完就直接退出了

查看的话，也是很简单，登录后台，然后在 `Settings > Account > Security > Session History` 可以看 session 的历史记录， 如果有包含国家的话，就说明 地址库有生效了。 如果只看到 ip 地址，那么就是没有生效


### 5. 进入管理后台
因为管理后台的默认起的端口号是 9000， 所以可以访问进入到管理后台

![](2.png)

这样子最简单的安装就成功了。 接下来我们初始化用户并且配置邮件服务器.

---
相关 Sentry 系列文章:
- {% post_link sentry-1 %}
- {% post_link sentry-2 %}
- {% post_link sentry-3 %}
- {% post_link sentry-4 %}

---
参考资料:
- [Self-Hosted Sentry nightly](https://github.com/getsentry/self-hosted)
- [Self-Hosted Sentry](https://develop.sentry.dev/self-hosted/)
- [Self-Hosted Geolocation](https://develop.sentry.dev/self-hosted/geolocation/)
- [你知道Sentry 开源版与商业 SaaS 版的区别吗？](https://blog.csdn.net/o__cc/article/details/122445341)
- [前端异常监控平台之Sentry落地](https://blog.poetries.top/2022/07/27/sentry-summary/)
- [线上bug追踪之Sentry初步尝试（一）](https://zhuanlan.zhihu.com/p/75385179)
- [线上bug追踪之Sentry release+sourceMap（二）](https://zhuanlan.zhihu.com/p/75534743)
- [线上bug追踪之Sentry 定制错误信息（三）](https://zhuanlan.zhihu.com/p/75534976)
- [抓 Bug 神器的工作原理——聊聊 Sentry 的架构](https://baijiahao.baidu.com/s?id=1691152324720003928)