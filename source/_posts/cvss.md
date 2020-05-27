---
title: web 安全之 - 使用 CVSS V3.0 来判断安全漏洞的严重性
date: 2020-05-25 15:48:37
tags: security
categories: web安全
---
## 前言
作为一个站点，肯定少不了那些好心的 researcher 给你的站点或者服务提出各种各样的安全缺陷, 让你去修改，并且适当的给他们一点奖励。 所以如何评价安全漏洞的危害性就显得很重要, 毕竟你也不希望一个小安全问题就给一大笔现金奖励吧，钱多也不是这么花的。 而且除了对安全漏洞评分之外， 还需要根据不同的危害性给予不同的奖励。 比如我们就是以 CVSS 3.0 标准来对安全漏洞进行级别划分，并且根据以下级别进行奖励:
1. 无危害性 --> `0`  --> 没有奖励
2. 低危害性 --> `0.1 - 3.9` --> 我们产品的一年 vip 使用权限
3. 中危害性 --> `4.0 - 6.9` --> $xxx 现金奖励
4. 高危害性 --> `7 - 10` --> $xxxx 现金奖励

接下来接下来讲一下 CVSS 3.0 是一个怎么样的标准:
<!--more-->
## 安全漏洞评分系统（CVSS）
CVSS : Common Vulnerability Scoring System， 即“通用漏洞评分系统”，是一个“行业公开标准，其被设计用来评测漏洞的严重程度，并帮助确定所需反应的紧急度和重要度”。

