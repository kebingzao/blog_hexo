---
title: 基于 prometheus 打造监控报警后台 (4) - 使用 alertmanager 发送警报
date: 2022-11-29 18:11:11
tags: 
- prometheus
- alertmanager
categories: prometheus 相关
---
## 前言
通过 {% post_link prometheus-3-grafana %} 我们已经能够在 grafana 上面创建漂亮的数据监控仪表盘了。

本节我们讲一下怎么在 prometheus 上设置预警并且发送警报。 

简单的架构图如下

![](1.png)

在 prometheus 监控系统中，采集与警报是分离的，所以是分为两个步骤的:
1. 在 prometheus 中创建警报规则，并监控警报和触发警报
2. 触发警报之后(firing)，prometheus 将报警信息转发给独立组件 alertmanager，然后经过 alertmanager 对报警信息处理之后，最后通过接收器发送给指定用户

同时 alertmanager 支持多种接收器(`receiver`), 比如 `Email`, `Slack` , `钉钉`, `企业微信` , `Webhook`

## 再添加一台 node 节点监控
为了便于后面更好的测试警报， 我这边又在另外的云服务器上添加另一个 node_exporter 监控，对应配置改成:
```text
- job_name: "vmware-host"
    static_configs:
      - targets:
        - localhost:9100
        - 43.153.11.234:9100
```

这时候 job `vmware-host` 就有两台实例主机监控了

![](2.png)

## 创建警报规则
prometheus 可以创建两种规则:
1. [记录规则(`recording rules`)](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/)
2. [警报规则](`alerting rules`)](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)

其中 `记录规则` 不在本次警报章节中，后面会单独开篇章讲述。 本章主要讲的是如何用 `警报规则` 创建一条警报

默认情况下，可以在 prometheus 的后台看到，其实并不会有 警报规则，以及对应的 alert 警报

![](3.png)

![](4.png)

### 1. 创建一条警报文件
首先就要创建一个警报规则:
```text
# 先在 prometheus 目录下创建一个 rules 目录，专门用来存放 rules 的 yml 文件
[root@VM-64-9-centos prometheus]# mkdir rules
[root@VM-64-9-centos prometheus]# cd rules

# 接下来创建一个 test-1.yml 的 rule 文件， 使用 YAML 文件格式
# 里面包含一条警报规则，就是监控 node 的节点存活
[root@VM-64-9-centos rules]# cat test-1.yml 
# rules/test-1.yml
groups:
  - name: test-1
    rules:
      - alert: 节点存活
        expr: up{job="vmware-host"} == 0
        for: 1m
        labels:
          level: critical
        annotations:
          summary: "机器 {{ $labels.instance }} 挂了"
          description: "{{$labels.instance}} 宕机(当前值: {{ $value }})

# 接下来使用工具检查 test-1.yml 的格式是否正常，一样用 promtool 检查
[root@VM-64-9-centos prometheus]# ./promtool check rules rules/test-1.yml 
Checking rules/test-1.yml
  SUCCESS: 1 rules found
```

这样子就创建了一个警报规则文件叫做 `test-1.yml`, 里面包含了一条警报规则，名称叫做 `节点存活`, 并且有使用 promtool 工具有校验 rule 文件格式是正确的(如果格式不正确，将无法应用)。

> 关于检查格式，如果 rules 目录下有多个规则文件，那么可以这样子一次性检查 `./promtool check rules rules/*.yml` 所有文件

> 一个规则文件里面可以包含多个警报规则，可以写在同一组 group 里面的 rules 下面，也可以单独写在另一个 group 中。

### 2. 将警报规则添加到 prometheus 配置文件中
在 prometheus 文件中，添加 `rule_files` 这个节点，其他不变
```text
# 编辑 prometheus.yml 添加 rule_files 节点，指定对应位置的警报规则文件
[root@VM-64-9-centos prometheus]# cat prometheus.yml 
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "rules/test-1.yml"

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "vmware-host"
    static_configs:
      - targets:
        - localhost:9100
        - 43.153.11.234:9100

# 验证配置文件是否格式正确 (连同 rules 文件一起验证)
[root@VM-64-9-centos prometheus]# ./promtool check config prometheus.yml 
Checking prometheus.yml
  SUCCESS: 1 rule files found
 SUCCESS: prometheus.yml is valid prometheus config file syntax

Checking rules/test-1.yml
  SUCCESS: 1 rules found

# 重新加载配置文件，让 rules 文件生效
[root@VM-64-9-centos prometheus]# curl -X POST localhost:9090/-/reload
```

