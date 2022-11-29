---
title: 基于 prometheus 打造监控报警后台 (3) - 使用 grafana 创建仪表盘
date: 2022-11-29 16:57:42
tags: 
- prometheus
- grafana
categories: prometheus 相关
---
## 前言
通过 {% post_link prometheus-2-metrics %} 我们可以知道，虽然 prometheus 采集数据很棒， 但是数据渲染这一块，实在是做的不咋的， 而且官方也推荐使用 grafana 来进行数据渲染: [grafana support for prometheus](https://prometheus.io/docs/visualization/grafana/)

所以本节就安装一下 grafana 并创建 dashboard 仪表盘

## 安装
接下来我们直接按照官方文档进行安装: [Download Grafana](https://grafana.com/grafana/download), 一样采用二进制包的方式来安装

```text
# 下载安装
[root@VM-64-9-centos ~]# cd /usr/local/
[root@VM-64-9-centos local]# wget https://dl.grafana.com/enterprise/release/grafana-enterprise-9.2.5-1.x86_64.rpm
[root@VM-64-9-centos local]# sudo yum install grafana-enterprise-9.2.5-1.x86_64.rpm

# 加入到 system 并启动 (安装的时候，有创建 grafana-server.service 文件了，这边不再需要手动创建)
[root@VM-64-9-centos local]# systemctl enable grafana-server.service
[root@VM-64-9-centos local]# systemctl start grafana-server.service

# 查看端口监听， 3000 端口
[root@VM-64-9-centos local]# netstat -anlp  | grep 3000
tcp6       0      0 :::3000                 :::*                    LISTEN      8897/grafana-server
```

<!--more-->

这时候就可以访问管理后台页面了, 初始化用户名和密码都是 `admin`

![](1.png)

登录进去之后，他会让你重设一个密码, 然后就可以进入到后台主界面了

![](2.png)

## 添加 prometheus 数据源
grafana 支持多种的数据源， 其中就包括 prometheus, 因此接下来添加 prometheus 的数据源，具体步骤可以看这个: [Creating a Prometheus data source](https://prometheus.io/docs/visualization/grafana/#creating-a-prometheus-data-source)

```text
1. 单击边栏中的“齿轮”以打开“配置”菜单。
2. 单击“数据源”。
3. 单击“添加数据源”。
4. 选择“prometheus”作为类型。
5. 设置适当的 Prometheus 服务器 URL（例如，http://localhost:9090/）
6. 根据需要调整其他数据源设置（例如，选择正确的访问方法）。
7. 单击“保存并测试”以保存新数据源。
```

最后点击 `save & test`, 出现这个就可以了

![](3.png)

这时候在 数据源 就可以看到 prometheus 这个数据源了

![](4.png)

## 创建一个 dashboard
那接下来就可以创建一个 dashboard 了，叫做 `prometheus-self`， 表示这个 dashboard 就是用来展示 prometheus 自身 job (安装默认自带的那个 job) 的相关数据和图表

这时候我就可以新加一个面板，然后查询方式跟在 prometheus 后台的差不多，就是 `指标 + 标签组`，比如这个查询 prometheus 这个程序的 goroutines 的数量变化

![](5.png)

点击 `apply`，这样子一个面板就好了

![](6.png)

接下来就可以在这个 dashboard 上面添加各种各样的面板

## 通过导入 dashboard 模板来创建 node_exporter 的 dashboard
那么有没有更快的方式，比如直接根据现有的一些 exporter 来使用现成的一些 dashboard 模板 ，省的每次都要一个一个面板来添加，有点浪费时间。

其实是有的，以 `node_exporter` 这个 exporter 来说， 其中官网就有提供很丰富的各种 dashboard 模板可供导入:  [grafana dashboards](https://grafana.com/grafana/dashboards/)

比如找到了这个: [1860 node exporter full](https://grafana.com/grafana/dashboards/1860-node-exporter-full/)

那么要怎么导入这个 dashboard 模板? 有三种方式:
1. **Upload JSON file 方式** -> [官网](https://grafana.com/grafana/dashboards/) 下载 Dashboard 的json 格式文件然后使用这种方式导入
2. **通过 grafana.com 上面的 id 导入** -> 每一个模板的主页都有一个 importId， 输入那个 id，然后就可以加载到这个模板的 json 数据了
3. **将 json 字串粘贴到输入框中** -> 跟第一种类似，只是不用传文件的方式，只是直接将文件中的 json 字串粘贴过去输入框

在有外网的情况下， 最简单的肯定是直接输入 id，然后拉取就行了

![](7.png)

找到这个 id，然后复制粘贴进去，点击 load 按钮

![](8.png)

这时候 json 模板就加载下来了，接下来然后最后选择数据源，然后设置 dashboard 名称， 点击 import 按钮，这时候就导入数据并创建这个 dashboard 了。

![](12.png)

![](9.png)

可以看到里面的面板非常的丰富，并且左上角还可以选择不同的主机来查看。 真的非常的方便。 而且随便点进去某一个面板的 edit 按钮，进入到面板的编辑，就可以看到 promql 语法查询了

![](10.png)

可以在上面进行调整和修改。

## 通过导入 dashboard 模板来创建 prometheus 的 dashboard
同样的道理，我们一样如果想要 prometheus 的自身 job 的数据做 dashboard 的时候，一样可以去找是否有对应的模板，而不是像一开始那样一个面板一个面板的处理

比如就找到了这个: [普罗米修斯 2.0 概述](https://grafana.com/grafana/dashboards/3662-prometheus-2-0-overview/), 然后 id 是 3662

跟上面导入 node_exporter 的模板一样， 我们导入了这个模板，并且设置 dashboard 名称为 `prometheus 2.0`, 最后得到的 dashboard 效果如下

![](11.png)

这样子就得到了整个 prometheus server 的自身服务的一些监控数据了。

## 总结
通过接入 grafana， 我们可以很轻松的在上面创建基于 prometheus 数据源的各种各样的 dashboard，以达到我们的监控数据查看需求。
> 其实这个只是最简单的第一步，grafana 还有一些进阶的功能可以完善我们的监控需求，比如权限管控, reports 等等，后面用到的时候，会再讲到

但是数据是有了， 出现故障或者异常的时候， 怎么办? 所以接下来我们就要开始创建预警了。 

prometheus 和 grafana 都有各自体系的预警手段 (但是基本原理几乎一致)， 后面会讲到。

---

参考资料:
- [grafana support for prometheus](https://prometheus.io/docs/visualization/grafana/)
- [grafana 使用 dashboard](https://grafana.com/docs/grafana/latest/dashboards/use-dashboards/)








