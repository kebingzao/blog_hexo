---
title: 浅谈之 - web 安全渗透测试方案
date: 2022-01-11 20:13:50
tags: security
categories: 服务浅谈系列
---
## 前言
前段时间公司有请了安全服务公司来给公司的项目做 安全渗透， 尤其是 web 应用的安全渗透。 其中可测试点非常多。 

其实关于 web 安全方面， 我之前也写了一些文章: [分类 - web 安全](https://kebingzao.com/categories/web%E5%AE%89%E5%85%A8/), 但是更多的是各种的安全的案例以及对应的解决方案。好像还没有从大的方向上来总结和归纳。

所以本次我也想总结一下 web 安全渗透到底是什么，为什么要做， 以及怎么做?

## 什么是渗透测试
渗透测试（Penetration Test）是指安全工程师尽可能完整地摸拟黑客使用的漏洞发现技术和攻击手段，对 目标网络/系统/主机/应用 的安全性做深入的探测，发现系统最脆弱的环节的过程，渗透测试能够直观的让管理人员知道自己网络面临的问题。

<!--more-->

渗透测试是一种专业的安全服务，类似于军队里的 `实战演习` 或者 `沙盘推演`，通过实战和推演，让用户清晰了解目前网络的脆弱性、可能造成的影响，以便采取必要的防范措施。

渗透测试的方式分为以下几种:
### 1. 黑盒测试
黑箱测试又被称为所谓的 `zero-knowledge testing`, 渗透者完全处于对系统一无所知的状态，通常这类型测试，最初的信息获取来自于DNS、Web、Email及各种公开对外的服务器。
### 2. 白盒测试
白盒测试与黑箱测试恰恰相反，测试者可以通过正常的渠道向被单位取得各种资料，包括网络拓扑、员工资料甚至网站或其它程序的代码片断，也能够与单位的其它员工（销售、程序员、管理者……）进行面对面的交流。这类测试的目的是模拟企业内部雇员的越权操作。
### 3. 隐秘测试
隐秘测试是对被测单位而言的，通常情况下，接受渗透测试的单位网络管理部门会收到通知：在某些时段进行测试，因此能够监测网络中出现的变化，但隐性测试则被测单位也仅有极少数人知晓测试的存在，因此能够有效地检验单位中的信息安全事件监控、响应、恢复等做的是否到位。

## 渗透测试目的
渗透测试是完全模拟黑客可能使用的攻击技术和漏洞发现技术，对目标系统的安全作深入的探测，发现系统最脆弱的环节。渗透测试能够直观的让管理人员知道自己网络所面临的问题。

渗透测试主要依据安全专家已经掌握的安全漏洞和安全检测工具，模拟黑客的攻击方法在客户的授权和监督下对客户的系统和网络进行非破坏性质的攻击性测试。

渗透测试主要利用网络安全扫描器、专用安全测试工具和富有经验的安全工程师对网络中的核心服务器及重要的网络设备，包括服务器、网络设备、防火墙等进行非破坏性质的模拟黑客攻击，目的是入侵系统和获取机密信息并将入侵的过程和细节产生报告给用户。

渗透测试和工具扫描可以很好的互相补充。工具扫描具有很好的效率和速度，但是存在一定的误报率和漏报率，并不能发现高层次、复杂、并且相互关联的安全问题；渗透测试需要投入的人力资源较大、对测试者的专业技能要求很高（渗透测试报告的价值与渗透测试者的专业技能直接相关），但是非常准确，可以发现逻辑性更强、更深层次的弱点。

## 渗透测试流程
渗透测试服务通过远程利用目标应用系统等安全弱点，模拟真正的黑客入侵攻击方法，以人工渗透为主，以漏洞扫描工具为辅，在保证整个渗透测试过程都在可以控制和调整的范围之内尽可能的获取目标信息系统的管理权限以及敏感信息。

一般的流程是这样子的

![](1.png)

### 1. 信息收集
信息收集是指渗透实施前尽可能多地获取目标信息系统相关信息，例如网站注册信息、共享资源、系统版本信息、已知漏洞及弱口令等等。通过对以上信息的收集，发现可利用的安全漏洞，为进一步对目标信息系统进行渗透入侵提供基础。

### 2. 弱点分析及功能分析
对收集到的目标信息系统可能存在的可利用安全漏洞或弱点进行分析，并确定渗透方式和步骤实施渗透测试。

### 3. 获取权限
对目标信息系统渗透成功，获取目标信息系统普通权限。

### 4. 权限提升
当获取目标信息系统普通管理权限后，利用已知提权类漏洞或特殊渗透方式进行本地提权，获取目标系统远程控制权限。

## 渗透测试的几种类型
1. web 应用安全渗透
2. 接口类应用安全渗透
3. 前端应用安全渗透
4. 系统软件安全渗透
5. 硬件设备安全渗透
6. 其他类型应用安全渗透(业务场景，网络层，数据存储，中间件)

本文主要是总结 web 安全渗透的，所以只包含前三种，也就是:
1. web 应用安全渗透
2. 接口类应用安全渗透
3. 前端应用安全渗透

还有就是 其他类型应用安全渗透的 web 业务场景的渗透测试。

## 测试工具说明
渗透测试人员模拟黑客入侵攻击的过程中使用的是操作系统自带网络应用、管理和诊断工具、黑客可以在网络上免费下载的扫描器、远程入侵代码和本地提升权限代码以及自主开发的安全测试工具。

这些工具经过全球数以万计的程序员、网络管理员、安全专家以及黑客的测试和实际应用，在技术上已经非常成熟，实现了网络检查和安全测试的高度可控性，能够根据使用者的实际要求进行有针对性的测试。

但是安全工具本身也是一把双刃剑，为了做到万无一失，渗透测试工程师也要针对系统可能出现的不稳定现象提出相应对策，以确保服务器和网络设备在进行渗透测试的过程中保持在可信状态。


### 1. 系统自带工具
以下列出了主要应用到的系统自带网络应用、管理和诊断工具，渗透测试工程师将用到但不限于只使用以下系统命令进行渗透测试。

|工具名称| 风险等级 | 获取途径 | 主要用途 | 存在风险描述 | 风险控制方法 |
|---|---|---|---|---|---|
| ping | 无 | 系统自带 | 获取软件部署的信息 | 无 | 无 |
| telnet | 无 | 系统自带 | 登录系统 | 无 | 无 |
| ftp | 无 | 系统自带 | 传输文件 | 无 | 无 |
| tracert | 无 | 系统自带 | 获取网络信息 | 无 | 无 |
| net use | 无 | 系统自带 | 建立连接 | 无 | 无 |
| net user | 无 | 系统自带 | 查看系统用户 | 无 | 无 |
| echo | 无 | 系统自带 | 文件输出 | 无 | 无 |
| nslookup | 无 | 系统自带 | 获取软件部署的主机信息 | 无 | 无 |
| IE | 无 | 系统自带 | 获取 web 信息，进行 SQL 注入 | 无 | 无 |


### 2. 自由软件和渗透测试工具
> 以下工具不限于只用来做 web 安全渗透

以下列出了渗透测试中常用到的网络扫描工具、网络管理软件等工具，这些工具都是网络上的免费软件。渗透测试工程师将可能利用到但是不限于利用以下工具。

远程溢出代码和本地溢出代码需要根据具体系统的版本和漏洞情况来选择，由于种类繁杂并且没有代表性，在这里不会一一列出。

|工具名称| 风险等级  | 主要用途 | 存在风险描述 | 风险控制方法 |
|---|---|---|---|---|
|[nc](https://zh.wikipedia.org/wiki/Netcat) |	无 |	 对软件部署的主机扫描端口连接工具|	无 |	无 |
|远程溢出工具|	中	| 通过对软件部署的主机漏洞远程进入系统	|溢出程序可能造成服务不稳定 | 	备份数据，服务异常时重启服务
|本地溢出工具|	中 |	通过对软件部署的主机漏洞本地提升权限|	溢出程序可能造成服务不稳定| 	备份数据，服务异常时重启服务
|[Arachni](https://www.arachni-scanner.com/)|	低 | 	高性能安全Web扫描程序| 	可能造成网络资源的占用	| 如果软件部署的主机负载过高，停止扫描
| [XssPy](https://github.com/faizann24/XssPy)|	低 | 	XSS（跨站脚本）漏洞扫描器	| 可能造成网络资源的占用	| 如果软件部署的主机负载过高，停止扫描
| [w3af](http://w3af.org/) |	低	 | Web应用程序进行审计	|无 |	无
| [Nikto](https://cirt.net/Nikto2)	| 低 | 	扫描Web服务器配置错误、插件和Web漏洞|	可能造成网络资源的占用 |	如果软件部署的主机负载过高，停止扫描
|[Wfuzz](https://github.com/xmendez/wfuzz)	| 无 |	对字段的HTTP请求中的数据进行模糊处理，对Web应用程序进行审查 |	无 | 	无 |
|[OWASP ZAP](https://www.zaproxy.org/)	| 低 |	在浏览器和Web应用程序之间拦截和检查消息	 |无 |	无
| [Wapiti](https://wapiti.sourceforge.io/)	| 低 | 	扫描特定的目标网页，寻找能够注入数据的脚本和表单	| 可能造成网络资源的占用|	如果软件部署的主机负载过高，停止扫描
| [Vega](https://subgraph.com/vega/)|	低 | 	查找XSS，SQLi、RFI和其它的漏洞|	可能造成网络资源的占用|	如果软件部署的主机负载过高，停止扫描
|[SQLmap](https://sqlmap.org/)	| 低 |	 对后台数据库进行渗透测试和漏洞查找	| 可能造成网络资源的占用	 |如果软件部署的主机负载过高，停止扫描
|[Grabber](https://rgaucher.info/beta/grabber/)	| 无 | 	管理和运行渗透工具分析器的流行安全工具框架 |	无 |	无
| [OWASP Xenotix XSS](https://github.com/ajinabraham/OWASP-Xenotix-XSS-Exploit-Framework)|	低 |	用于查找和利用Web跨站点脚本的高级框架|	可能造成网络资源的占用|	如果软件部署的主机负载过高，停止扫描
| [Burpsuite](https://portswigger.net/burp)	| 低 |	用于渗透Web应用程序的集成平台	|可能造成网络资源的占用	| 如果软件部署的主机负载过高，停止扫描


## 测试环境说明
为更好的保护系统安全，实际生产环境和测试开发环境应该隔离。在生产环境中的任何改动，都需要严格遵循变更管理流程，做到执行人、执行时间、执行对象和具体改动均记录在案，并有企业信息安全部门进行事前审核和事后审计。

技术人员一般不要直接调试生产系统，可以在测试环境中调试完成后再更新生产系统，以避免调试过程中开启某些接口、更改某些配置或者保存某些调试信息造成安全隐患。如果非要在线调试生产系统，而且需要保存调试信息时，应避免将调试信息直接保存到服务器本地，同时调试完成后在第一时间删除相关调试信息并恢复系统配置。

## Web应用安全渗透测试详情
[OWASP Top 10](https://zhuanlan.zhihu.com/p/393635352) 的首要目的是教导开发人员、设计人员、架构师、管理人员和企业组织，让他们认识到最严重Web应用程序安全弱点所产生的后果。Top 10 提供了这些高风险问题的描述。渗透测试工程师应该重点针对 OWASP 漏洞清单内容逐一进行检查。

### OWASP Top 10
具体的 OWASP Top 10 的漏洞缺陷如下:

#### 1. 注入
将不受信任的数据作为命令或查询的一部分发送到解析器时，会产生诸如SQL注入、NoSQL注入、OS注入和LDAP注入的注入缺陷。攻击者的恶意数据可以诱使解析器在没有适当授权的情况下执行非预期命令或访问数据。

##### 危害
注入可以导致数据丢失或被破坏，缺乏可审计性或拒绝服务。注入漏洞有时甚至可导致完全接管主机

##### 常见的注入
- sql注入
- –os-shell
- LDAP（轻量目录访问协议）
- xpath（XPath即为XML路径语言，它是一种用来确定XML（标准通用标记语言的子集）文档中某部分位置的语言）
- HQL注入

##### 如何防范
1. 使用安全的API，避免使用解释器
2. 对输入的特殊的字符进行ESCAPE转义处理
```text
例子：LIKE ‘%M%’ ESCAPE ‘M’

使用ESCAPE关键字定义了转义字符“M”，告诉DBMS将搜索字符串“%M%”中的第二个百分符（%）作为实际值，而不是通配符
```
3. 使用白名单来规范化的输入验证方法

#### 2. 失效的身份认证
通常，通过错误使用应用程序的身份认证和会话管理功能，攻击者能够破译密码、密钥或会话令牌，或者利用其它开发缺陷来暂时性或永久性冒充其他用户的身份。

##### 危害
这些漏洞可能导致部分甚至全部账户遭受攻击，一旦攻击成功，攻击者就能执行合法的任何操作

##### 如何防范
- 使用内置的会话管理功能
- 通过认证的问候
- 使用单一的入口点
- 确保在一开始登录SSL保护的网页

#### 3. 敏感数据泄露
许多Web应用程序和API都无法正确保护敏感数据，例如：财务数据、医疗数据和PII数据。攻击者可以通过窃取或修改未加密的数据来实施信用卡诈骗、身份盗窃或其他犯罪行为。

未加密的敏感数据容易受到破坏，因此，我们需要对敏感数据加密，这些数据包括：传输过程中的数据、存储的数据以及浏览器的交互数据。

#### 4. XML外部实体(XXE)
XXE 全称为 `XML External Entity attack`, 即XML(可扩展标记语言) 外部实体注入攻击，

许多较早的或配置错误的XML处理器评估了XML文件中的外部实体引用。攻击者可以利用外部实体窃取使用URI文件处理器的内部文件和共享文件、监听内部扫描端口、执行远程代码和实施拒绝服务攻击。

#### 5. 失效的访问控制
未对通过身份验证的用户实施恰当的访问控制。攻击者可以利用这些缺陷访问未经授权的功能或数据，例如：访问其他用户的帐户、查看敏感文件、修改其他用户的数据、更改访问权限等。

##### 危害
这种漏洞可以损坏参数所引用的所有数据

##### 如何防范
1. 使用基于用户或会话的间接对象访问，这样可防止攻击者直接攻击未授权资源
2. 访问检查：对任何来自不受信源所使用的所有对象进行访问控制检查
3. 避免在url或网页中直接引用内部文件名或数据库关键字
4. 验证用户输入和url请求，拒绝包含`./`  `…/` 的请求

#### 6. 安全配置错误
安全配置错误是最常见的安全问题，这通常是由于不安全的默认配置、不完整的临时配置、开源云存储、错误的HTTP 标头配置以及包含敏感信息的详细错误信息所造成的。

因此，我们不仅需要对所有的操作系统、框架、库和应用程序进行安全配置，而且必须及时修补和升级它们。

之前就出现过几起因为不安全配置导致的安全缺陷问题:
- {% post_link s3-bucket-index %}
- {% post_link s3-subdomain-takeover %}
- {% post_link ssl-caa %}

##### 危害
系统可能在未知的情况下被完全攻破，用户数据可能随着时间被全部盗走或篡改。甚至导致整个系统被完全破坏

##### 如何防范
1. 自动化安装部署
2. 及时了解并部署每个环节的软件更新和补丁信息
3. 实施漏洞扫描和安全审计

#### 7. 跨站脚本（xss）
xss攻击全称为跨站脚本攻击,

当应用程序的新网页中包含不受信任的、未经恰当验证或转义的数据时，或者使用可以创建 HTML 或 JavaScript 的浏览器API 更新现有的网页时，就会出现 XSS 缺陷。

XSS 让攻击者能够在受害者的浏览器中执行脚本，并劫持用户会话、破坏网站或将用户重定向到恶意站点。

##### 危害
攻击者在受害者浏览器中执行脚本以劫持用户会话，插入恶意内容，重定向用户，使用恶意软件劫持用户浏览器等

XSS 的漏洞有好几种类型，包含 存储型， 反射型， DOM 型

##### 如何防范
1. 验证输入
2. 编码输出（用来确保输入的字符被视为数据，而不是作为html被浏览器所解析）

#### 8. 不安全的反序列化
不安全的反序列化会导致远程代码执行。即使反序列化缺陷不会导致远程代码执行，攻击者也可以利用它们来执行攻击，包括：重播攻击、注入攻击和特权升级攻击。

#### 9. 使用含有已知漏洞的组件
组件（例如：库、框架和其他软件模块）拥有和应用程序相同的权限。如果应用程序中含有已知漏洞的组件被攻击者利用，可能会造成严重的数据丢失或服务器接管。

同时，使用含有已知漏洞的组件的应用程序和API可能会破坏应用程序防御、造成各种攻击并产生严重影响。

##### 如何防范
1. 识别正在使用的组件和版本，包括所有的依赖
2. 更新组件或引用的库文件到最新
3. 建立安全策略来管理组件的使用

#### 10. 不足的日志记录和监控
不足的日志记录和监控，以及事件响应缺失或无效的集成，使攻击者能够进一步攻击系统、保持持续性或转向更多系统，以及篡改、提取或销毁数据。

大多数缺陷研究显示，缺陷被检测出的时间超过200天，且通常通过外部检测方检测，而不是通过内部流程或监控检测。


## 接口类应用安全渗透测试详情

### 1. 常规接口安全风险
接口的安全风险主要有信息泄露、接口非法调用、不安全的对象引用等。如采取访问控制安全策略：例如相关服务器的数据信息只允许经授权的接口调用，其它来源均禁止。

### 2. 业务接口调用
业务接口调用是以内部操作分离出外部调用的方法，使其能被修改而不影响外界其他其交互的方式，常见业务接口方式有重放攻击，短信炸弹，邮件轰炸。
- 重放攻击：在短信或邮件调用业务（例如：短信验证码，邮件验证码），对其业务环节进行调用（重放）测试。如果业务经过调用（重放）后被多次生成有效的业务或数据结果。
- 短信炸弹：在测试的过程中，一般系统平台仅在前端通过JS校验时间来控制短信发送按钮，但后台并未对发送做任何限制，导致可通过重放包的方式大量发送恶意短信。
- 邮件轰炸: 同短信炸弹，只不过媒介是邮件

### 3. Web认证业务接口调用
测试内容包括但不限于：
- 会话重放，信息轰炸
- 更换不同账号，反复请求，资源消耗
- 非法调用/未验证会话

### 4. API接口处理响应测试
包括利用 `Peach` 使用 `Fuzzing` 的方式进行接口处理响应测试。

### 5. 接口数据验证测试
检查外部接口传输给应用程序的数据是否经过了校验，校验规则是否完善，对非法字符是否进行了阻断。

### 6. 不安全的生态接口
包括生态系统中不安全的Web、后端API、云或移动接口，导致设备或相关组件遭攻陷。

常见的问题包括缺乏认证或授权、缺乏加密或弱加密以及缺乏输入和输出过滤。

### 7. 接口访问控制机制
应当考虑访问控制机制的可用性与成熟度及实现的难易程度。

如通过接口对用户访问的系统功能进行限制，如：对不同的用户显示不同的功能菜单。

### 8. 接口返回敏感数据
此问题是由于接口控制不严导致，解决方法一般是严格控制查询接口和响应数据，不需要返回的数据不返回给客户端。

## 前端应用安全渗透测试详情
通过采用适当测试手段，发现测试目标在服务系统认证及授权、代码审查、被信任系统的测试、文件接口模块报警响应等方面存在的安全漏洞,并现场演示再现利用该漏洞可能造成的客户资金损失，并提供避免或防范此类威胁、风险或漏洞的具体改进或加固措施。

应用程序及代码在开发过程中，由于开发者缺乏安全意识，疏忽大意极为容易导致应用系统存在可利用的安全漏洞。一般包括:
- SQL 注入
- 跨站脚本
- 表单绕过
- 上传漏洞
- 文件包含
- 已知木马
- 敏感信息泄露
- 恶意代码
- 解析漏洞
- 远程代码执行漏洞
- 任意文件读取
- 目录遍历
- 目录列出
- 跨站请求伪造
- 弱口令
- 不安全对象引用
- 安全配置错误
- 链接地址重定向
- 跳转漏洞
- 后台管理
- 会话管理
- 无效验证码

### 1. SQL注入
SQL注入漏洞的产生原因是网站程序在编写时，没有对用户输入数据的合法性进行判断，导致应用程序存在安全隐患。

SQL注入漏洞攻击就是利用现有应用程序没有对用户输入数据的合法性进行判断，将恶意的SQL命令注入到后台数据库引擎执行的黑客攻击手段。

### 2. 跨站脚本
跨站脚本攻击简称为 XSS 又叫CSS (Cross Site Script Execution)，是指服务器端的CGI程序没有对用户提交的变量中的HTML代码进行有效的过滤或转换，允许攻击者往WEB页面里插入对终端用户造成影响或损失的HTML代码。

### 3. 表单绕过
表单绕过是指在登录表单时可以利用一些特殊字符绕过对合法用户的认证体系，这造成对用户输入的字符没有进行安全性检测，攻击者可利用该漏洞进行SQL注入攻击。

### 4. 上传漏洞
上传漏洞是指网站开发者在开发时在上传页面中针对文件格式（如asp、php等）和文件路径过滤不严格，导致攻击者可以在网站上上传木马，非法获取WebShell权限。

### 5. 文件包含
目标网站允许用户调用网站程序函数进行文件包含，同时未对所包含文件的类型及内容进行严格过滤。

### 6. 已知木马
目标网站被攻击者植入恶意木马，已知木马包括攻击者在进行网站入侵时留下的后门程序和网页挂马两种：
- **后门程序** 严重危害网站安全，攻击者可利用该后门直接获取整个网站的控制权限，可对网站进行任意操作，甚至以网站为跳板，获取整个内网服务器的控制权限
- **网页挂马** 严重危害网站用户安全，用户对已被挂马的网页进行浏览和访问，其PC机自动下载并执行木马程序，导致用户PC机被攻击，同时严重危害网站的信誉和形象。

### 7. 敏感信息泄露
敏感信息泄漏漏洞指泄漏有关WEB应用系统的信息，例如，用户名、物理路径、目录列表和软件版本。

尽管泄漏的这些信息可能不重要，然而当这些信息联系到其他漏洞或错误设置时，可能产生严重的后果。例如：某源代码泄漏了SQL服务器系统管理员账号和密码，且SQL服务器端口能被攻击者访问，则密码可被攻击者用来登录SQL服务器，从而访问数据或运行系统命令。

以下几个是比较典型的敏感信息泄露漏洞：
- 源码信息泄露
- 备份信息泄露
- 错误信息泄露
- 测试账户泄露
- 测试文件泄露
- 绝对路径泄露

### 8. 恶意代码
恶意代码泛指没有作用却带来危险的代码，其普遍的特征是具有恶意的目的；本身是一个独立的程序，通过执行发生作用。由于应用系统存在可被利用的安全漏洞，可能已被恶意人员植入恶意代码以获取相应权限或用以传播病毒。

### 9. 解析漏洞
解析漏洞是指没有对解析的内容进行严格的定义，被攻击者利用，可能会使系统对带有木马的文件进行了解析并执行，导致敏感信息被窃取、篡改，甚至是系统奔溃。

### 10. 远程代码执行漏洞
远程执行任意代码漏洞是指由于配置失误（有时我们在用户认证只显示给用户认证过的页面和菜单选项，而实际上这些仅仅是表示层的访问控制而不能真正生效），攻击者能够很容易的就伪造请求直接访问未被授权的页面。

### 11. 任意文件读取
系统开发过程中没有重视安全问题或使用不安全的第三方组件等，导致任意文件可读取，可导致入侵者获得数据库权限，并利用数据库提权进一步获得系统权限。

### 12. 目录遍历
目录遍历是指由于程序中没有过滤用户输入的`../` 和 `./` 之类的目录跳转符,导致攻击者通过提交目录跳转来遍历服务器上的任意文件。

### 13. 目录列出
目录列出是指攻击者通过对访问的URL分析，得到一个敏感的一级或二级目录名称，然后访问该目录，返回结果会将会显示指定目录及其子目录下的所有文件，从而可以寻找并获取敏感信息（如备份文件存放地址、数据库连接文件源码、系统敏感文件内容等），甚至可以通过在地址栏中修改URL来挖掘这个目录结构。

### 14. 跨站请求伪造
跨站请求伪造（Cross-site request forgery，缩写为CSRF），也被称成为 `one click attack` 或者 `session riding`， 通常缩写为`CSRF`或者`XSRF`，是一种对网站的恶意利用。

尽管听起来像跨站脚本（XSS），但它与XSS非常不同，并且攻击方式几乎相左。XSS利用站点内的信任用户，而CSRF则通过伪装来自受信任用户的请求来利用受信任的网站。

与XSS攻击相比，CSRF攻击往往不大流行（因此对其进行防范的资源也相当稀少）和难以防范，所以被认为比XSS更具危险性。

### 15. 弱口令
弱口令通常有以下几种情况：用户名和密码是系统默认、空口令、口令长度过短、口令选择与本身特征相关等。

系统、应用程序、数据库存在弱口令可以导致入侵者直接得到系统权限、修改盗取数据库中敏感数据、任意篡改页面等。

### 16. 不安全对象引用
不安全的对象引用是指程序在调用对象的时候未对该对象的有效性、安全性进行必要的校验。

如：某些下载程序会以文件名作为下载程序的参数传递，而在传递后程序未对该参数的有效性和安全性进行检验，而直接按传递的文件名来下载文件，这就可能造成恶意用户通过构造敏感文件名而达成下载服务端敏感文件的目的。

### 17. 安全配置错误
某些HTTP应用程序，或第三方插件，在使用过程中由于管理人员或开发人员的疏忽，可能未对这些程序或插件进行必要的安全配置和修改，这就很容易导致敏感信息的泄露。

而对于某些第三方插件来说，如果存在安全隐患，更有可能对服务器获得部分控制权限。

### 18. 链接地址重定向
重定向就是通过各种的方法将各种网络请求重新定个方向转到其它位置（如：网页重定向、域名的重定向、路由选择的变化也是对数据报文经由路径的一种重定向）。

而某些程序在重定向的跳转过程中，对重定向的地址未进行必要的有效性和安全性检查，且该重定向地址又很容易被恶意用户控制和修改，这就可能导致在重定向发生时，用户会被定向至恶意用户事先构造好的页面或其他URL，而导致用户信息受损。

之前就有遇到一个关于重定向的安全问题: {% post_link a-target-blank %}

### 19. 跳转漏洞
跳转漏洞是指网站用户访问时对其输入的参数没有进行验证，浏览器直接返回跳转到指定的URL，跳转漏洞可引发XSS漏洞，攻击者可利用这个漏洞进行恶意欺骗。

### 20. 后台管理
后台管理是指由于网站设计者的疏忽、后台管理员的配置不当或失误，导致攻击者可通过某些非法手段直接访问后台数据库页面，从而获取重要敏感信息，上传木马，甚至可获取后台管理员的权限，可进行删除、添加等非法操作，篡改后台数据库数据。

### 21. 会话管理
会话管理主要是针对需授权的登录过程的一种管理方式，以用户密码验证为常见方式，通过对敏感用户登录区域的验证，可有效校验系统授权的安全性，测试包含以下部分：
#### 1. 用户口令易猜解
通过对表单认证、HTTP认证等方式的简单口令尝试，以验证存在用户身份校验的登录入口是否存在易猜解的用户名和密码。
#### 2. 是否存在验证码防护
验证码是有效防止暴力破解的一种安全机制，通过对各登录入口的检查，以确认是否存在该保护机制。

像 google 机器人验证码，就是我最常用的验证码防护方式: {% post_link use-google-recaptcha %}

#### 3. 是否存在易暴露的管理登录地址
某些管理地址虽无外部链接可介入，但由于采用了容易猜解的地址（如：admin）而导致登录入口暴露，从而给外部恶意用户提供了可乘之机。

#### 4. 是否提供了不恰当的验证错误信息
某些验证程序返回错误信息过于友好，如：当用户名与密码均错误的时候，验证程序返回“用户名不存在”等类似的信息，通过对这一信息的判断，并结合 `HTTP Fuzzing` 工具便可轻易枚举系统中存在的用户名，从而为破解提供了机会。

会话管理主要是针对验证通过之后，服务端程序对已建立的、且经过验证的会话的处理方式是否安全，一般会从以下几个角度检测会话管理的安全性：

#### 5. Session是否随机
Session作为验证用户身份信息的一个重要字符串，其随机性是避免外部恶意用户构造Session的一个重要安全保护机制，通过抓包分析Session中随机字符串的长度及其形成规律，可对Session随机性进行验证，以此来确认其安全性。

#### 6. 校验前后Session是否变更
通过身份校验的用户所持有的Session应与其在经过身份验证之前所持有的Session不同。

#### 7. 会话储存是否安全
会话存储是存储于客户端本地（以cookie的形式存储）还是存储于服务端（以Session的形式存储），同时检测其存储内容是否经过必要的加密，以防止敏感信息泄露。

### 22. 无效验证码
目标系统管理入口（或数据库外部连接）存在缺少验证码，攻击者可利用弱口令漏洞，通过进行暴力猜解，获取网站管理权限，包括修改删除网站页面、窃取数据库敏感信息、植入恶意木马；甚至以网站为跳板，获取整个内网服务器控制权限。

## 其他类型中的 业务场景 测试详情
### 1. 注册场景
- 批量注册检测，会话重放攻击、验证码绕过攻击
- 任意手机/邮箱注册
- 短信/邮件轰炸
- 资源竞争，并发注册相同用户名，造成系统业务逻辑状态错误
- 覆盖注册

### 2. 登录场景
- 社工库遍历
- 爆破验证码
- 用户名不变，爆破密码
- 密码不变，爆破用户，
- 固定会话攻击
- 短信/邮件轰炸
- Oauth授权缺陷绕过
- 修改响应包绕过登录验证
- 万能密码登录绕过场景

### 3. 密码重置
- 爆破验证码
- 客户端本地验证绕过
- 验证码直接在response中返回
- 跳过验证步骤，直接访问修改密码页面
- 篡改接受验证码手机、邮箱
- 越权重置他人密码
- 密码修改逻辑放在前台JS脚本，根据逻辑绕过修改密码
- 修改密码步骤SQL注入
- 重置密码弱token破解

之前我有针对密码重置链接的场景做了一个安全方面的总结: [web 安全之 - 关于重置密码链接的几个安全问题](https://kebingzao.com/2020/04/29/reset-pwd-security/)

### 4. 界面锁定
- 修改返回包
- 禁用JS


## 常见的十大业务逻辑漏洞的检查清单
正常情况，在进行安全渗透的时候，有一些场景业务是一定要测试并且检查的:

|业务风险| 风险描述|
|---|---|
| 身份认证安全 | 	在没有验证码限制(意思：没有验证码限制)或者一次验证码可以多次使用的地方，使用已知用户对密码进行暴力破解或者用一个通用密码对用户进行暴力破解。 |
| 业务一致性安全|	手机号篡改描述: 抓包修改手机号码参数为其他号码尝试，例如在办理查询页面，输入自己的号码然后抓包，修改手机号码参数为其他人号码，查看是否能查询其他人的业务。
| 业务数据篡改| 	1. 金额数据篡改描述: 抓包修改金额等字段，例如在支付页面抓取请求中商品的金额字段，修改成任意数额的金额并提交，查看能否以修改后的金额数据完成业务流程。 <br> 2. 商品数量篡改描述: 抓包修改商品数量等字段，将请求中的商品数量修改成任意数额，如负数并提交，查看能否以修改后的数量完成业务流程。<br> 3. 最大数限制突破描述: 很多商品限制用户购买数量时，服务器仅在页面通过js脚本限制，未在服务器端校验用户提交的数量，通过抓包修改商品最大数限制，将请求中的商品数量改为大于最大数限制的值，查看能否以修改后的数量完成业务流程。
| 用户输入合规性	| 用户输入合规性描述：用户输入合规性功能测试，一般用户登陆处用的多一些，未对用户输入的参数未做html转义过滤（要过滤的字符包括：单引号、双引号、大于号、小于号，&符号等），防止脚本执行。在变量输出时进行HTML ENCODE 处理。
| 密码找回漏洞| 	密码找回逻辑描述：互联网中，有用户注册的地方，基本就会有密码找回的功能。密码找回功能里可能存在此逻辑漏洞，而这些漏洞往往可能产生非常大的危害，如用户账号被盗等。
|验证码突破|	验证码描述：验证码一般是防止批量注册、刷票、论坛灌水，有效防止某个黑客对某一个特定注册用户用特定程序暴力破解方式进行不断的登陆尝试验证码。
|业务授权安全|	1. 未授权访问描述：<br>非授权访问是指用户在没有通过认证授权的情况下能够直接访问需要通过认证才能访问到的页面或文本信息。可以尝试在登录某网站前台或后台之后，将相关的页面链接复制于其他浏览器或其他电脑上进行访问，看是否能访问成功。<br>2. 越权访问描述：<br> 越权漏洞的成因主要是因为开发人员在对数据进行增、删、改、查询时对终端请求的数据过分相信而遗漏了权限的判定. <br> a) 垂直越权（垂直越权是指使用权限低的用户可以访问权限较高的用户）<br> b) 水平越权（水平越权是指相同权限的不同用户可以互相访问）
|业务流程乱序|	业务流程乱序描述：一般一个程序上，针对不同的业务需求，业主会设定相应的业务流程序，快速提高相对应的工作效率。业务流程顺序一般先A过程后B过程然后C过程最后D过程，比如出现的逻辑乱序是先A过程后直接到D过程。
|业务接口调用|	业务接口调用描述：业务接口调用是以内部操作分离出外部调用的方法，使其能被修改而不影响外界其他其交互的方式，常见业务接口方式有重放攻击，短信炸弹。<br> 1. 重放攻击：在短信或邮件调用业务（例如：短信验证码，邮件验证码），对其业务环节进行调用（重放）测试。如果业务经过调用（重放）后被多次生成有效的业务或数据结果。<br>2. 短信炸弹：在测试的过程中，一般系统平台仅在前端通过JS校验时间来控制短信发送按钮，但后台并未对发送做任何限制，导致可通过重放包的方式大量发送恶意短信。
|时效性绕过|	时效性绕过描述：针对某些带有时间限制的业务，修改其时间限制范围。

## web 安全渗透 检查清单
接下来就是具体的渗透测试的 检查清单了，一般分为两种
1. 针对安全缺陷的类型来逐一测试, 以保证常见的安全缺陷都可以被覆盖到
2. 针对具体业务场景来逐一测试，以保证常见的用户场景都可以被覆盖到

### 1. 针对安全缺陷漏洞的检查清单

|序号|	检查项
|---|---|
| 1 |	SQL注入攻击
| 2 |	反射式跨站脚本攻击
| 3 |	存储式跨站脚本攻击
| 4 |	DOM跨站脚本攻击
| 5 |	会话固定
| 6 |	HTTP头安全配置
| 7 |	HTTP方法
| 8 |	服务器版本信息
| 9 |	会话生命周期
| 10 |	水平权限溢出
| 11 |	纵向权限溢出
| 12 |	默认凭证
| 13 |	弱密码规则测试
| 14 |	账户锁定测试
| 15 |	安全问题测试
| 16 |	密码修改测试
| 17 |	目录遍历
| 18 |	文件包含
| 19 |	授权绕过
| 20 |	权限溢出
| 21 |	不安全对象直接引用
| 22 |	Cookie属性
| 23 |	会话管理
| 24 |	登出测试
| 25 |	会话超时测试
| 26 |	账户恶意锁定
| 27 |	用户账户唯一性测试
| 28 |	Session随机性与唯一性测试
| 29 |	CRLF注入测试
| 30 |	LDAP注入测试
| 31 |	XML注入测试
| 32 |	ORM注入测试
| 33 |	Xpath注入测试
| 34 |	本地文件包含
| 35 |	远程文件包含
| 36 |	命令执行
| 37 |	缓冲区溢出
| 38 |	堆溢出
| 39 |	栈溢出
| 40 |	格式化字符串测试
| 41 |	HTTP Split测试
| 42 |	代码异常处理
| 43 |	栈异常跟踪
| 44 |	SSL/TLS加密算法测试
| 45 |	PaddingOracle测试
| 46 |	敏感信息泄露
| 47 |	上传文件测试
| 48 |	事务完整性测试
| 49 |	HTML注入
| 50 |	URL重定向攻击
| 51 |	CSS注入攻击
| 52 |	Blind注入测试
| 53 |	点击劫持测试
| 54 |	WebSocket测试
| 55 |	本地存储测试
| 56 |	Web Messaging测试
| 57 |	IMAP/SMTP注入测试
| 58 |	代码注入测试
| 59 |	处理响应时间测试
| 60 |	可用性测试
| 61 |	CORS测试
| 62 |	FlashSWF测试
| 63 |	恶意文件上传测试
| 64 |	暴力破解
| 65 |	传输层常见安全测试
| 66 |	错误代码分析
| 67 |	失效验证测试
| 68 |	使用已知不安全组件测试
| 69 |	未验证重定向与转发
| 70 |	不安全对象失效间接引用
| 71 |	默认配置安全测试
| 72 |	用户隐私信息检查
| 73 |	CSRF客户端请求伪造测试
| 74 |	HTML5安全测试
| 75 |	邮件头注入测试
| 76 |	常见框架已知脆弱性检查
| 77 |	SSRF服务端请求伪造测试
| 78 |	XXE


### 2. 针对业务场景的检查清单

|序号|分类|检查项|
|---|---|---|
| 1 | 登录场景 |	社工库遍历
| 2 | |		爆破验证码
| 3 | |		用户名不变，爆破密码
| 4 | |		密码不变，爆破用户名
| 5 | |		固定会话攻击
| 6 | |		短信/邮件轰炸
| 7 | |		Oauth授权缺陷绕过
| 8 | |		万能密码登录绕过
| 9 | |		修改响应包绕过登录验证
| 10 | |		验证码安全测试（验证码方案、识别率、验证码绕过等）
| 11 | |		安全协议测试
| 12 | |		http安全配置检查
| 13 | |		安全登录验证策略检查
| 14 | |		弱口令、空口令检测
| 15 | |		用户名口令容错性测试
| 16 | |		登录信息有效性测试，边界测试
| 17 | |		SQL注入，XSS等
| 18 | |		多种查询语句的组合测试
| 19 | 密码找回场景<br>（根据实际业务） |		短信/邮件/微信验证绕过
| 20 | |		密码暴露破解
| 21 | |		信息劫持漏洞
| 22 | |		绕过时间控制
| 23 | 密码重置场景<br>（根据实际业务）|	爆破验证码
| 24 | |		客户端本地验证绕过
| 25 | |		验证码直接在response中返回
| 26 | |		跳过验证步骤，直接访问修改密码页面
| 27 | |		篡改接收验证码手机、邮箱
| 28 | |		越权重置他人密码
| 29 | |		密码修改逻辑放在前台JS脚本，根据逻辑绕过修改密码
| 30 | |		修改密码步骤SQL注入
| 31 | |		重置密码弱token破解
| 32 | 界面锁定<br>（根据实际业务）|		修改返回包
| 33 | |		禁用JS
| 34 | 注册场景<br>（根据实际业务）|		批量注册（会话重放/验证码绕过）
| 35 | |		任意手机/邮箱注册
| 36 | |		短信/邮件轰炸攻击
| 37 | |		资源竞争 ，并发注册相同用户名，造成业务系统逻辑错误
| 38 | |		覆盖注册
| 39 | |		信息劫持（合法输入、验证码，激活邮件）
| 40 | web认证业务接口调用|		会话重放，信息轰炸
| 41 | |		更换不同账号，反复请求，资源消耗
| 42 | |		非法调用/未验证会话
| 43 | 上传/下载场景|		上传/下载文件格式限制饶过
| 44 | |		上传/下载文件大小限制饶过
| 45 | |		饶过拓展名
| 46 | |		上传/下载空间位置限制
| 47 | |		特殊字符饶过
| 48 | |		并发处理逻辑验证
| 49 | |		上传/下载流程绕过
| 50 | |		乱序流程测试
| 51 | 权限控制|		基于URL的横向越权、纵向越权
| 52 | |		权限最小化的安全设计
| 53 | |		普通用户可以获取admin权限
| 54 | |		普通用户可以查看其它用户身份证、密码、房产等敏感信息
| 55 | |		页面的权限控制，如必须经过A-B-C的页面是否直接可以A-C绕过
| 56 | |		后台资源暴露
| 57 | |		删除修改正在登陆系统人员权限。 程序是否正常处理
| 58 | |		系统资源同步问题
| 59 | 敏感数据|		上传敏感数据（身份证、房产、密码等）使用安全协议，保证数据的机密性和完整性
| 60 | |		数据在本地存储采用安全加密算法
| 61 | |		日志、页面等处是否有暴露出的敏感信息
| 62 | 其他|		最新漏洞测试
| 63 | |		配置检查类，组件版本类问题
| 64 | |		不必要的开放端口，造成的权限绕过

## 漏洞定级
具体看: {% post_link cvss %}

---

参考资料:
- [github渗透测试工具库](https://www.freebuf.com/sectool/246434.html)
- [小白日志: kali 渗透测试](https://blog.csdn.net/zixuanfy/category_6404684.html)
- [小白必看!OWASP top 10详解](https://zhuanlan.zhihu.com/p/393635352)


