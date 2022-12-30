---
title: prometheus + grafana 实战篇(1) - 定义标签组和规范
date: 2022-12-30 15:49:12
tags: 
- prometheus
- grafana
categories: prometheus 相关
---
## 前言
经过前面的学习，我们已经能够很好的掌握 prometheus 并建立监控后台了，但是那个毕竟是学习篇，用于循循渐进的学习和掌握。
- [基于 prometheus 打造监控报警后台](https://kebingzao.com/categories/prometheus-%E7%9B%B8%E5%85%B3/)

但是真正涉及到线上的实战，其实复杂度还是跟学习的情况差很多的， 因此我这边结合在实际项目上的一些实战，也捋了个实战篇。

## prometheus 推荐的最佳实践
在仪表板(dashboard)上显示尽可能多的数据虽然很爽，尤其是当像 Prometheus 这样的系统提供了对应用程序进行如此丰富的检测的能力时。这可能会导致控制台由于包含太多信息而变得难以理解，即使是系统专家也难以从中获得意义。

所以千万不要将所有的数据都往 dashboard 上面搬，请考虑你建这个 dashboard 的初衷是什么，想要解决什么问题，如果你想要解决的问题是一个很复杂的问题的话，请把他进行拆解，并且分成多个 dashboard 来分批解决。

prometheus 官方推荐的 dashboard 指南:
1. dashboard 上面的图表最好不要超过 5 个
2. 每张图的图(线) 也不超过 5
3. 如果使用的是 dashboard 的第三方模板，请避免右侧表格中的条目超过 20-30 个

<!--more-->
上面的三个指南，不一定要严格遵守，但是核心目的只有一个，就是你建的图表一定要可以解决你的某一个问题，或者暴露某一个问题。 而且遇到警报通知的时候，也可以通过这些图表快速检索和定位到问题。

至于第三点，就是使用第三方 dashboard 模板的时候，要怎么瘦身，因为大部分的第三方的 dashboard 模板，里面的图表其实不少，少一点的可能也有 10+ 个，多点的 30+ 以上，比如 node_exporter。

我一般的做法就是保留完整版，然后根据自己平时经常看的，再建一个常用版，常用版就保留自己经常看的几个图表，一般不会超过 10 个， 之所以保留完整版， 是因为完整版保留的信息比较多，如果遇到问题的时候， 常用版看不出问题，这时候就会转而用完整版来定位问题。

![](1.png)

## 定义标签规范
因为是线上实战，所以涉及的主机实例和对应的服务，一定是非常多的， 至少 100+，所以就需要对这些 instance 实例进行统一的标签定义规范

因为我们的业务比较复杂，有个人版线产品，有企业版产品，虽然都有用云主机， 但是并没有只用一家，可能会用 aws， 腾讯云， 阿里云 等云服务厂商的主机实例可能都会使用到。
### 1. 通用标签
所以在我们的项目中，我定了 5 个通用标签(这 5 个标签的范围是层层缩进的)，这 5 个标签是添加 instance 的时候，必须要设定的

|标签|值|描述|
|---|---|---|
| `env` | `prod` / `test` | 表示环境，prod 是生产环境，test 是测试环境
| `product_line` | `biz` / `personal` | 表示产品线， toC 产品或者 toB 产品
| `region` | `cn` / `jp` / `us-east` / `us-west` / `eu-west` / `sa-east` 等等 | 实例所在主机的云服务区域<br>这个按照自己项目的使用情况来定义
| `cloud_platform` | `aws` / `tx` / `aliyun`  | 实例所属主机的云服务厂商，比如 aws， 腾讯云，阿里云
| `server_type` | `mysql` / `redis` / `push` / `mqtt`  | 当前实例的服务类别，有第三方的，比如 nginx，mysql，也有自研的，比如 push， mqtt 这种

有了这 5 个通用标签，我们就可以很快的定位到某一个环境，某一个产品线，某一个区域下，某一个云服务厂商下的某一个服务，比如 我要添加线上环境，个人线产品在美西地区的 aws 主机的一台 mysql 的实例，那么就是:
```text
- targets:
  - 172.xx.xx.5:9104
  labels:
    env: prod
    product_line: personal
    region: us-west
    cloud_platform: aws
    server_type: mysql
```

### 2. 自定义标签
虽然上面的 5 个通用标签可以绝大部分定位到某一台服务，但是有时候我们也是需要对某一些服务有一些自定义标签的。

最常用的自定义标签就是 `wan`, 也就是外网 ip 地址， 因为在请求 metrics 接口的时候，一般都是走内网地址，在看内网地址的时候，我们一般都是很难辨别是哪一台， 所以针对这种 target 是内网地址的，我们一般都会再添加一个 `wan` 的标签，来指定所对应的外网地址。

或者当我有一个区域有存在多台负载均衡的时候，比如上述那个例子如果存在 3 台相关服务的负载均衡的时候，也得通过 wan 标签来区分是哪一台

关于自定义标签的定义，一般不限制，一般都是根据各自具体的业务来添加，比如 mysql 有主从机制，那么就需要一个自定义标签来表示这一台是 master 还是 salve。


## 定义 job 命名规范
因为涉及到的服务实例和主机比较多，所以我们也需要对 job 的命名做规范，因为 job 本身也是一个 label 标签，我的定义规则是: **根据该服务是否有自己的 metrics 接口来区分**:

也就是我们要采集的 instance 实例其实是有两种的:
### 1. 该服务自己有能力暴露 metrics 接口供 prometheus 抓取
一种是服务本身是有自己暴露出 metrics 接口的，可供 prometheus 采集， 这种情况也分两种:
#### 1.1 自研的程序自己加入 metrics 接口
prometheus 有提供很多主流语言的 client sdk，比如 golang，python， php 等等，因此我们完全可以在自己自研的程序自定义一些采集指标，并且供 prometheus 采集。

比如我们自己自研的 golang 程序都会接入 metrics 接口，用来采集一些业务的数据。
 
#### 1.2 服务本身没有暴露 metrics 接口，但是有对应的第三方的 exporter 可以帮忙采集，并且供 prometheus 抓取
这一种就是用的第三方服务有对应的 exporter 可以采集数据，比如 mysql, redis, nginx 等服务都有对应的第三方的 exporter 可以直接用。

而且 prometheus 也有提供各种第三方的导出器: [Exporters and integrations | Prometheus](https://prometheus.io/docs/instrumenting/exporters/)

对于以上这两种情况，我们认为该服务自己有能力采集自身的业务数据，所以他的 job 的命名就为: `server-xxx` (xxx 就是该服务的名称), 比如如果是我们自研的推送服务，那么就是 `server-push`, 如果是 mysql 那么就是 `server-mysql`, 如果是 nginx 那么就是 `server-nginx`

### 2. 服务本身没有自己的 metrics 接口，只能走 node_exporter
还有一种就是这个服务没有自己的 metrics (来不及自研，或者自研成本高，或者用的第三方软件就没有对应开源的 exporter，因为一些比较小众的可能就没有)，这时候要监控的话，只能监控该服务所在的主机，也就是统一安装 node_exporter。

这种就是要以 `node-xxx` 来命名，比如有一个业务服务是 id，并且没有自己的 metrics 接口，那么 job name 就是 `node-id`, 或者是一个第三方的服务，比如 webrtc 的 coturn 服务，也没有 metrics 接口， 那么他的 job name 就是 `node-turn` 。

这样子通过定义 job 的命名规范，我们就可以清楚的跟进我们所要抓取的服务情况，来定义对应的命名，而不至于混乱。

当然上面可能会有重复采集的情况，比如如果有服务在同一台主机实例上，比如 id 和 turn 服务，并且都没有自己的 metrics 接口，那么 node-id 和 node-turn 都会去采集这一台主机，就会造成重复采集，因为 job 不一样。 关于这种情况，我觉得还好，如果只是单纯有一台有交集的话，可以不管，如果全部都交集的话，可以共用一个 job，比如 `node-id-turn`， 这样子就是 job 下面的 target 实例都应该包含这两个服务才对。

## 使用文件服务发现来分离 prometheus 配置文件
我们都知道 prometheus 的所有的采集配置，都是配置在 prometheus.yml 这个配置文件的，那如果我有成百上千个 instance 服务要采集，那全部写在这个配置文件，这个配置文件就会变得很难维护，生怕改错。

之前我们有学过可以通过文件服务发现来动态添加实例: 
- {% post_link prometheus-10-sd %}

因此我们也可以通过这种方式来根据不同的 job 来进行配置文件的分割，比如这样子:
```text
scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "server-dataforward"
    basic_auth:
      username: zache
      password: xxxxxxxxxx
    file_sd_configs:
    - files:
      - file_sd/server-dataforward.yml

  - job_name: "node-id"
    file_sd_configs:
    - files:
      - file_sd/node-id.yml

  - job_name: "server-nginx"
    file_sd_configs:
    - files:
      - file_sd/server-nginx.yml

  - job_name: "server-mysql"
    file_sd_configs:
    - files:
      - file_sd/server-mysql.yml

```
这样子我们有几个 job，就有几个服务发现文件了，里面所涉及到的服务实例都是在这里面， 服务发现的文件名称也最好跟 job 名称保持一致
```text
[root@VM-64-9-centos file_sd]# cat node-id.yml 
# id 服务相关监控
# 以下是个人线的 id
- targets:
  - 172.xx.xx.5:9100
  labels:
    env: prod
    product_line: personal
    region: us-west
    cloud_platform: aws
    server_type: id
    wan: 43.xx.xx.176

# 更多实例...
```

## 总结
在开始实战之前，涉及到这么多的实例和主机，一定要先定规范，后面才不会乱。

---

参考资料:
- [Consoles and dashboards | Prometheus](https://prometheus.io/docs/practices/consoles/)


