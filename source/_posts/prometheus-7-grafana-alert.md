---
title: 基于 prometheus 打造监控报警后台 (7) - grafana 后台创建警报
date: 2022-12-07 14:01:09
tags: 
- prometheus
- alertmanager
- grafana
categories: prometheus 相关
---
## 前言
之前我们已经尝试了在 prometheus 上创建警报，并且使用 alertmanager 来发送通知了: {% post_link prometheus-4-alertmanager %}

但是正常情况下，我们都是在 grafana 后台查看我们所创建的 dashboard 仪表盘。 同时 grafana 后台也可以显示我们在 prometheus 上创建的规则 (包含警报规则和记录规则)

> 不同版本的 grafana 可能界面会有差，甚至操作也会有区别，本节演示的 grafana 的版本是 `v9.2.5 (042e4d216b)`

## grafana 显示 prometheus 创建的规则
继续延伸上一节的规则配置 (配置了两个 rule 文件，一个是警报规则，一个是记录规则)，当我们查看 grafana 后台的 alerting 的时候，可以看到他也有把我们创建的规则一起显示出来
<!--more-->

![](1.png)

可以看到他展示的是 prometheus 作为数据源所创建的规则， 当有触发预警的时候，也可以正常显示

![](2.png)

但是其实 grafana 自己也可以创建警报规则，并且也可以发送警报通知。 而且这个是 grafana 这个系统自己创建的警报规则，跟 prometheus 不冲突，也就是你完全可以在 prometheus.yml 中将 alertmanager 和 rule 的配置去掉，不创建任何的警报规则:
```text
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          #- "localhost:9093"

rule_files:
  #- "rules/*.yml"
```

然后在 grafana 上创建警报规则就可以代替了

### 1. 创建警报规则
点击这个，到创建规则页面

![](3.png)

这边创建分为 4 个小步骤:

### 2. step 1 设置规则
选择数据源是 Prometheus， 因为我们只导入了这个数据源，所以默认就选择这个，然后设置警报规则
> 这边跟 prometheus 的警报规则一样，也可以用 记录规则的指标

我们设置一个跟 prometheus 的节点存活的表达式一样的规则:
```text
up{job="vmware-host"} == 0
```

![](4.png)

其实就是先选择指标 (A 面板)，然后再根据选择的指标选择表达式内容(B 面板)，所以规则就是 `up{job="vmware-host"} < 1 `
> 这边也可以设置很复杂的规则，或者可以使用 记录规则 先把复杂的计算 先变成指标，然后表达式就会比较好写

### 3. step 2 设置评估时间和等待时间
这个就很好理解了，其实就是 prometheus 的警报规则的 interval 和 for 参数，也就是评估周期和等待周期，这边设置为 30s 和 1m
> 这些概念在讲述 alertmanager 的时候，已经讲的很清楚了，这边不赘述

![](5.png)

### 4. step 3 设置警报名称和描述
这一步就跟我们在设置 prometheus 的警报规则一样，照填就行了

![](6.png)

```text
rules:
      - alert: 节点存活
        expr: up{job="vmware-host"} == 0
        for: 1m
        labels:
          serverity: critical
        annotations:
          summary: "[监控报警] 机器 {{ $labels.instance }} 挂了"
          description: "{{$labels.instance}} 宕机(当前值: {{ $value }})"
```

然后创建了一个 alert 的目录来存放这个规则，然后设置组别为 node

### 5. step 4 设置通知手段
这一块可以自定义，但是我们直接用默认的 root route 就行了，他跟 alertmanager 的配置一样，只不过他没有设置的时候，是会自动给默认的接收者来发送

![](7.png)

最后点击右上角的 `save and exit`, 最后就可以创建好了

![](8.png)

## 配置通知接收者
上面我们虽然配置了一个警报规则， 但是我们最后走的是 grafana 默认的接收者

![](9.png)

默认用的通知手段是 mail，然后联系人是 `grafana-default-email`, 所以我们接下来要配置这个邮件接收者的 smtp 服务，在 `Contact points` 标签页

如果我们没有配置的话，那么当警报触发的时候，就会出现这个错误

![](10.png)

### 1. 配置邮件 smtp 服务
这个是在 `/etc/grafana/grafana.ini` 这个配置文件的  smtp 节点配置