这边如果有多个 rules 文件要应用的话，可以这样子写
```text
rule_files:
  - "rules/*.yml"
```
或者这样子写:
```text
rule_files:
  - "rules/test-1.yml"
  - "rules/test-2.yml"
```

接下来就可以看到在 prometheus 的后台上， rules 和 alerts 都有值了

![](5.png)

可以看到出现了一个 group 为 `test-1` 的警报规则组， 里面有一条警报，名称为 `节点存活`

alerts 那边也可以看到确实存在一个警告了， 但是出于 未激活的 `inactive` 状态， 这个说明当前状态是正常的

![](6.png)

### 3. 警报规则参数
关于 rules 文件的参数配置，可以参照官方的文档: [rules configuration](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/)
> 记录规则 和 警报规则 的配置差别仅仅在于 `rules` 小节下的第一个配置的 key 是 record 还是 alert。 其他都一样

整个 rule 文件是以一个 groups 规则组开头的:
```text
groups:
  [ - <rule_group> ]
```
里面可以包含各个规则组，记录规则 和 警报规则都可以写一起。 然后每一个组 `rule_group` 的参数如下:
```text
# 组名
name: <string>

# 规则评估间隔，没有设置就默认吃 prometheus 配置文件的这个 global.evaluation_interval 值
[ interval: <duration> | default = global.evaluation_interval ]

# 限制可触发警报数量，默认 0 表示不限制
# 如果有设置的话，一旦由该组规则触发的警报大于这个值，之后将不会触发这个规则引起的警报
[ limit: <int> | default = 0 ]

rules:
  [ - <rule> ... ]
```

接下来就是单个 rule 的配置, 分两种语法，一种是记录规则，一种是警报，我们本次只讲警报规则配置。
```text
# 警报规则的名称
alert: <string>

# 评估执行的 PromQL 语法表达式，结果应该是 true 或者 false
# 如果为 true，那么就会触发警报，警报的状态就会从 inactive 的未激活，转换为 pending 或者 firing
expr: <string>

# 等待期，一旦规则警报被触发之后，会变成 pending 中间态，这时候就会进入等待期
# 如果等待期内，该规则警报没有重新变成正常（也就是在这期间的评估结果有一次 expr 的执行结果为 false, 状态就会变成 inactive 的正常态), 状态就会变成结束态的 firing。 然后 prometheus 就会将这个警报发送给 alertmanager
# 如果为 0 的话，将直接跳过 pending 状态，一旦评估的 expr 为 true，那么马上变成 firing 状态
[ for: <duration> | default = 0s ]

# 警报标签,可自己定制，这个添加的标签，可以被模板提取出来
labels:
  [ <labelname>: <tmpl_string> ]

# 警报的注释，用于描述，比如后面发送邮件 或者 钉钉通知的话，用得着，也可以被模板提取出来
annotations:
  [ <labelname>: <tmpl_string> ]
```

这边需要注意的是， for 这个参数，一般我们不希望偶尔的网络波动导致 prometheus 抓取数据失败，就会马上触发警报(firing 状态)，从而就收到警报邮件了

因此我们就会设置一个比较合理的等待值。 在等待周期之内，如果触发评估规则了，并且 expr 表达式为 false，那么就会将状态变成 inactive，这时候就不会触发警报了

这个值要大于评估周期 `evaluation_interval` 才有价值，比如评估周期是 15s， 然后 for 设置的是 20s，然后触发顺序是这样子:
1. 触发评估周期，然后执行 expr 表达式，为 true， 警报状态从 `inactive` 变成 `pending`, 进入等待期(执行 for 子句)
2. 过了 15s，又触发评估周期，执行 expr 表达式，这时候就有两种情况:
  2.1 执行结果为 true，但是还没有到 for 的 20s，继续保持 pending 状态，然后又过了 5s，for 的等待期结束，状态从 `pending` 变成 `firing`, 触发警报， prometheus 将这个警报扔给 alertmanager 处理
  2.2 执行结果为 false，警报状态从 `pending` 变成 `inactive`， 警报恢复正常，重新等待下一轮评估