CVSS的研究是由美国国家基础建设咨询委员会 (National Infrastructure Advisory Council, NIAC)从2003年发起的，並在2005年2月完成第一版CVSS (CVSSv1)，后续则转由 [FIRST](https://www.first.org/cvss/) 进行进一步改善和发展，FIRST也因此建立一个CVSS特别兴趣群 (CVSS Special Interest Group, CVSS SIG)，负责以现行 CVSS 为基准，研发下一版本的 CVSS 应如何进行修正和改善。在采纳各方意见后，2007年6月第二版的CVSS (CVSSv2)正式发布，而随着各界持续提供许多意见，在2015年6月则发布了目前正在使用的第三版 CVSS (CVSSv3.0)。

CVSS 采用了模块化评分体系，包含三个部分：基本维度（Base Metric Group），时间维度（Temporal Metric Group）和环境维度（Environmental Metric Group）。由这三个部分分别计算所得的分数，组成了一个0到10分的漏洞总体得分，10分代表最严重。
1. 基本维度奠定了漏洞得分的基础, 他的核心特性不会随时间而改变，即使添加了不同的目标环境，它们也不会改变。
2. 时间维度会随着时间推移而影响一个漏洞的紧迫程度。
3. 环境维度决定了被发现的漏洞对组织机构和其利益相关者可能产生什么样的影响。这一维度根据漏洞潜在附带损害和目标分布来评估其风险。

顾名思义，基本维度是漏洞得分的基础，时间维度和环境维度在此基础上起到提高或降低分数的作用。 想测试一下的话，可以到[CVSS 3.0 计算器](https://www.first.org/cvss/calculator/3.0) 这个站点尝试一下打分。

因为我们用的是 CVSSv3.0，所以我们接下来简单介绍一下基础维度的一些标准。因为一般评分也都是只会在基础维度进行评分， 除非是要更细的或者更复杂的安全漏洞，才会用到另外两个维度一起评分。

## CVSS V3.0 的基础维度
基础维度有8个方向来进行评分，并得出一个 0.0 - 10.0 的分数， 分数越高就代表漏洞维护程度越高
### 1. 攻击向量 (Attack Vector, AV)
- Network (N)：通过网络进行攻击
- Adjacent (A)： 通过受限制的网络进行攻击，比如局域网或者蓝牙等
- Local (L)：在不连接网络的情况下进行攻击
- Physical (P)：需要接触到实体机器才能攻击

### 2. 攻击复杂度  (Attack Complexity, AC)
- Low (L)：低，攻击可被轻易重现
- High (H)：高，需攻击者达成数项条件才能成功

### 3. 是否需要提权 (Privileges Required, PR)
- None (N)：不需要
- Low (L)：需要一般使用者权限
- High (H)：需要管理者权限

### 4. 是否需要使用者操作 (User Interaction, UI)
- None (N)：不需要
- Required (R)：需要使用者操作某些动作才能让攻击成功

### 5. 影响范围 (Scope, S)
- Unchanged (U) ：僅影響含有漏洞的元件本身 仅影响含有漏洞的元件本身
- Changed (C)：会影响到含有漏洞的元件以外的元件

### 6. 机密性影响 (Confidentiality, C)
- None (N)：无影响
- Low (L)：攻击者可以取到资料，但没法使用该资料
- High (H)：攻击者可以取到资料并使用

### 7. 完整性影响(Integrity, I)
- None (N) ：无影响
- Low (L) ：攻击者有部分权限修改某些资料，但对漏洞本身的元件影响较小
- High (H)：攻击者有权限修改所有资料，并且对漏洞本身的元件有严重影响

### 8. 可用性影响 (Availability, A)
- None (N)：无影响
- Low (L)：可用性受到影响，导致元件或者服务可被部分取得
- High (H)：可用性受到严重影响，导致元件或者服务完全不可被取得

举个例子，以 `WannaCry` 使用的漏洞 [CVE-2017-0144](https://nvd.nist.gov/vuln/detail/CVE-2017-0144) 的 CVSS v3.0 分数是 [8.1](https://www.first.org/cvss/calculator/3.0#CVSS:3.0/AV:N/AC:H/PR:N/UI:N/S:U/C:H/I:H/A:H/E:F/RL:T/RC:R)，其中8个选项如下：
- Attack Vector (AV): Network
- Attack Complexity (AC): High
- Privileges Required (PR): None
- User Interaction (UI): None
- Scope (S): Unchanged
- Confidentiality (C): High
- Integrity (I): High
- Availability (A): High

![png](3.png)

## 关于 CVE
既然有安全漏洞，那么就会有一个存放一些披露出来的安全漏洞的地方，其实就是 CVE。

CVE的英文全称是 Common Vulnerabilities & Exposures（公共漏洞和暴露）。CVE就好像是一个字典表，为广泛认同的信息安全漏洞或者已经暴露出来的弱点给出一个公共的名称。使用一个共同的名字，可以帮助用户在各自独立的各种漏洞数据库中和漏洞评估工具中共享数据，虽然这些工具很难整合在一起。这样就使得CVE成为了安全信息共享的“关键字”。如果在一个漏洞报告中指明的一个漏洞，如果有CVE名称，你就可以快速地在任何其它CVE兼容的数据库中找到相应修补的信息，解决安全问题。

而且获得CVE编号并不等于这个漏洞是有价值的，甚至说这个漏洞都不一定是真实存在的。这主要源于CVE编号颁发机构开放式的工作模式。

CVE开始是由MITRE Corporation负责日常工作的。但是随着漏洞数量的增加，MITRE将漏洞编号的赋予工作转移到了其CNA（CVE Numbering Authorities）成员。CNA涵盖5类成员。目前共有69家成员单位。
1. MITRE：可为所有漏洞赋予CVE编号；
2. 软件或设备厂商：如Apple、Check Point、ZTE为所报告的他们自身的漏洞分配CVE ID。该类成员目前占总数的80%以上。
3. 研究机构：如Airbus，可以为第三方产品漏洞分配编号；
4. 漏洞奖金项目：如HackerOne，为其覆盖的漏洞赋予CVE编号；
5. 国家级或行业级CERT：如ICS-CERT、CERT/CC，与其漏洞协调角色相关的漏洞。

这些机构都可以赋予 cve 编号，所以 cve 本身就没啥价值的，有价值的是这个安全漏洞本身。可以说危害性越大，范围越广的漏洞，那么价值就越大。

## 关于 CVSS 3.0 的评分原则以及一些站点参考
一些 researcher 报的安全漏洞，比如是那种比较常见的安全问题，比如以下两种:
- Missing 'X-Frame-Options' Header
- Missing HTTP Strict Transport Security Policy

他们本身就是因为缺少一些安全的 header 字段，导致会出现一些安全问题，比如 Clickjacking 之类的。 但是这种安全问题在 cve 中是找不到具体的安全漏洞编号的，或者说可以找到好多个相关的。 这个是因为本身这种安全缺陷是要结合具体的环境才能产生安全漏洞问题，而 cve 里面才会有收录了具有这个安全缺陷的安全漏洞。比如这个 `CVE-2017-5782`, 他的描述就是:

{% blockquote nvd https://nvd.nist.gov/vuln/detail/CVE-2017-5782 %}
A missing HSTS Header vulnerability in HPE Matrix Operating Environment version v7.6 was found.
{% endblockquote %}

他要结合到具体的环境中，才能产生安全问题，所以他本身就是一个具有安全缺陷的诱因。

那么怎么对这种安全缺陷进行评分呢，一种就是老老实实的根据你自己的业务情况去评估风险。 但是这种有时候主观性可能就比较大，比如你认为是级别是 low， 但是 research 认为是 medium， 这时候就会有争议。 另一种就是有一些比较专业的关于这种安全缺陷的站点，里面就有关于这个安全缺陷的评分，可以直接采用他们的，这样也比较有公信力。

以 `Missing 'X-Frame-Options' Header` 为例，我可以在这三个站点找到关于这个安全缺陷对应的 CVSS 3.0 的评分
- [acunetix](https://www.acunetix.com/vulnerabilities/web/clickjacking-x-frame-options-header-missing/) -> low
- [netsparker](https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-x-frame-options-header/) -> low
- [tenable](https://www.tenable.com/plugins/was/98060) -> low

![png](1.png)

上面三个都认为是 low， 所以我们评分的时候，危险级就是 low

以 `Missing HTTP Strict Transport Security Policy` 也可以在这两个站点找到对应的 CVSS 3.0 的评分
- [netsparker](https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/http-strict-transport-security-hsts-policy-not-enabled/) -> medium
- [tenable](https://www.tenable.com/plugins/was/98056) -> medium

![png](2.png)

评分的时候，危险级就是 medium。

所以我现在对这些缺陷的评分原则就是，先去这些站点查一下是否已经有存在，如果存在就直接用，如果不存在，再自己根据项目的情况进行评分。


---
参考资料:
- [给漏洞打分 介绍一下CVSS与Tripwire的差异](https://www.aqniu.com/learn/45706.html)
- [聊聊CVE漏洞编号和正式公开那些事](http://blog.nsfocus.net/cve-vulnerability-numbers-officially-disclose/#more-9054) 