默认这一节是这样子配置
```text
[smtp]
;enabled = false
;host = localhost:25
;user =
# If the password contains # or ; you have to wrap it with triple quotes. Ex """#password;"""
;password =
;cert_file =
;key_file =
;skip_verify = false
;from_address = admin@grafana.localhost
;from_name = Grafana
# EHLO identity in SMTP dialog (defaults to instance_name)
;ehlo_identity = dashboard.example.com
# SMTP startTLS policy (defaults to 'OpportunisticStartTLS')
;startTLS_policy = NoStartTLS
```
后面改成用 qq 的邮箱代理发送， 所以改成这样子就可以了
> 关于怎么用 qq 邮箱配置 smtp 邮箱服务，可以看我之前的文章: {% post_link sentry-2 %}

```text
[smtp]
enabled = true
host = smtp.qq.com:25
user = zxxxxe@foxmail.com
# If the password contains # or ; you have to wrap it with triple quotes. Ex """#password;"""
password = ypgxxxxxxxxxxxx
;cert_file =
;key_file =
;skip_verify = false
from_address = zxxxxe@foxmail.com
;from_name = Grafana
# EHLO identity in SMTP dialog (defaults to instance_name)
;ehlo_identity = dashboard.example.com
# SMTP startTLS policy (defaults to 'OpportunisticStartTLS')
;startTLS_policy = NoStartTLS
```
然后重启一下 grafana
```text
[root@VM-64-9-centos grafana]# systemctl restart grafana-server.service
```
接下来我们就在后台确认一下邮箱配置是否正常, 在 `Contact points` 标签页点击 `grafana-default-email` 这一行最右边的编辑按钮，进入编辑验证页面，在 address 那边填入收件人邮箱，最后点击发送测试通知

![](11.png)

这样子我的 gmail 邮箱就可以收到这一份测试邮件了

![](12.png)

这样子邮箱发送设置就配置好了

### 2. 配置钉钉通知
跟 alertmanager 一个接收者(receivers) 可以有多种通知手段类似，grafana 的一个 Contact points 也可以有多个通知手段，我们刚才已经配置好了邮件发送设置了，接下来我们再配置一下 钉钉 通知

![](13.png)

因为本来就有支持 钉钉 webhook 通知，所以直接输入 url 就行了

这样子这个 Contact points 就有两种通知手段了，一个是邮箱，一个是 钉钉通知

![](14.png)

从上面的截图来看，这个是有配置 `send resolved` 的，就是警报恢复之后，还会有 resolved 通知

而且也可以看到，我们其实并没有配置 通知内容的模板， 默认的都是 grafana 默认的模板。

## 触发警报
接下来测试一下警报的触发，先把其中一台的 node_exporter 关掉

这时候看预警面板

![](17.png)

这时候就可以收到了

![](15.png)

![](16.png)

查看发送记录正常

![](18.png)

接下来恢复之后，也会有对应的恢复数据

![](19.png)

![](20.png)

不过我看了一下，之前配置跟 prometheus 一样的 summary， description 这两个字段并没有成功传递具体的变量值，而是变成 `<no value>`, 说明 grafana 找不到值。

所以变量应该不是用这种方式的, 对了一下发现 `$value` 这个值等于:
```text
[ var='B0' metric='Value' labels={__name__=up, instance=43.153.11.234:9100, job=vmware-host} value=0 ]
```
 