所以我们刚才的那个配置，其实就是我们有一个叫做 `test-1` 的规则警报组，里面有一条警报叫 `节点存活`, 评估周期是 15s， 抓取周期是 15s，等待周期是 1 分钟，然后判断的表达式是 `up{job="vmware-host"} == 0`

### 4. 警报的状态
prometheus 警报有 4 种状态:
- **inactive** -> 非活动状态，表示正在监控，但是还未有任何警报触发
- **pending** -> 警报被触发，但是在等待期内，如果等待期内没有恢复正常，就会转化为 firing
- **firing** -> 结束态，警报已经触发，prometheus 将按照配置将报警信息发送给 alertmanager
- **resolve** -> 恢复态，如果是 firing 状态，后面又恢复正常，警报又会重新变成 inactive。 这时候 prometheus 就会发送一个 resolve 的恢复态给 alertmanager，让 alertmanager 发送报警恢复邮件 (假设有配置的话)

虽然以上有 4 种，但是在 prometheus 后台的 alert 的监控显示上， 只有前三种，因为对于状态显示来说， 第四种的恢复态，其实就是第一种的 inactive。 也就是只有 alertmanager 才需要恢复态

### 5. 抓取周期(scrape_interval) 和 评估周期 (evaluation_interval)
在 prometheus server 中， 抓取周期 和 评估周期是分开的，可以各自设置。 但是最好评估周期要 大于等于 抓取周期。 不能小于， 不然就会出现评估的时候，取的是旧值，导致误报。

举个例子，假设抓取周期是 2 分钟， 然后评估周期是 15s， 等待周期是 1m，这时候就会出现这种情况:
1. 第一次抓取的时候，节点挂了， up 为 0
2. 15s 后触发评估周期，发现节点挂了，状态变成 pending， 启动等待周期 1m
3. 有过了 10s，节点启动正常
4. 有过了 5s，触发第二次评估，但是因为还没有触发抓取周期，评估的数据还是旧的，所以判断还是 pending
5. 又过了 45s，期间触发了 3 轮的评估，因为都是旧值，同时因为过了等待周期，状态变为 firing，触发报警
6. 又过了一分钟， 触发抓取周期，节点正常，同时时间周期也触发了评估周期，表达式返回 false， 状态从 firing 变成 inactive，警报恢复正常

以上就会有误报的情况。

所以再有等待期(for 不为 0)的情况下，其实就是 `for 等待周期` >= `评估周期` >= `抓取周期`

### 6. 操作触发警报
我们将 prometheus 这一台的 node_exporter 节点服务停止
```text
[root@VM-64-9-centos prometheus]# systemctl stop node_exporter.service
```
因为抓取周期和评估周期是 15s，所以这时候就可以看到 alert 状态变成 pending 了

![](7.png)

然后过了一分钟之后，过了等待期，彻底变成 firing 了

![](8.png)

同时实例监控这边，会有显示有一个节点 down  了

![](9.png)

接下来我们将这个 node_exporter 重新启动，可以看到一旦到新一轮的评估周期，就会变成正常的 inactive

接下来也试过先关掉服务，然后再过了等待期之前，再开启服务，这时候只要新一轮的评估周期在等待期内，一样会从 pending 变成 inactive，不会触发 firing

