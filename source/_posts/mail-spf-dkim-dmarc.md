---
title: 电子邮件欺诈防护之 SPF+DKIM+DMARC
date: 2020-03-17 15:10:08
tags: security
categories: web安全
---
## 前言
早在之前，有收到一个白客的邮件，说我们发送电子邮件的域名缺少安全检查，让我们最好补上对应的安全检查：
```text
 I'm an independent cyber security researcher i have found multiple issues in your website.

 Vulnerability : Missing SPF 


 I am just looking at your SPF records then found following. SPF Records missing safe check which can allow me to send mail and phish easily any victim.

 PoC:

 <?php

 $to = "VICTIM@example.com";

 $subject = "Password Change";

 $txt = "Change your password by visiting here - [VIRUS LINK HERE]l";

 $headers = "From: https://karve.io";

 mail($to,$subject,$txt,$headers);

 ?>
 SPF record lookup and validation for: xxx.com
```
后面检查了一下，发现确实是这样子，我们的电子邮件没有加上对应的安全检查，很容易被别人冒发，从而有可能对我们的用户造成严重的电子邮件欺诈。

## 发送欺诈电子邮件的危害
举个例子，黑客可以通过这个站点 [emkei.cz](https://emkei.cz/) 来冒充我们的官方 support 邮箱来发送一些钓鱼邮件给用户。如果用户信以为真，那么就会对用户的利益造成损失，这就是电子邮件欺诈
<!--more-->
![png](1.png)

这时候我们的用户就会收到这一封冒充的邮件，如果用户相信了，就很容易被骗子给利用了。

![png](2.png)

所以电子邮件欺诈的危害性还是很高的。这方面的安全问题，一定要尽快解决。

## 电子邮件的缺陷
> 在解决这个问题的时候，我们要了解一下为啥电子邮件会有这个问题?

电子邮件系统的基础是简单邮件传输协议(`Simple Mail Transfer Protocol`, `SMTP`), SMTP是发送和中继电子邮件的互联网标准. 但是, 正如1981年最初设想的, `SMTP`不支持邮件加密、完整性校验和验证发件人身份。
 
 由于这些缺陷, 发送方电子邮件信息可能会被网络传输中的监听者截取流量读取消息内容导致隐私泄漏, 也可能遭受中间人攻击(`Man-in-the-Middle attack`, `MitM`)导致邮件消息篡改, 带来网络钓鱼攻击. 为了解决这些安全问题应对日益复杂的网络环境, 邮件社区开发了诸多电子邮件的扩展协议, 例如`STARTTLS`, `S/MIME`, `SPF`, `DKIM`和`DMARC`等协议。 当前的邮件服务厂商大都也是采用以上扩展协议的一种或几种组合, 辅以应用防火墙、贝叶斯垃圾邮件过滤器等技术, 来弥补电子邮件存在的安全缺陷.

你可以理解为 `SMTP` 本身有毛病，机密性和安全性都不够，所以就多了几个插件来帮他:
1. 提高传输的机密性，包括端对端加密 --> `STARTTLS`, `S/MIME`
2. 邮件的身份验证 --> `SPF`, `DKIM`, `DMARC`

电子邮件欺诈本质上就是邮件接收端没法判断这一封邮件来源是否合法。所以这边就涉及到了邮件的身份验证这一块。而且虽然`STARTTLS`和`SMTP-STS`保证了邮件在传输过程中的加密, 防止遭受窃听读取, 但是其仍无法解决发件方身份伪造、消息篡改等问题。 所以邮件的身份验证这一块尤为重要。

所以本节主要讲 `SPF`, `DKIM`,`DMARC` 这三个协议的设置并结合我们的站点来实践。

## SPF

### SPF 概念
发件人策略框架(`Sender Policy Framework`, `SPF`)是一种以IP地址认证电子邮件发件人身份的检测电子邮件欺诈的技术, 是非常高效的垃圾邮件解决方案。SPF允许组织授权一系列为其域发送邮件的主机, 而存储在DNS中的SPF记录则是一种TXT资源记录, 用以识别哪些邮件服务器获允代表本网域发送电子邮件。 SPF阻止垃圾邮件发件人发送假冒本网域中的“发件人”地址的电子邮件, 收件人通过检查域名的SPF记录来确定号称来自该网域的邮件是否来自授权的邮件服务器。 如果是, 就认为是一封正常的邮件, 否则会被认为是一封伪造的邮件而进行退回。 SPF还允许组织将其部分或全部SPF策略委托给另一个组织, 通常是将SPF设置委托给云提供商。

简单的来说，这个协议就是去核实邮箱源IP地址然后匹配它的dns中txt记录的spf信息，如果查找到，那么就是合法的。如果没有查找到，那就邮件的来源有问题。这时候一般邮箱接收端就会有三种处理方式:
1. 直接拒绝或者删除
2. 认为是垃圾
3. 正常的收件箱，但是邮件会有？的等警告，告诉收件者无法验证邮件的真实身份

至于哪一种，不同的邮箱接收端的策略不一样。 下面的图(网络图片)说的很清楚:

![png](3.png)

### SPF 设置
SPF 的设置非常简单，直接在公共的 dns 上进行配置即可。因为我们站点的域名 dns 是在 aws 的 router 53 中设置的。 而且因为我们的发送 mail 的邮箱有两种:
1. 发送提醒邮件的 `no-reply` 邮箱, 这一类邮件是接的 aws 的 ses 服务发送的，是程序自动发送的
2. 发送营销邮件的 support 和 sale 邮箱，这一类的邮箱是 gmail 邮箱，是人工发送的

所以 ses 和 google 这两个域名都要配置，所以我们就在 `router 53` 在 `xxx.com` 这个二级域名中在原有的 TXT 记录中，添加这一条:
```text
"v=spf1 include:_spf.google.com include:amazonses.com -all"
```

![png](4.png)

这样子 SPF 就配置好了。 是不是非常的简单。

### 测试 SPF 是否生效
配置之后，我们测一下 SPF 是否真的生效， 同样用 [emkei.cz](https://emkei.cz/) 这个站点发送一封欺诈邮件到 gmail ，这时候可以看到收到了：

![png](5.png)

邮件收到了，但是好像跟没有配置没啥差别??? 其实不然,我们打开 `显示原始邮件` 看下

![png](6.png)

发现端倪了，邮件发送源(`emkei.cz`)的 ip 地址是 `93.99.104.21`, 然后 gmail 邮件接收端通过 SPF 去查找我们站点域名的 SPF 配置，发现这个 ip 不存在 `_spf.google.com` 和 `amazonses.com` 旗下的所有的 ip 中。所以 SPF 就校验失败了。 而对于 gmail 邮件接收端来说， 如果 SPF 验证失败，就会提示一个问号, 表示 gmail 无法验证邮件的正确来源，使用者要谨慎。

### 理解 “all” 在SPF中的设置
SPF有多种不同的设置，98%的域名使用 ~all (softfail)，意思是如果spf条目与源邮件服务器不对应，标识邮件为软失败。下表将展示SPF中 all 的区别：

|参数	|结果|	含义|
|---|---|---|
|+all|	pass|	Permits all the email, like have nothing configured.（允许所有邮件，相当于未设置）|
|-all|	fail|	Will only mark the email like pass if the source Email Server fits exactly, IP, MX, etc. with the SPF entry.（仅通过匹配spf中正确配置IP、MX、etc.）|
|~all|	softfail|	Allows to send the email, and if something is wrong will mark it like softfail.（有错误的标识为软失败）|
|?all|	neutral|	Without policy（策略之外）|

而我们设置的就是直接设置为失败的 `-all`

### 检测 SPF 是否正确
配置完之后，我们也可以通过一些外部工具站点来检测我们的 SPF 配置是否正常：http://tools.wordtothewise.com/spf 这个站点将显示所有能使用域名发送的ip的概述

![png](21.png)

也就是说如果发件方的来源的 ip，不在这些 ip 列表里面。那么 SPF 检测就会通不了。就会变成 FAIL。

## DKIM 设置
虽然我们设置了 SPF, 但是结果也看到了，虽然在邮件详情中，标记了 `Fail`, 但是用户还是收到了，并且在正常的收件箱里面，只不过旁边多了一个问号提示 (其实上不同邮件接收端的针对 SPF 验证fail 的处理方式不一样，有些是放到垃圾箱中), 但是这个对于有些粗心的用户还是会被骗。 所以我们有必要再加强身份验证这一块。也就有了 DKIM 的设置。

### DKIM 概念
DKIM 域名密钥识别邮件标准(`Domain Keys Identified Mail`)是一种通过检查来自某域签名的邮件标头来判断消息是否存在欺骗或篡改的检测电子邮件欺诈技术。
 
DKIM 是利用加密签名和验证的原理, 在发件人发送邮件时候, 将与域名相关的私钥加密的签名字段插入到消息头, 收件人收到邮件后通过DNS检索发件人的公钥(`DNS TXT记录`), 就可以对签名进行验证, 判断发件人地址及消息的真实性. 需要注意的是, DKIM 只能验证发件人地址来源的真伪, 无法辨识邮件内容的真实性. 同时, 如果接收到无效或缺少加密签名的消息, DKIM 无法指定收件人采取什么措施. 此外, 私钥签名是对本域所有外发邮件进行普遍的签名, 是邮件中继服务器对邮件进行 DKIM 签名而不是真正的邮件发送人, 这意味着, DKIM 并不提供真正的端对端的电子签名认证。

简单的来说，就是使用 DKIM 发送的电子邮件包括一个 `DKIM-Signature` 标头字段，该字段包含邮件的加密签名表示形式 (里面有私钥)。接收邮件的提供商可以使用公有密钥（这在发件人的 DNS 记录中发现）对签名进行解码。然后，电子邮件提供商使用此信息来确定邮件是否是真实的。

举个例子， 比如我们正常的 gmail 或ses 发送邮件时，通过和域关联，自动生成的私钥加密邮件，生成DKIM-Signature签名，然后将DKIM-Signature签名及相关信息插入到邮件标头，发送给接收方, 而接收方收邮件时，通过DNS 查询获得公钥，验证邮件DKIM签名的有效性，从而确认在邮件发送的过程中，防止邮件被篡改，保证邮件内容的完整性。所以只要能用找到的公钥，去解标头字段里面的私钥信息，就说明是真实的，其实就是非对称的端对端加密，只不过我们把公钥暴露在域名的 DNS TXT记录中，接收邮件的提供商可以在公网上查到。 下面这张图可以解释这个过程:

![png](7.png)

### DKIM 设置
设置其实就是在 DNS TXT 设置公钥，因为我们用有用 aws ses 服务和 gmail， 所以要设置两种:

#### SES DKIM 设置
通过 [在 Amazon SES 中使用 DKIM 对电子邮件进行身份验证](https://docs.aws.amazon.com/zh_cn/ses/latest/DeveloperGuide/send-email-authentication-dkim.html) 我们可以用最简单的方式，就是 SES 提供的 [Easy DKIM](https://docs.aws.amazon.com/zh_cn/ses/latest/DeveloperGuide/send-email-authentication-dkim-easy.html), 简单来说，就是在已经验证过的域名的配置那边，点击生成 DKIM 的相关配置，这时候就会生成 3 条 cname 记录，然后把这三条 cname 记录放到 router 53 中就行了, 具体教程 [为域设置 Easy DKIM](https://docs.aws.amazon.com/zh_cn/ses/latest/DeveloperGuide/send-email-authentication-dkim-easy-setup-domain.html)

![png](8.png)

![png](9.png)

这样子 SES 的 DKIM 就配置好了。

#### Gmail DKIM 配置
接下来我们配置 Gmail 的，Gmail 默认会有默认的DKIM设置,但是官方建议使用自己的域名，目前 Gmail 也是通过我们自己的域名做的验证，直接一行 TXT 记录:

![png](10.png)

其中 p 字段后面就是一大段的私钥。

### 测试 DKIM 是否生效
一样发送一封正常的邮件到我的邮箱，如果 DKIM 有设置，并且成功的话，再邮箱详情中就会出现 PASS

![png](11.png)

以上截图说明我们有设置 DKIM，并且通过正确源发送的邮件 DKIM 的校验是正常的。

![png](12.png)

如果是冒充的，那么就不会是 PASS，而是 FAIL。

而且这边要注意一个细节的就是: 所有 DKIM 密钥存储在一个子域，命名为“_domainkey”。

上面截图中，给定DKIM签名字段 `d=xxx.com，s=google`，DNS查询为：`google.domainkey.xxx.com` (或者是 ses 对应的 domainkey 域名)，`google.domainkey.xxx.com` 这条 txt 记录会在我们 route 53 完成解析, 通过邮件表头，通过DNS查询公钥信息，如果和发送邮件的私钥进行匹配，通过了，那可能就不是垃圾邮件，没有通过，那邮件有可能是垃圾邮件，或邮件内容被篡改了。

## DMARC 设置
一般来讲，通过设定 SPF 和 DKIM,可以有效的阻挡伪造邮件了，但是，因为不同的邮件厂商，处理垃圾邮件的策略不同，会存在一定的风险，比如，一封伪造邮件，腾讯和网易处理的方式，可能是直接拒绝，而 gmail 通常还是会正常进入信箱，只是通过？提示，无法确定邮件来源。 这样的话，风险提示容易被忽视，会有极大的隐患，导致部分用户被欺骗,考虑到我们大多数用户对 gmail 的使用范围,这个问题还是要彻底解决，通过进一步了解邮件服务安全设定问题，发现 DMARC 恰好能解决我们的问题，我们期望最终的效果就是，伪造邮件为通过验证后，进入垃圾箱或直接拒收。

也就是之前哪怕我们设置了 SPF 和 DKIM, 但是不通过的邮件，可能因为不同的邮件厂商的关系，有可能是留在垃圾箱，也有可能留在收件箱，只不过带有问号。这个还是有一定的风险的。所以我们需要一种策略来告诉邮件接收端，如果收到伪造邮件，直接按照我指定的策略走。不要根据你们自己的喜好来走。这个就是 DMARC 设置。

### DMARC 概念
SPF 和 DKIM 中共同存在的问题是缺少有效的策略和反馈机制, 这两个协议并未定义如何处理来自声称为某域的未经身份验证的电子邮件, 如何处理第三方声称托管的某域的未经身份验证的邮件, 如何反馈和统计声称是某域的身份认证成功或失败的电子邮件。

DMARC 以域名为基础的邮件认证、报告和一致性标准(`Domain-based Message Authentication, Reporting, and Conformance`) 就是用来解决 SPF 和 DKIM 中存在的这些问题, DMARC 的主要用途在于设置“策略”, 这个策略包含接收到来自某个域未通过身份验证的邮件时应执行什么操作、该域授权的第三方提供商发送了未经身份验证的邮件时该如何处理. DMARC 还会让 ISP 发送有关某个域身份验证成功或失败的报告, 这些报告将发送至“rua”(汇总报告)和“ruf”(取证报告)中定义的地址中. 同时, DMARC 依靠出站邮件流配置的 SPF 记录和 DKIM 密钥来确保邮件来源及签名的完整性, 当未通过 SPF 或 DKIM 检查的邮件时便会触发 DMARC 策略。 

下面这张图可以说明这个过程:

![png](13.png)

### DMARC 常用参数
DMARC 配置的常用参数如下:

- **adkim**：（纯文本；可选的；默认为“r”）表明域名所有者要求使用严格的或者宽松的DKIM身份校验模式，有效值如下：
```text
r: relaxed mode
s: strict mode
```
- **aspf**：（纯文本；可选的；默认为“r”）表明域名所有者要求使用严格的或者宽松的SPF身份校验模式，有效值如下：
```text
r: relaxed mode
s: strict mode
```
- **fo**：故障报告选项（纯文本；可选的；默认为0），以冒号分隔的列表，如果没有指定“ruf”，那么该标签的内容将被忽略。
  - **0**：如果所有身份验证机制都不能产生“pass”结果，那么生成一份 DMARC 故障报告；
  - **1**：如果任一身份验证机制产生“pass”以外的结果，那么生成一份 DMARC 故障报告；
  - **d**：如果消息的签名验证失败，那么生成一份 DKIM 故障报告；
  - **s**：如果消息的 SPF 验证失败，那么生成一份SPF故障报告。
- **p**：要求的邮件接收者策略（纯文本；必要的）表明接收者根据域名所有者的要求制定的策略。
  - **none**：域名所有者要求不采取特定措施
  - **quarantine**：域名所有者希望邮件接收者将 DMARC 验证失败的邮件标记为可疑的。
  - **reject**：域名所有者希望邮件接收者将 DMARC 验证失败的邮件拒绝。
- **pct**：（纯文本0-100的整型；可选的，默认为100）域名所有者邮件流中应用DMARC策略的消息百分比。
- **rf**：用于消息具体故障报告的格式（冒号分隔的纯文本列表；可选的；默认为“afrf”）
- **ri**：汇总报告之间要求的间隔（纯文本32位无符号整型；可选的；默认为86400）.表明要求接收者生成汇总报告的间隔不超过要求的秒数。
- **rua**：发送综合反馈的邮件地址（逗号分隔的 DMARC URI纯文本列表；可选的）
- **ruf**：发送消息详细故障信息的邮件地址（逗号分隔的DMARC URI纯文本列表；可选的）
- **sp**：要求邮件接收者对所有子域使用的策略（纯文本；可选的），若缺省，则“p”指定的策略将应用到该域名和子域中。
- **v**：版本（纯文本；必要的）值为“DMARC1”，必须作为第一个标签。

### DMARC 配置
我们站点的具体配置一样在 router 53 上配置(google 和 ses 的配置都一样，所以只要统一配置一条即可), SES 有提供了具体的配置 [使用 Amazon SES 遵守 DMARC](https://docs.aws.amazon.com/zh_cn/ses/latest/DeveloperGuide/send-email-authentication-dmarc.html), 比如这个:
```text
"v=DMARC1;p=quarantine;pct=25;rua=mailto:dmarcreports@example.com"
```
简单地说，此策略告知电子邮件提供商执行以下操作：
- 查找“From”域为 example.com 且未通过 SPF 或 DKIM 身份验证的所有电子邮件。
- 将 25% 的未通过身份验证的电子邮件发送到垃圾邮件文件夹来进行隔离（您也可以使用 p=none 不采取任何操作；或使用 p=reject 直接拒收邮件）。
- 在摘要中发送有关未通过身份验证的所有电子邮件的报告（即，在特定时间段内汇总数据的报告，而不是为每个事件发送单独的报告）。电子邮件提供商通常每天发送一次此类汇总报告，但具体政策因提供商而异。

所以我们具体的配置如下:
```text
_dmarc.xxx.com.
```
```text
"v=DMARC1;p=reject;rua=mailto:dmarc-reports@xxx.com; adkim=s;aspf=s"
```

![png](14.png)

表示会执行以下操作:
- 严格要求 SPF 和 DKIM 的身份验证
- 一旦验证失败，那么将拒绝接收
- 拒绝之后，发送反馈到指定的邮箱

如果要查看你的 SPF 和 DKIM 是否有遵守 DMARC, 可以通过指令来查看:
```text
nslookup -type=TXT _dmarc.xxx.com
```

![png](15.png)

可以看到是没问题的。 当然如果不会配置 DMARC 的话，这边有个[配置助手](https://www.kitterman.com/dmarc/assistant.html) 可以帮助你配置:

![png](16.png)

### DMARC 测试效果
#### 放垃圾箱
既然配置好了，那么接下来就可以测试了，加上我们将策略改成放入垃圾箱 (p=quarantine)，这时候通过 [emkei.cz](https://emkei.cz/) 发送的邮件，当 SPF 没有通过的时候，就会放入垃圾箱

![png](17.png)

#### 拒绝接收
如果改为拒绝的话 (p=reject), 直接拒绝邮件，并每天发送退回邮件到发件箱，这里说下，比如 hacker 仿造了一封邮件发给用户，被拒绝了，那我们的正常的邮箱，比如support 会收到退信报告:

![png](18.png)

#### 正常
如果正常的话， 就会三个策略都会 PASS, 因为我们有两个发送源，所以两个都要试下
1. 这个是 ses 的 no-reply 系列邮件

![png](19.png)

2. 这个是 gmail 的 support 系列邮件

![png](22.png)

可以看到邮件 id 是不一样的，一个是 ses， 一个是 gmail

### DMARC 测试 SPF 和 DKIM 是否正常
我们知道在配置 DMARC 的时候， SPF 和 DKIM 一定要配好。虽然可以通过 `nslookup`。 但是也可以通过 外部的一些 第三方的站点来判断，比如这个 [DMARC Record Checker](https://dmarcian.com/dmarc-inspector/?domain=google.com#), 输入你的域名，就可以检查你的 DMARC 是否配置正常。

![png](20.png)

## 总结
通过 SPF + DKIM + DMARC, 我们的站点可以很好的解决电子邮件欺诈的问题了。


## 参考资料
- [在 Amazon SES 中使用 SPF 对电子邮件进行身份验证](https://docs.aws.amazon.com/zh_cn/ses/latest/DeveloperGuide/send-email-authentication-spf.html)
- [在 Amazon SES 中使用 DKIM 对电子邮件进行身份验证](https://docs.aws.amazon.com/zh_cn/ses/latest/DeveloperGuide/send-email-authentication-dkim.html)
- [使用 Amazon SES 遵守 DMARC](https://docs.aws.amazon.com/zh_cn/ses/latest/DeveloperGuide/send-email-authentication-dmarc.html)
- [How To use an SPF Record to Prevent Spoofing & Improve E-mail Reliability](https://www.digitalocean.com/community/tutorials/how-to-use-an-spf-record-to-prevent-spoofing-improve-e-mail-reliability)
- [基于DANE的电子邮件安全研究](http://www.c-s-a.org.cn/html/2018/7/6427.html)
- [DMARC指引](https://service.mail.qq.com/cgi-bin/help?subtype=1&&no=1001508&&id=16)
- [启用 DMARC](https://support.google.com/a/answer/2466563?hl=zh-Hans)

### SPF 的测试工具
- http://tools.wordtothewise.com/spf (将显示所有能使用域名发送的ip的概述)
- http://www.kitterman.com/spf/validate.html (简洁而有效，将展示dns条目和结果：pass, softfail, fail, neutral, etc. )
- http://mxtoolbox.com/ (经典)

### DKIM 的测试工具
- http://dkimvalidator.com/  (你需要发送一封邮件，如果测试通过，当你有DKIM配置，你将可以在网站上查看到DKIM部分的返回结果)
- http://dkimcore.org/tools/keycheck.html (填写selector 和域名进行检查，可以对是否正确设置 DKIM 进程检查)
- http://www.hhlink.com/检测DKIM/1 (通过给他发送一封邮件，他可以帮你验证 SPF 和 DKIM 是否正确)

### DMARC 的测试工具
- http://www.kitterman.com/dmarc/assistant.html (帮助你更好的配置 DMARC)
- https://dmarcian.com/dmarc-inspector/google.com (显示你的所有DMARC信息域)

### 伪造邮件发送测试站点
- https://emkei.cz/ (发送各种各样的冒牌邮件)
- http://tool.chacuo.net/mailanonymous (伪造邮件、任意发件人发送Email邮件、伪造邮件地址发送电子邮件、任意邮件地址发送电子邮件)