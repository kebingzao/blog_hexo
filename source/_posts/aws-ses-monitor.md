---
title: AWS 的邮件发送服务 SES 设置邮件事件监控
date: 2022-03-25 15:02:16
tags: 
- aws
- ses
- sns
categories: aws 相关
---
## 前言
前段时间运营需要对投递发送的邮件进行更详细的追踪用户行为分析，包括邮件发送的成功率，是否有被退回，反弹，是否有被打开，以及打开之后，是否有点击邮件里面的链接，点击的是哪个链接。

其实关于在邮件里面进行用户行为分析的话，早期的话，有用 ga 的方式试过:
- {% post_link mail-use-ga %}

这个的好处就是适合各种的邮件发送服务厂商， 坏处就是只能统计邮件打开率， 其他的比如退回，拒绝，反弹都不行。

不过我们用的邮件发送服务是 AWS 的 SES 服务，他其实是有匹配的邮件监控事件机制的。 甚至我们之前还结合另一个通知服务 SNS 来统计邮件的反弹率:
- {% post_link aws-ses-sns %}

本身邮件的反弹率就是邮件监控事件的一部分了，所以那时候就有 touch 到这个点了。
<!--more-->
## 使用 Amazon SES 事件发布监控电子邮件发送
这边文档写的很清楚 [使用 Amazon SES 事件发布监控电子邮件发送](https://docs.aws.amazon.com/zh_cn/ses/latest/dg/monitor-using-event-publishing.html), 为了能够以某个精细级别跟踪电子邮件发送，可以将 Amazon SES 设置为基于您定义的特征将电子邮件发送事件发布到 Amazon CloudWatch、Amazon Kinesis Data Firehose 或 Amazon Simple Notification Service。

您可以跟踪多种类型的电子邮件发送事件，包括发送、送达、打开、点击、退回、投诉、拒绝、呈现失败和送达延迟。此信息可用于操作和分析目的。例如，您可以将电子邮件发送数据发布到 CloudWatch 并创建跟踪电子邮件营销活动绩效的控制面板，也可以使用 Amazon SNS 在发生某些事件时向您发送通知。

而我们选择的其实就是 SES + SNS 通知的方式来记录电子邮件的事件类型。

### 可分析的几种事件
以下列表定义了与 Amazon SES 事件发布相关的术语。发送事件包括：
- **发送** – 发送请求成功，Amazon SES 将尝试将邮件发送到收件人的邮件服务器。（如果使用账户级别或全局抑制，SES 仍会将其计为发送，但会抑制送达。）
- **呈现失败** – 由于模板呈现问题，未发送电子邮件。当模板数据丢失或模板参数与数据不匹配时，可能会发生此事件类型。（此事件类型仅在您使用 `SendTemplatedEmail` 或 `SendBulkTemplatedEmail` API 操作发送电子邮件时发生。）
- **拒绝** – Amazon SES 已接受电子邮件，但确定它包含病毒，并且未尝试将其发送到收件人的邮件服务器。
- **送达** – Amazon SES 已将电子邮件成功送达至收件人的邮件服务器。
- **查无此人的邮件** – 收件人的邮件服务器永久拒绝了电子邮件。（只有当 Amazon SES 重试一段时间后仍无法发送邮件时才包括软退回邮件。）
- **投诉** – 电子邮件已成功送达收件人的邮件服务器，但收件人将其标记为垃圾邮件。
- **送达延迟** – 无法将电子邮件传送给收件人的邮件服务器，因为临时出现问题。例如，当收件人的收件箱已满，或者当接收电子邮件服务器遇到临时问题时，可能会发生传送延迟。
- **订阅** – 电子邮件已成功发送，但收件人通过单击电子邮件页眉中的 List-Unsubscribe 或页脚中的中的 Unsubscribe 链接更新了订阅首选项。
- **打开** – 收件人已收到邮件并在其电子邮件客户端中打开了邮件。
- **单击** – 收件人单击了电子邮件中包含的一个或多个链接。

## 具体实操
要设置事件发布的话， 我们需要在 SES 后台创建一个配置集， 具体教程: [设置 Amazon SES 事件发布](https://docs.aws.amazon.com/zh_cn/ses/latest/dg/monitor-sending-using-event-publishing-setup.html)

### 1. 创建配置集
因为非常简单，直接按照教程来就行了

![1](1.png)

这边需要注意的是，我有设置了一个重定向的自定义域名。 这个是因为如果要统计邮件的点击事件的话，其实 SES 是会在原有的链接上再进行一次包裹的，这样子第一次跳转的时候，其实是 SES 自己的域名，然后经过统计之后，再重定向到我们自己的链接。

{% blockquote 配置自定义域以处理打开和单击跟踪 https://docs.aws.amazon.com/zh_cn/ses/latest/dg/configure-custom-open-click-domains.html %}
当您使用事件发布来捕获打开和单击事件时，Amazon SES 将对您发送的电子邮件进行细微更改。
要捕获打开事件，Amazon SES 会将一个 1 x 1 像素的透明图像添加到每封电子邮件的底部。
每封电子邮件中的此图像都有一个唯一的文件名，且托管在由 Amazon SES 运营的服务器上。
为了捕获链接单击事件，Amazon SES 会将电子邮件中的链接替换为指向由 Amazon SES 运营的服务器的链接。
这会立即将收件人重定向到其预期目标。
某些 Amazon SES 客户可能希望使用自己的域（而不是由 Amazon SES 拥有和运营的域），为其收件人打造更综合的体验。
{% endblockquote %}

> 注意这一块链接的替换是 SES 自己处理的，我们只要设置好配置集就行了， 原始模板该设置什么连接还是照样设置

如果我们没有在创建配置集的时候，不指定自定义域名的话，那么邮件里面的链接就会是一个 SES 自己的域，很长又不好理解， 所以我们一般是自己再自定义一个我们自己的域名。

具体官方教程: [配置自定义域以处理打开和单击跟踪](https://docs.aws.amazon.com/zh_cn/ses/latest/dg/configure-custom-open-click-domains.html)

不过需要注意的是， 在配置集默认创建的自定义域名，其实是 http 协议，他不是 https 的。

![1](2.png)

如果要设置为 https 的话，是要 cloudfront 那边支持的， 简单的来说，我们在配置集那边配置好自定义域名之后呢，要再去 cloudfront 设置一个域名，将其指向到这个备用域名，

![1](3.png)

然后接下来要将这个 cloudfront 的源指向到这个自定义域名在配置集中对应的源，

![1](4.png)

![1](5.png)

其实他这个源，其实是每一个地区，就一个，这个有文档: [设置用于处理打开和单击链接的 HTTP 子域](https://docs.aws.amazon.com/zh_cn/ses/latest/dg/configure-custom-open-click-domains.html), 因为我们创建的区域是 美国东部（弗吉尼亚北部）， 所以其实就是 `r.us-east-1.awstrack.me`

最后在 router 53 将其 cname 记录设置为 cloudfront 的加速地址即可

![1](6.png)

这样子点击邮件内链接的自定义 https 域名就配置好了。

### 2. 配置触发事件的 SNS 通知事件通知
接下来我们就可以在新建的配置集那边点击 `Event destinations` 添加事件通知了， 这边就可以创建一个 sns 的主题了 (只有主题而已，没有订阅)

![1](7.png)

但是这样子还不够，因为这个主题刚创建完之后，还需要到 SNS 那边创建这个主题下的订阅和绑定对应的 webhook 推送接口，所以我们还需要去 SNS 后台那边针对这个主题创建对应的订阅和设置 webhook

![1](8.png)

这时候就需要涉及到对于端点域名的验证，这一块在 {% post_link aws-ses-sns %} 其实有说过，不再赘述， 同时协议选择 HTTPS， 然后之前在 SES 创建的 SNS 的 topic 的主题要选择标准，不能为 FIFO

这样子一旦触发这些规定的 event type，那么就会推送 webhook 到我们规定的接口来

### 3. 部署 webhook 接口
接口部署这一块没啥好讲的， 值得注意的是， 他每一个事件类型，对应的 post 的 body 的内容也不太一样， 比如发送事件的 json 串就是:
```javascript
{
  "eventType":"Complaint",
  "complaint": {
    "complainedRecipients":[
      {
        "emailAddress":"recipient@example.com"
      }
    ],
    "timestamp":"2017-08-05T00:41:02.669Z",
    "feedbackId":"01000157c44f053b-61b59c11-9236-11e6-8f96-7be8aexample-000000",
    "userAgent":"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/60.0.3112.90 Safari/537.36",
    "complaintFeedbackType":"abuse",
    "arrivalDate":"2017-08-05T00:41:02.669Z"
  },
  "mail":{
    "timestamp":"2017-08-05T00:40:01.123Z",
    "source":"Sender Name <sender@example.com>",
    "sourceArn":"arn:aws:ses:us-east-1:123456789012:identity/sender@example.com",
    "sendingAccountId":"123456789012",
    "messageId":"EXAMPLE7c191be45-e9aedb9a-02f9-4d12-a87d-dd0099a07f8a-000000",
    "destination":[
      "recipient@example.com"
    ],
    "headersTruncated":false,
    "headers":[
      {
        "name":"From",
        "value":"Sender Name <sender@example.com>"
      },
      {
        "name":"To",
        "value":"recipient@example.com"
      },
      {
        "name":"Subject",
        "value":"Message sent from Amazon SES"
      },
      {
        "name":"MIME-Version","value":"1.0"
      },
      {
        "name":"Content-Type",
        "value":"multipart/alternative; boundary=\"----=_Part_7298998_679725522.1516840859643\""
      }
    ],
    "commonHeaders":{
      "from":[
        "Sender Name <sender@example.com>"
      ],
      "to":[
        "recipient@example.com"
      ],
      "messageId":"EXAMPLE7c191be45-e9aedb9a-02f9-4d12-a87d-dd0099a07f8a-000000",
      "subject":"Message sent from Amazon SES"
    },
    "tags":{
      "ses:configuration-set":[
        "ConfigSet"
      ],
      "ses:source-ip":[
        "192.0.2.0"
      ],
      "ses:from-domain":[
        "example.com"
      ],
      "ses:caller-identity":[
        "ses_user"
      ]
    }
  }
}
```
这一块是有详细的文档的: [Amazon SES 发布到 Amazon SNS 的事件数据示例](https://docs.aws.amazon.com/zh_cn/ses/latest/dg/event-publishing-retrieving-sns-examples.html#event-publishing-retrieving-sns-send)

所以在接口逻辑中，我们就获取对应的 event type，然后入到表里面或者 log 文件里面就行了。

### 4. 发送邮件的代码逻辑
之前我们发送 ses 邮件的代码逻辑是不涉及到配置集的，只是普通的发送，也就是普通的 `sendEmail` 接口。

现在因为要进行事件追踪，所以要采用配置集，因此发送的 api 也不一样，是 `SendRawEmail`, 以 golang 为例，伪代码大概是这样子的

```javascript
  params := &ses.SendRawEmailInput{
    Destinations:         destinations,
    Source:               aws.String(from),
    ConfigurationSetName: aws.String(configSetName),
    RawMessage:           &message,
  }
  _, err := self.svc.SendRawEmail(params)
```

### 5. 测试结果
假设我将事件都录到表里面，最后的呈现结果就是，我可以看我的某一封邮件从发送到送达，再到打开，再到点击了哪些链接，一整串的操作顺序和事件点

> ses 这边记录的点击的 url，为了脱敏， 只包含最基础的 url，也就是 host+path， 不包含参数

![1](9.png)

## 需要注意的事项
这个是有常见的一些问题的: [Amazon SES 电子邮件发送指标常见问题](https://docs.aws.amazon.com/zh_cn/ses/latest/dg/faqs-metrics.html#faqs-metrics-clicks), 有几个可能会跟我们比较有关系。

### 1. 如果用户打开了某封电子邮件多次，或者单击了某封电子邮件中的某个链接多次，这些事件是否都会被单独跟踪？
如果收件人打开了某封电子邮件多次，那么 Amazon SES 会将每次打开计为一个唯一的打开事件。同样，如果收件人单击了同一链接多次，那么 Amazon SES 会将每次单击计为一个唯一的单击事件。但是，这些计数可能会因上述备注框中概述的情况而偏斜。

### 2. 您是否跟踪纯文本电子邮件的打开情况？
打开情况跟踪仅适用于 HTML 电子邮件。由于打开情况跟踪依赖于包含一个图像，因此无法为使用纯文本 (非 HTML) 电子邮件客户端打开电子邮件的用户收集打开情况指标。

### 3. 我能否禁用单击情况跟踪？
您可以通过将属性 ses:no-track 添加到电子邮件的 HTML 正文中的锚点标签，禁用单独链接的单击跟踪。例如，如果您链到 AWS 主页，那么正常的锚点链接将类似于以下这样：
```javascript
<a href="https://aws.amazon.com">Amazon Web Services</a>
```
要禁用该链接的单击跟踪，请修改为类似以下内容：
```javascript
<a ses:no-track href="aws.amazon.com">Amazon Web Services</a>
```
因为 `ses:no-track` 不是标准的 HTML 属性，所以 Amazon SES 会从到达您的收件人收件箱的电子邮件版本中自动删除它。

您还可以对使用特定配置集发送的所有邮件禁用单击跟踪。要禁用单击跟踪，请修改配置集事件目标，从而不捕获单击事件。

### 4. 在每封电子邮件中可以跟踪多少个链接？
单击跟踪系统最多可以跟踪 250 个链接。

### 5. 不跟踪我的电子邮件中的链接。为什么不跟踪？
Amazon SES 需要电子邮件中的链接包含正确编码的 URL。具体来说，链接中的 URL 必须符合 RFC 3986 的要求。如果电子邮件中的链接编码不正确，收件人仍会在电子邮件中看到链接，但 Amazon SES 不会跟踪链接的单击事件。

与不正确编码相关的问题通常出现在包含查询字符串的 URL 中。例如，如果电子邮件中的链接 URL 在查询字符串中包含非编码空格字符（例如以下示例中“John”和“Doe”之间的空格：http://www.example.com/path/to/page?name=John Doe），则 Amazon SES 不会跟踪此链接。然而，如果此 URL 改用编码空格字符（例如以下示例中的“%20”：`http://www.example.com/path/to/page?name=John%20Doe`），则 Amazon SES 将按预期跟踪它。

### 6. aws 自定义域名失效，需有预备方案，暂替原域名做中转
如果因为某种原因，比如 cloudfront 地址的证书过期了，导致 ses 链接点击的自定义域名失效了，我们需要有预备方案，暂替原域名做中转，

这时候就可以用 `openresty+nginx+lua` 来进行正确的转发跳转

```javascript
server {
    server_name mail-stat.example.com
    listen 80;
    listen       443 ssl http2;
    location / {
        default_type text/html;
        rewrite_by_lua_block {
            ngx.redirect("https://www.example.com/")
        }
    }
    location /CL0 {
        default_type text/html;
        rewrite_by_lua_block {
                local redirect_uri = string.match(ngx.var.request_uri,"/CL0/(.+)")
                local function  unescape (str)
                    str = string.gsub (str, "+", " ")
                    str = string.gsub (str, "%%(%x%x)", function(h) return string.char(tonumber(h,16)) end)
                    return str
                end
                ngx.redirect(unescape(redirect_uri))
        }
    }
}
```

### 7. 为什么有些用户在打开邮件中链接地址时，报400错误
首先我们需要了解 ses 什么时候才会生成跟踪链接:

Amazon SES 需要电子邮件中的链接包含正确编码的 URL。具体来说，链接中的 URL 必须符合 RFC 3986 的要求。如果电子邮件中的链接编码不正确，收件人仍会在电子邮件中看到链接，但 Amazon SES 不会跟踪链接的单击事件。

与不正确编码相关的问题通常出现在包含查询字符串的 URL 中。例如，如果电子邮件中的链接 URL 在查询字符串中包含非编码空格字符（例如以下示例中“John”和“Doe”之间的空格：`http://www.example.com/path/to/page?name=John Doe`），则 Amazon SES 不会跟踪此链接。然而，如果此 URL 改用编码空格字符（例如以下示例中的“%20”：`http://www.example.com/path/to/page?name=John%20Doe`），则 Amazon SES 将按预期跟踪它。

比如，我们邮件服务生成验证邮件时，会生成类似的邮件验证地址：

```
https://id.example.com/signinverify?code=E68JJD9MV3&mail=jm-kkbbzz%40163.com&name=OPPO A33m
```

这里看到query string中mail的值做了url编码，而name的值并没有做url编码（不论什么语言，新的实现方式都要求使用rfc3986的标准，对于旧编码兼容旧标准），所以像以上的邮件验证地址，是不会被ses跟踪的。如果是下面这个的话

```
https://id.example.com/signinverify?code=E68JJD9MV3&mail=jm-kkbbzz%40163.com&name=OPPO
```

这个链接地址满足rfc3986的url串，是会被ses跟踪。

那么为啥有些用户在打开邮件中的链接地址的时候，会报 400 错误? 我们了解到了什么时候ses才会生成跟踪链接，而大部分我们接收到用户报400错误的邮件跟踪地址都是因为邮件客户端url串解码的问题。

比如139邮箱客户端，他会对url做了二次解码，导致我们ses生成的URL地址无法被ses统计服务正确解析URL，所以报了http 400 bad request，也就是请求格式不正确。

以下就是在 139 邮箱客户端点击访问的最终的 url:
```text
GET https://mail-stat.example.com/CL0/https://id.example.com/signinverify?code=NZJUK4YMRJST&mail=kkbbzz1%40163.com&name=KOZ-AL00/1/010001816026a82e-ed8bf730-9218-4bda-b6f6-f4cee9b40211-000111/nVJFAfXMPlF3tMn_oOcfaMxq9lDgIr3xTGYzY=252 HTTP/1.1
```

但是正确的资源 url 应该是:
```text
GET https://mail-stat.example.com/CL0/https:%2F%2Fid.example.com%2Fsigninverify%3Fcode=NZJUK4YMRJST%26mail=kkbbzz1%2540163.com%26name=KOZ-AL00/1/010001816026a82e-ed8bf730-9218-4bda-b6f6-f4cee9b40211-000111/nVJFAfXMPlF3tMn_oOcfaMxq9lDgIr3xTGYzY=252 HTTP/1.1
```

从上面可以看出139邮箱在使用外部浏览器或内置浏览器访问url时，提前做了一次url解码，导致 mail-stat 读取到时没有找到正确的原始地址。

所以建议用户使用浏览器或是其他邮箱客户端访问 (web、iOS邮箱客户端/网易邮箱大师、安卓电子邮件客户端/邮箱大师，都能正常打开)。

## 总结
通过 SES + SNS 的邮件事件监控机制，我们可以了解到我们发出去的每一封邮件的具体用户行为结果，甚至是哪些投递是失败的情况， 更好的进行运营分析

---

参考资料:
- [使用 Amazon SES 事件发布监控电子邮件发送](https://docs.aws.amazon.com/zh_cn/ses/latest/dg/monitor-using-event-publishing.html)
- [Amazon SES 电子邮件发送指标常见问题](https://docs.aws.amazon.com/zh_cn/ses/latest/dg/faqs-metrics.html#faqs-metrics-clicks)
- [Amazon SES 发布到 Amazon SNS 的事件数据示例](https://docs.aws.amazon.com/zh_cn/ses/latest/dg/event-publishing-retrieving-sns-examples.html)