后面查了一下 [Templating labels and annotations](https://grafana.com/docs/grafana/latest/alerting/fundamentals/annotation-label/variables-label-annotation/), 发现确实有变量可以注入，但是有前提条件的

展开标签和注解时可以使用以下模板变量：

|姓名|	描述 |
|---|---|
|$labels	| 来自查询或条件的标签。例如，`$labels.instance` 和 `$labels.job`。这在规则使用 `classic condition` 时不可用。
|$values |	为此警报规则评估的所有 reduce 和数学表达式的值。 例如，`$values.A` 、`$values.A.Labels` 和 `$values.A.Value`，其中 A 是 reduce 或数学表达式的 refID。 如果规则使用 `classic condition` 而不是 reduce 和数学表达式，则 `$values` 包含条件的 refID 和位置的组合。
|$value |	警报实例的值字符串。例如，`[ var='A' labels={instance=foo} value=10 ]`。

因为我们用的是 `classic condition` 的匹配方式，所以没办法走 `$labels` 来注入模板变量，只能用 `$values` 的方式， 所以 refID 对于本例来说就是 `BO`, 所以 summary 和 description 应该改成:

```text
description: {{ $values.B0.Labels.instance }} 宕机(当前值: {{ $values.B0.Value }})
summary: [监控报警] 机器 {{ $values.B0.Labels.instance }} 挂了
```

> 这边之所以是 `B0`, 我猜测可能是因为我们这个警报采取的是 `B-expression` 的方式，并且是创建的第一个警报，所以是 `B0`， 所以如果是采用 `B-expression` 的方式创建的第二个警报，那么 id 就应该是 `B1`， 反之如果采用的是 `A-query` 的方式，那么就是 A 开头的，比如 `A0`, `A1` 按顺序排下去

![](21.png)

当然也可以从这个地方看，预览的时候触发预警的话，也可以看到完整的 value 字串，里面就有 id

![](27.png)

![](22.png)

这时候就可以看到警报的信息正常了

![](23.png)

## [自定义警报模板](https://grafana.com/docs/grafana/v9.2/alerting/contact-points/message-templating/)
刚才虽然我们可以发送警报了，但是警报的内容其实用的是默认的模板，mail 的还好，至少还可以看， 但是 钉钉通知的模板简直没法看， 所以还是要自定义一下警报的模板

grafana 的警报模板和 alertmanager 的警报模板的方式是一样，都是通过 [Go templating system](https://pkg.go.dev/text/template)
### 1. 默认的邮件 html 模板
grafana 有提供了一个默认的模板内容: [granafa default template](https://github.com/grafana/grafana/blob/main/pkg/services/ngalert/notifier/channels/default_template.go)

然后关于邮件的警报内容，用的是 html 文件，是放在这个路径: 
```text
<grafana-install-dir>/public/emails/ng_alert_notification.html
```
对于本例来说，就是:
```text
[root@VM-64-9-centos local]# head /usr/share/grafana/public/emails/ng_alert_notification.html
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
	<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
	<meta name="viewport" content="width=device-width" />
	
<style>body {
width: 100% !important; min-width: 100%; -webkit-text-size-adjust: 100%; -ms-text-size-adjust: 100%; margin: 0; padding: 0;
}
img {
```
我们可以直接修改这个 html 文件来改变默认的 邮件通知样式。

### 2. 创建自定义模板
在 grafana 创建通知模板，也很简单，在 `Contact points` 中点击 `New template` 按钮

![](24.png)

这时候我们就可以创建模板了，然后还很贴心的将模板的参数都列出来了

![](25.png)

接下来创建两个模板，一个是标题 `my.title`, 一个是内容 `my.message`

![](28.png)

具体内容:
```text
{{/* 定义消息体片段 */}}
{{ define "my_text_alert_list" }}{{ range . }}

【value】: {{.ValueString}}

【标签组】:
{{ range .Labels.SortedPairs }} - {{ .Name }} = {{ .Value }}
{{ end }}
【备注】:
{{ range .Annotations.SortedPairs }} - {{ .Name }} = {{ .Value }}
{{ end }}
{{ if gt (len .GeneratorURL) 0 }} 来源: {{ .GeneratorURL }}
{{ end }}
{{ if gt (len .SilenceURL) 0 }} 设置静默: {{ .SilenceURL }}
{{ end }}
{{ if gt (len .DashboardURL) 0 }} 仪表盘: {{ .DashboardURL }}
{{ end }}
{{ if gt (len .PanelURL) 0 }} 所属面板: {{ .PanelURL }}
{{ end }}
发生时间: {{ .StartsAt }}
{{ if eq .Status "resolved" }}
恢复时间: {{ .EndsAt }}
{{ end }}
{{ end }}{{ end }}


{{/* 定义消息体 */}}
{{ define "my.message" }}
{{ if gt (len .Alerts.Firing) 0 }}**--------Firing---------**
{{ template "my_text_alert_list" .Alerts.Firing }}
{{ end }}
{{ if gt (len .Alerts.Resolved) 0 }}**-------Resolved------**
{{ template "my_text_alert_list" .Alerts.Resolved }}
{{ end }}
{{ end }}
```

```text
{{ define "my.title" }}[监控报警]: [{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ if gt (.Alerts.Resolved | len) 0 }}, RESOLVED:{{ .Alerts.Resolved | len }}{{ end }}{{ end }}] {{ .GroupLabels.SortedPairs.Values | join " " }} {{ if gt (len .CommonLabels) (len .GroupLabels) }}({{ with .CommonLabels.Remove .GroupLabels.Names }}{{ .Values | join " " }}{{ end }}){{ end }}{{ end }}
```

然后接下来填充 mail 和 钉钉 通知的内容体:

![](29.png)

接下来就是测试预警和恢复了，将其中一台的 node_exporter stop 掉，就可以收到邮件了

![](30.png)

钉钉就是

![](31.png)

然后后面将其再开起来，就可以收到 resolved 的恢复通知了

![](32.png)

钉钉就是

![](33.png)

不过有看到这边的 `来源` 和 `设置静默` 的连接都是 localhost，所以点进去其实落地页是错的， 我们要改成正确的 url 地址， 这个是需要在 grafana 的配置文件上改的: `/etc/grafana/grafana.ini`， 其实就是将 domain 改成正确的 ip 地址，或者域名(本例就是 ip):
```text
# The public facing domain name used to access grafana from a browser
;domain = localhost
domain = 43.153.96.96
```

然后重启 grafana， 这样子链接就对了

![](39.png)

#### 遇到的几个坑
1. 刚开始是 `my.title` 和 `my.message` 是写在一起的，就跟我那时候在 alertmanager 那样子，但是 `my.title` 会一直报错:

```text
Dec 09 16:49:55 VM-64-9-centos grafana-server[15553]: logger=alerting.notifier.email t=2022-12-09T16:49:55.537719997+08:00 level=warn msg="failed to template email message" err="template: myTemplate:5:37: executing \"my.title\" at <toUpper>: invalid value; expected string"
```

后面发现，分开就可以。 但是写在一起就是不行，甚至我试过直接写在 mail 或者 钉钉 的消息体输入框，也是可以的，就是写在一起不行。 最后没办法就分开定义了

2. resolved 邮件的时候，发现 `.ValueString` 这个变量竟然为空，甚至 summary 里面的变量也没有，不知道是不是 bug 的原因，但是 alertmanager 是可以的， 后面查了一下 grafana 的 issues， 发现确实有人提了这个 issue: [valueString webhook body is empty](https://github.com/grafana/grafana/issues/45459), 并且在最新的版本依然问题存在，所以我猜测这个应该是 grafana 的 bug。

## 创建通知策略
grafana 的通知策略也允许跟 alertmanager 一样，创建这三个参数:
```text
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 10m
```

![](26.png)

这个就没啥好说的， 不再赘述

## 静默策略 和 分组
grafana 也有跟 alertmanager 一样的静默策略， 不再赘述

![](38.png)

分组策略也是跟 alertmanager 差不多，不再赘述

## 总结
从本节的实践来看， grafana 和 prometheus + alertmanager 的警报通知是相互独立， 不过都是读 prometheus 的数据。

然后整个特性和概念也是几乎一样。 如果之前熟悉了 alertmanager 的特性，再去用 grafana 创建警报的话，就会变得很容易

当然，在生产环境中， 看是要用 grafana 自己的警报系统，还是用 alertmanager ，这个要看个人的喜好。

如果是重度依赖监控 + 警报 的用户，我是觉得 alertmanager 其实会更加的灵活和可定制化。 毕竟 alertmanager 至少有两个功能是 grafana 的警报没有的:
1. 抑制规则
2. 可配置的有效时间和静默时间

但是 grafana 有一个优点就是，直接在 grafana 后台配置即可，不需要上服务器修改 `alertmanager.yml` 和 `rule/*.yml` 系列配置文件。 这一个是真的方便。

接下来我们将重心重新回到 prometheus 上，之前我们的数据采集都是用现成的 exporter，但是其实生产环境上， 我们的业务程序也是要抛送 prometheus 数据，那么怎么在我们的业务程序上使用 prometheus 来采集自定义数据呢?


---

参考资料:
- [Create a Grafana managed alerting rule](https://grafana.com/docs/grafana/latest/alerting/alerting-rules/create-grafana-managed-rule/)
- [Message templating](https://grafana.com/docs/grafana/v9.2/alerting/contact-points/message-templating/#message-templating)
- [Template data](https://grafana.com/docs/grafana/v9.2/alerting/contact-points/message-templating/template-data/)
- [Create a message template](https://grafana.com/docs/grafana/v9.2/alerting/contact-points/message-templating/create-message-template/)
- [Change Email Alert Template](https://sbcode.net/grafana/email-alert-template/)
- [How To Use Alert Message Templates in Grafana](https://community.grafana.com/t/how-to-use-alert-message-templates-in-grafana/67537)








