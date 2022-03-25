---
title: AWS 的邮件发送服务 SES 订阅 SNS 来处理硬反弹邮件
date: 2019-06-05 11:28:04
tags: 
- aws
- ses
- sns
categories: aws 相关
---
## 前言
通过 {% post_link aws-ses-block-again %} 我们知道 AWS 之所以会禁掉我们的 SES 账号是因为我们的邮件的反弹率(bounce)太高了, 因此在服务稳定了之后，就打算好好研究一下 SES，看看有没有什么方式可以来处理硬反弹邮件。
## 电子邮件发送的 Amazon SES 通知
事实上，SES 在发生邮件退回和投诉的时候，会通过电子邮件发送 SES 通知。这个是默认启用的功能。 具体文档：[通过电子邮件发送的 Amazon SES 通知](https://docs.aws.amazon.com/zh_cn/ses/latest/DeveloperGuide/notifications-via-email.html)
<!--more-->
{% blockquote 通过电子邮件发送的 Amazon SES 通知 https://docs.aws.amazon.com/zh_cn/ses/latest/DeveloperGuide/notifications-via-email.html %}
启用电子邮件反馈转发
默认情况下会启用电子邮件反馈转发。如果您之前禁用了该功能，您可以使用本节中的以下步骤来启用它。使用 Amazon SES 控制台通过电子邮件启用退回邮件和投诉转发：
    1. 登录 AWS 管理控制台并通过以下网址打开 Amazon SES 控制台：https://console.aws.amazon.com/ses/。
    2. 在导航窗格中的身份管理下，选择电子邮件地址 (如果您想为电子邮件地址配置退回邮件和投诉通知) 或域 (如果您想为域配置退回邮件和投诉通知)。
    3. 在经验证的电子邮件地址或域的列表中，选择要为其配置退回邮件和投诉通知的电子邮件地址或域。
    4. 在详细信息窗格中，展开通知部分。
    5. 选择 Edit Configuration。在 Email Feedback Forwarding 下，选择 Enabled。
    注意:在此页面上进行的更改可能需要几分钟才能生效。
{% endblockquote %}
![1](1.png)
以下是一封收到了无效邮箱(邮箱退回)的邮件通知，而且下面会附上这封邮件的内容，
![1](2.png)
在这一封邮件的最下面，有提供了一个从SES黑名单中删除电子邮件地址的链接，这个是因为：
{% blockquote 从 Amazon SES 黑名单中删除电子邮件地址 https://docs.aws.amazon.com/zh_cn/ses/latest/DeveloperGuide/remove-from-suppression-list.html %}
Amazon SES 会保留一份黑名单，其中包含近期对任何 Amazon SES 客户造成“查无此人的邮件”的收件人电子邮件地址。如果尝试通过 Amazon SES 向黑名单中的地址发送电子邮件，您可以成功调用 Amazon SES，但 Amazon SES 会将该邮件视为“查无此人的邮件”，而不会尝试将其发送出去。与“查无此人的邮件”类似，黑名单退回邮件也会计入发送配额和退回邮件率。电子邮件地址可在黑名单上保留最多 14 天。
{% endblockquote %}
如果你确信黑名单中的某个地址有效，则可以使用以下过程将其从列表中删除，他提供了一个很简单的操作后台：
![1](3.png)
## Amazon SNS 通知
虽然 SES 在发生邮件退回或者投诉的时候，会向我们的发送者邮箱发送邮件通知。但是这样子并不利于我们收集无效邮箱的操作。因此它又提供了另一种类似于 webhook 机制的：[为 Amazon SES 配置 Amazon SNS 通知](https://docs.aws.amazon.com/zh_cn/ses/latest/DeveloperGuide/configure-sns-notifications.html)。
简单的来说，就是它允许你在 SNS 中创建一个主题，并有终端订阅它， 最后在 SES 中，启用这个订阅通知。
### SNS 中创建主题
Amazon Simple Notification Service (SNS) 这个是 AWS 的另一个服务:
{% blockquote Amazon Simple Notification Service https://aws.amazon.com/cn/sns/ %}
Amazon Simple Notification Service (SNS) 是一种高度可用、持久、安全、完全托管的发布/订阅消息收发服务，可以轻松分离微服务、分布式系统和无服务器应用程序。Amazon SNS 提供了面向高吞吐量、多对多推送式消息收发的主题。借助 Amazon SNS 主题，发布系统可以向大量订阅终端节点（包括 Amazon SQS 队列、AWS Lambda 函数和 HTTP/S Webhook 等）扇出消息，从而实现并行处理。此外，SNS 可用于使用移动推送、短信和电子邮件向最终用户扇出通知。
{% endblockquote %}
首先我们再在 SNS 后台创建一个主题：名字叫做 **ses_bounce_notification**
![1](4.png)
### SNS 创建订阅
接下来针对这个主题，创建一个订阅，因为我们要用 webhook 的方式来通知，所以选择协议 https，并输入要接收的 url： 比如 https://www.example.com/sns/ses
![1](5.png)
注意，这个url是要验证的，你必须证明是你自己的域名和接口。所以创建成功之后，SNS 会向这个接口发送一条验证 url，
{% blockquote %}
当您完成终端节点的订阅后，Amazon SNS 会向该终端节点发送一条订阅确认消息。终端节点上的代码必须检索来自订阅确认消息的 SubscribeURL 值，然后访问 SubscribeURL 指定的位置或让其对您可用，这样您就可以手动访问 SubscribeURL，例如，通过 Web 浏览器访问。
在订阅得到确认前，Amazon SNS 不会向终端节点发送消息。当您访问 SubscribeURL 时，响应将包含一个 XML 文档，该文档反过来包含用于指定订阅的 ARN 的 SubscriptionArn。
{% endblockquote %}
![1](6.png)
得到 url，然后拿到浏览器访问：
![1](7.png)
响应是一个 XML 文档，说明没错，接下来可以看到创建的订阅的状态变成确定了（之前是 pending 状态）
![1](8.png)
### SES 控制台配置通知
既然先决条件都弄好了，那么就可以在 SES 控制台配置了： [为 Amazon SES 配置 Amazon SNS 通知](https://docs.aws.amazon.com/zh_cn/ses/latest/DeveloperGuide/configure-sns-notifications.html)
在 Email Address 中选择要启用 SNS 通知的邮件，然后点击 **Notifications** 下的 **Edit Configuration** 按钮, 可以看到 **SNS topic Configuration** 下面有三个项:
- Bounces 邮件反弹率通知
- Complaints 邮件投诉通知
- Deliveries 邮件送达通知

因为我们只对反弹率做通知，所以只配置了 Bounces，选择主题 **ses_bounce_notification**:
![1](9.png)
### 完善接口通知
这样子就全部配好了，接下来就完善接口通知了，而且要注意一点的是，就算是 bounce 的通知，也有分好几种情况， [具体文档](https://docs.aws.amazon.com/zh_cn/ses/latest/DeveloperGuide/notification-contents.html#bounce-types)
- Undetermined 收件人的电子邮件提供商发送退回邮件消息
- Permanent 硬退回邮件
- Transient 软退回邮件

而我们只处理硬退回邮件，因此只考虑 bounceType 为 Permanent 的情况, 所以具体代码如下：
```go
// amazon ses 发布的事件
func (s *SNS) Ses() revel.Result {
	res := Res{}
	// 获取参数
	body, err := ioutil.ReadAll(s.Request.Body)
	defer s.Request.Body.Close()
	if err != nil {
		log.Error(err)
		return s.RenderJson(res)
	}
	// 解析用于验证 subscriber 身份的 subscribe confirmation
	confirmation := structs.SNSSubscribeConfirmation{}
	err = json.Unmarshal(body, &confirmation)
	if err != nil {
		log.Error(err)
	}
	if confirmation.SubscribeURL != "" {
		log.Info("==========================================")
		log.Infof("confirmation topic arn : %v", confirmation.TopicArn)
		log.Infof("confirmation sub url : %v", confirmation.SubscribeURL)
		log.Info("==========================================")
		return s.RenderJson(res)
	}

	// 解析正常 sns 发布的 ses 消息
	notification := structs.SNSNotificationFromSES{}
	err = json.Unmarshal(body, &notification)
	if err != nil {
		log.Error(err)
		return s.RenderJson(res)
	}
	// 处理通知
	switch notification.NotificationType {
	case "Bounce":
		// Permanent 表示硬退回，邮件被硬退回则对应的 mail_to 入库
		if notification.BounceInfo.BounceType == "Permanent" {
			for _, recp := range notification.BounceInfo.BouncedRecipients {
				models.InsertMailBounced(recp.EmailAddress)
			}
		}
	default:
		break
	}

	res.Code = 1
	return s.RenderJson(res)
}
```
逻辑很简单，如果收到硬退回邮件，那么就入库。
### 邮件发送服务过滤无效邮箱
这样子我们就有一张表来存放这些硬退回的无效邮件了，接下来就是在邮件发送服务那边针对这些无效邮件做过滤。简单的来说，就是发送前判断一下，这个邮箱是否在无效邮箱名单里面，如果在的话，就跳过，不发送。
```go
	// 如果 mail_to 在 mail_bounced 表中，不发送
	if IsMailBounced(q.MailTo) == true {
		log.Infof("Mail to %v is bounced", q.MailTo)
		return StatusMailBounced
	}
```
### 定期清除黑名单
SES 那边会对无效邮件黑名单保存14天，这个是因为有些邮箱当下可能是无效的，但是过一段时间可能就有效了，所以黑名单机制肯定会有一个过期时间的。而我们的处理方式跟 SES 一样，也是保存 14 天。因此我们会有一个计划任务，每天跑一次，将黑名单里面的超过14天的邮箱从黑名单剔除掉。

## 总结
自从做了这个无效邮箱黑名单机制之后，我们的 SES 的反弹率又降低了很多，之前是 5% 左右，现在是 2% 左右，可以说是进步了很多。
![1](10.png)

## 参考资料：
[Amazon SES 的 Amazon SNS 通知内容](https://docs.aws.amazon.com/zh_cn/ses/latest/DeveloperGuide/notification-contents.html#bounce-object)
[Amazon SES 的 Amazon SNS 通知示例](https://docs.aws.amazon.com/zh_cn/ses/latest/DeveloperGuide/notification-examples.html)
[为 Amazon SES 配置 Amazon SNS 通知](https://docs.aws.amazon.com/zh_cn/ses/latest/DeveloperGuide/configure-sns-notifications.html)
[电子邮件程序成功指标](https://docs.aws.amazon.com/zh_cn/ses/latest/DeveloperGuide/success-metrics.html)
[从 Amazon SES 黑名单中删除电子邮件地址](https://docs.aws.amazon.com/zh_cn/ses/latest/DeveloperGuide/remove-from-suppression-list.html)
[监控您的 Amazon SES 发送活动](https://docs.aws.amazon.com/zh_cn/ses/latest/DeveloperGuide/monitor-sending-activity.html)
[通过电子邮件发送的 Amazon SES 通知](https://docs.aws.amazon.com/zh_cn/ses/latest/DeveloperGuide/notifications-via-email.html)
[bounce rate 计算规则](https://www.reddit.com/r/aws/comments/7zvsw0/ses_complaintbounce_rate_calculation_periods/)