## alertmanager
我们已经能够从 prometheus server 中创建警报规则，并成功触发警报了， 但是我们还缺少触发之后的报警手段，比如发送 mail 到运维人员之类的手段
### 1. 安装
而这个就是 alertmanager 所要做的功能。 alertmanager 是一个独立的服务，因为我们要单独安装, 下载地址: [download](https://prometheus.io/download/), 一样采用二进制包的方式安装
```text
# 下载解压，重命名
[root@VM-64-9-centos ~]# cd /usr/local/
[root@VM-64-9-centos local]# wget https://github.com/prometheus/alertmanager/releases/download/v0.24.0/alertmanager-0.24.0.linux-amd64.tar.gz
[root@VM-64-9-centos local]# tar zxf alertmanager-0.24.0.linux-amd64.tar.gz
[root@VM-64-9-centos local]# mv alertmanager-0.24.0.linux-amd64  alertmanager

# 指定 owner
[root@VM-64-9-centos local]# chown root.root alertmanager -R

# 创建 system 启动脚本
[root@VM-64-9-centos local]# cat  /usr/lib/systemd/system/alertmanager.service
[Unit]
Description=Alertmanager
Documentation=https://prometheus.io/
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/alertmanager/alertmanager --config.file=/usr/local/alertmanager/alertmanager.yml
ExecStop=/bin/kill -s QUIT $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target

# 设置开机自启动，并启动服务
[root@VM-64-9-centos local]# systemctl daemon-reload
[root@VM-64-9-centos local]# systemctl enable alertmanager
[root@VM-64-9-centos local]# systemctl start alertmanager

# 查看服务
[root@VM-64-9-centos local]# netstat -anlp  | grep  alert
tcp6       0      0 :::9093                 :::*                    LISTEN      601/alertmanager

```

这样子就安装好了，alertmanager 有自己的查看后台， `http://ip:9093`

![](10.png)

可以看到，因为刚安装，也没有关联 prometheus， 所以没有警报信息

### 2. 设置配置文件
接下来我们配置一下配置文件，默认的配置文件是这样子的，在
```text
[root@VM-64-9-centos alertmanager]# cat /usr/local/alertmanager/alertmanager.yml
route:
  group_by: ['alertname']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 1h
  receiver: 'web.hook'
receivers:
  - name: 'web.hook'
    webhook_configs:
      - url: 'http://127.0.0.1:5001/'
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']
```
配置了以下规则:
1. 配置了几个时间设置间隔
2. 配置了分组方式(根据标签名 `alertname` 来分组)
3. 配置了 webhook 的接收者，方式是发送 webhook endpoint api
4. 配置抑制(`inhibit_rules`)规则

我们稍微改造一下，改成用 mail 来做接收者。 具体改为

```text
# 编辑调整
[root@VM-64-9-centos alertmanager]# cat alertmanager.yml 
global:
  smtp_smarthost: 'smtp.qq.com:25'
  smtp_from: 'zachke@foxmail.com'
  smtp_auth_username: 'zachke@foxmail.com'
  smtp_auth_password: 'ypgtxxxxxxxxxxxxx' 
  smtp_require_tls: false
route:
  group_by: ['alertname']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 10m
  receiver: default-receiver
receivers:
  - name: 'default-receiver'
    email_configs:
    - to: 'kebingzao@gmail.com'

# 使用工具检查 yml 格式
[root@VM-64-9-centos alertmanager]# ./amtool check-config alertmanager.yml
Checking 'alertmanager.yml'  SUCCESS
Found:
 - global config
 - route
 - 0 inhibit rules
 - 1 receivers
 - 0 templates

# 重启服务使得配置生效
[root@VM-64-9-centos alertmanager]# systemctl restart alertmanager.service 
```
其实就是去掉了抑制配置，将接收者改成 mail 邮箱，同时增加了 mail 配置，其他都不变。

二进制包安装的目标目录下有自带一个 yml 文件校验工具 `amtool`, 可以用它来校验 alertmanager 的配置文件是否格式正确
> 事实上， amtool 不仅仅是校验工具怎么简单，他还是一个可以配置 alertmanager 的 cli 命令工具， 感兴趣的可以看官方文档
i
### 3. prometheus 关联 alertmanager
接下来就要在 prometheus 管理 alertmanager 服务了，加 `alerting` 这个节点就行了，其他不变
```text
# 添加 alerting 小节，将 alertmanager 服务加上去
[root@VM-64-9-centos prometheus]# cat prometheus.yml 
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - "localhost:9093"

rule_files:
  - "rules/test-1.yml"

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "vmware-host"
    static_configs:
      - targets:
        - localhost:9100
        - 43.153.11.234:9100

# 检查语法
[root@VM-64-9-centos prometheus]# ./promtool check config prometheus.yml 
Checking prometheus.yml
  SUCCESS: 1 rule files found
 SUCCESS: prometheus.yml is valid prometheus config file syntax

Checking rules/test-1.yml
  SUCCESS: 1 rules found

# 更新配置文件
[root@VM-64-9-centos prometheus]# curl -X POST localhost:9090/-/reload
```















---












---

参考资料:
- [alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/)
- [alertmanager configuration](https://prometheus.io/docs/alerting/latest/configuration/)
- [prometheus alertmanager config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#alertmanager_config)
- [prometheus alerting rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)
- [通知模板参考](https://prometheus.io/docs/alerting/latest/notifications/)
- [Awesome Prometheus alerts](https://awesome-prometheus-alerts.grep.to/)



