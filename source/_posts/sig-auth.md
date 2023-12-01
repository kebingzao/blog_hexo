---
title: 使用签名算法来作为微服务内部调用或者开放 API 的请求权限验证
date: 2023-11-30 11:50:30
tags: 
- security
- aws
- auth
- golang
categories: web安全
---
## 前言
随着内部应用的微服务的增多，一定会涉及到频繁的微服务之间的互相调用，这时候要有一种验证方式来保证两个微服务的调用是合法的(这边只是网关级别的请求校验，该有的业务校验一样要有), 如果你的微服务相互调用是通过 http(s) 的方式，那么采用签名算法来做请求的安全性校验是业务比较常用的方式。
> 当然还有其他的方式，比如 ip 白名单，但是对于有负载均衡，横向扩展的服务来说，就没那么灵活

签名算法大家其实不陌生，很多第三方的 sdk，权限校验走的就是签名算法的校验方式，原理都大同小异，其中我接触最多的就是 aws 的签名算法: [AWS Signature Version 4](https://docs.aws.amazon.com/zh_cn/AmazonS3/latest/API/sigv4-auth-using-authorization-header.html#sigv4-auth-header-overview)

## 最基础的原理
最核心的原理就这几步，对于客户端来说:
1. 生成一对安全凭证，包括一个 `AccessKey` 和 `SecretKey`, 前者可以参与加密也可以不参与，主要是用来匹配对应的 `SecretKey`， 后者就是用来进行加密生成签名的
2. 将整个 http 请求对象进行聚合操作，生成一个待签名串 `StringToSign`
3. 用 `SecretKey` 加密待签名串 `StringToSign`，生成最后的签名 `Signature`
4. 将签名 `Signature` 和 `AccessKey` 放在 header 标头，连同整个 http 请求对象发送过去

<!--more-->
对于验证的后端来说就是:
1. 接收到请求之后，获取标头的签名 `Signature` 和 `AccessKey`
2. 通过 `AccessKey` 匹配找到对应的 `SecretKey` (服务端也要存这一对安全凭证，因为 `SecretKey` 不会随着传输的)
3. 服务端相同的算法将整个 http 请求对象生成待签名串 `StringToSign`
4. 然后通过 `SecretKey` 加密待签名串 `StringToSign`，生成服务端的签名 `Signature`
5. 对比这两个签名，如果一致，就说明签名算法通过

基础流程就是这样，然后这个过程可以辅助各种其他安全校验，比如:
1. 增加时间戳校验，将生成的时间戳也加进去校验，只有规定时间内的请求，才算合法
2. 加入域名白名单，校验 header host 标头，只有在白名单的才允许通过
3. 加入 ip 白名单，校验来源 ip，只有在 ip 白名单的才允许通过
4. 加入各种 ACL 权限的校验，对应的 `AccessKey` 也要有对应的 ACL 权限，才能通过 (要借助数据库的查询)

## AWS Signature Version 4
接下来我们来分析一下 `AWS Signature Version 4` (以下简称 `AWS4`) 的整个签名过程，来更好的理解整个签名过程，AWS 的这个版本的签名，十几年过去了，还是依然这么坚挺，说明不管是复杂程度，还是安全程度，都非常的靠谱。

``AWS4`` 的整个签名的信息，是放在 HTTP Authorization 标头的，这也是业内最常用的方式，比如:
```text
Authorization: AWS4-HMAC-SHA256 Credential=AKIAIOSFODNN7EXAMPLE/20130524/us-east-1/s3/aws4_request, SignedHeaders=host;range;x-amz-date, Signature=fe5f80f77d5fa3beca038a248ff027d0445342fe2855ddc963176630326f1024
```
它是按照空格隔开的，整个标头可以分成 4 部分信息:
```text
AWS4-HMAC-SHA256
Credential=AKIAIOSFODNN7EXAMPLE/20130524/us-east-1/s3/aws4_request, 
SignedHeaders=host;range;x-amz-date, 
Signature=fe5f80f77d5fa3beca038a248ff027d0445342fe2855ddc963176630326f1024
```
具体是

|组成|描述|
|---|---|
| `AWS4-HMAC-SHA256` |  这个是一个固定值，用于计算签名的算法。使用 `AWS4` 进行身份验证时，必须提供此值。指定 AWS 签名版本 4 (`AWS4`) 和签名算法 (HMAC-SHA256) |
| `Credential` | 访问密钥 `AccessKey` 和范围信息，包括用于计算签名的日期、区域和服务, 比如 <br> `<AccessKey>/<date>/<aws-region>/<aws-service>/aws4_request` |
| `SignedHeaders` | 用于计算 `Signature` 的请求标头的分号分隔列表。该列表仅包含标头名称，并且标头名称必须为小写。例如：`host;range;x-amz-date` <br> 不需要所有的标头都要参与签名计算，比较有意义才需要 |
| `Signature` | 256 位签名以 64 个小写十六进制字符表示 |

前面三个组成都很好理解，基本上都是明文，不需要计算，直接根据情况给就行了，主要是最后的 `Signature` 怎么算才是核心

### 1. 计算签名
要计算签名，首先需要一个待签字符串。然后，使用签名密钥计算待签字符串的 HMAC-SHA256 哈希值。贴一个官方的图

![1](1.png)

#### 1.1. 生成待签名串 StringToSign
生成待签名串 StringToSign， 待签名串的生成由两个步骤

##### 1.1.1 生成规范请求字串 CanonicalRequest
将请求的内容（主机、操作、标头等）组织为标准规范格式。规范请求是用于创建待签字符串的输入之一，比如:
```text
<HTTPMethod>\n
<CanonicalURI>\n
<CanonicalQueryString>\n
<CanonicalHeaders>\n
<SignedHeaders>\n
<HashedPayload>
```
这部分的资讯就是你的 http 请求体的各部分组成，包含

|请求体内容|描述|
|---|---|
| `HTTPMethod` | HTTP 方法，例如 GET、PUT、HEAD 和 DELETE |
| `CanonicalUri` | 绝对路径组件 URI 的 URI 编码版本，以域名后面的“/”开头，直至字符串结尾处，或者如果包含查询字符串参数，则直至问号字符（“?”）。如果绝对路径为空，则使用正斜杠字符（/）。<br> 比如 `/examplebucket/myphoto.jpg` |
| `CanonicalQueryString` |  URI 编码的查询字符串参数。可以单独对每个名称和值进行 URI 编码。必须按键名称的字母顺序对规范查询字符串中的参数进行排序。编码后进行排序 |
| `CanonicalHeaders` | 请求标头及其值的列表。各个标头名称和值对用换行符（“\n”）分隔,为了防止数据篡改，最好在签名计算中包含所有标头 |
| `SignedHeaders` | 按字母顺序排序、以分号分隔的小写请求标头名称列表。列表中的请求标头与在 CanonicalHeaders 字符串中包含的标头相同。<br> 比如 `host;x-amz-content-sha256;x-amz-date` |
| `HashedPayload` | 使用 HTTP 请求正文中的 `payload` 作为哈希函数的输入创建的字符串, 算法是 `Hex(SHA256Hash(<payload>)`, 如果请求中不包含有效 `payload`, 那也要按照空字符串去算 `Hex(SHA256Hash(""))`

**细节补充一**: 针对 `CanonicalQueryString` 中的参数，不管是 key 还是 value 都要在排序之后，进行 `UriEncode` 编码，比如以下这个:
```text
http://s3.amazonaws.com/examplebucket?prefix=somePrefix&marker=someMarker&max-keys=2
```
如果要组合的话，那么这一行的值就会变成 (应该只有一行的内容，只是为了更直观理解，用了换行):
```text
UriEncode("marker")+"="+UriEncode("someMarker")+"&"+
UriEncode("max-keys")+"="+UriEncode("20") + "&" +
UriEncode("prefix")+"="+UriEncode("somePrefix")
```
如果只有 key 有值，value 为空，那么就是 `UriEncode(key)=""`, 比如:
```text
http://s3.amazonaws.com/examplebucket?acl&marker=someMarker
```
那么就是:
```text
UriEncode("acl")+"="+""+"&"+
UriEncode("marker")+"="+UriEncode("someMarker")
```
如果 `CanonicalQueryString` 为空，也就是 path 之后没有带查询参数，那么这一行依然存在，只不过是空。不能省略

**细节补充二**: 针对 `CanonicalHeaders` ，示例如下:
```text
Lowercase(<HeaderName1>)+":"+Trim(<value>)+"\n"
Lowercase(<HeaderName2>)+":"+Trim(<value>)+"\n"
...
Lowercase(<HeaderNameN>)+":"+Trim(<value>)+"\n"
```
CanonicalHeaders 列表必须包含以下内容：
- HTTP `host` 标头
- 如果请求中存在 Content-Type 标头，则必须将其添加到 CanonicalHeaders 列表中
- 此外，还必须添加计划在请求中包含的所有 `x-amz-*` 标头。例如，如果您使用临时安全凭证，则请求中必须包含 `x-amz-security-token`。您必须将此标头添加到 CanonicalHeaders 列表中

每个标头名称必须：
- 使用小写字符
- 按字母顺序显示
- 后跟冒号（:）

对于值必须：
- 去除任何前导空格或尾随空格
- 将连续空格转换为单个空格
- 使用逗号分隔多值标头的值
- 签名中必须包含 `host` 标头（HTTP/1.1）或 `:authority` 标头（HTTP/2）以及任何 `x-amz-*` 标头。签名中也可以包含其他标准标头，例如 `content-type`

举个例子:
```text
host:s3.amazonaws.com
x-amz-content-sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
x-amz-date:20130708T220855Z
```

**细节补充三**: 针对 `HashedPayload` 这边的 payload 指的是 body 体，不包含 url 后面 `?` 的查询字符串(Query String)

请求负载（Request Payload）是请求主体（Body）的内容，它是在 HTTP 请求中发送给服务器的数据。这些数据通常包含在 POST、PUT 或 PATCH 请求中，而不是 GET 请求中，这些数据可以包含各种格式，比如:
- 表单数据（Form data）
- JSON（JavaScript Object Notation）
- XML（eXtensible Markup Language）
- 二进制数据：例如上传的文件

所以在 `AWS4` 的签名中，如果是 `GET` 请求的话，那么就是没有 payload，计算空字符串的哈希值。 如果是使用 `PUT` 请求上传对象时，就必须提供 payload



##### 1.1.2 生成待签名字符串 StringToSign
通过第一步我们规范了 http 请求，得到了一个包含多行的规范请求的字符串 CanonicalRequest， 接下来再基于这个字符串再进行拼接，得到最后的待签名字符串 StringToSign, 具体拼接过程如下:
```text
Algorithm \n
RequestDateTime \n
CredentialScope  \n
HashedCanonicalRequest
```

|内容|描述|
|---|---|
| `Algorithm` | 用于创建规范请求的哈希的算法。`AWS4` 固定是 `AWS4-HMAC-SHA256` |
| `RequestDateTime` | 在凭证范围内使用的日期和时间。该值是采用 ISO 8601 格式的当前 UTC 时间（例如 20130524T000000Z）|
| `CredentialScope` | 凭证范围。这会将生成的签名限制在指定的区域和服务范围内。该字符串采用以下格式：`YYYYMMDD/region/service/aws4_request` |
| `HashedCanonicalRequest` | 上述规范请求的哈希, 其实是就是 `Hex(SHA256Hash(<CanonicalRequest>))`

这边的 `CredentialScope` 内容，应该是跟 `Authorization` 标头中的第二部分 `Credential` 中的 `AccessKey` 后面的部分应该一致:
```text
Authorization Credential = <AccessKey>/<date>/<aws-region>/<aws-service>/aws4_request
CredentialScope = <date>/<aws-region>/<aws-service>/aws4_request
```

最后拼起来的待签名字串 StringToSign 就是:
```text
"AWS4-HMAC-SHA256" + "\n" +
timeStampISO8601Format + "\n" +
<Scope> + "\n" +
Hex(SHA256Hash(<CanonicalRequest>))
```

#### 1.2. 生成待签名密钥 SigningKey
本质上待签名密钥 SigningKey 就是通过这一组安全凭证中的 `SecretKey` 来转变生成的，以 `AWS4` 为例，要生成 SigningKey 要经过以下步骤, 每一步都会在上一步的结果中进行再计算:
```text
DateKey = HMAC-SHA256("AWS4"+"<SecretKey>", "<YYYYMMDD>")
DateRegionKey = HMAC-SHA256(<DateKey>, "<aws-region>")
DateRegionServiceKey = HMAC-SHA256(<DateRegionKey>, "<aws-service>")
SigningKey = HMAC-SHA256(<DateRegionServiceKey>, "aws4_request")
```
可以看到，核心还是以 `SecretKey` 来加密，然后再配合上述说的 `<CredentialScope>` 中的 `<date>/<aws-region>/<aws-service>/aws4_request`, 将其拆开变成 4 步，最后就可以得到待签名密钥 SigningKey

#### 1.3. 根据 SigningKey 和 StringToSign 进行签名计算
这个就很简单了
```text
Signature = HMAC-SHA256(SigningKey, StringToSign)
```
然后将签名从二进制转换为十六进制表示形式，使用小写字符， 如果是前端代码的话，可以这样子写:
```text
    let hash = CryptoJS.HmacSHA256(stringToSign, SigningKey)
    let signature = CryptoJS.enc.Hex.stringify(hash)
```

### 2. 将签名添加至请求
从上一步操作中，我们已经得到了签名 Signature 了, 接下来只要将其作为一部分添加到 Authorization 标头即可，四个部分的内容用空格隔开，开头说过，不再赘述。

不过要注意一个细节，算法和 Credential 之间没有逗号。其他三个元素必须使用逗号分隔其他元素，也就是四个元素之间要有一个空格先隔开，然后对于第二个和第三个元素，内容结尾还要再加一个逗号来分隔


### 3. demo 实例
AWS 有非常详尽的 sdk 和 demo 实例:
- [sigv4a signing examples for node js](https://github.com/aws-samples/sigv4a-signing-examples/tree/main/node-js)
- [请求签名示例](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/signature-v4-examples.html#signature-v4-examples-sdk)
- [javascript sdk v4.js](https://github.com/aws/aws-sdk-js/blob/master/lib/signers/v4.js)
- [golang sdk v4.go](https://github.com/aws/aws-sdk-go/blob/main/aws/signer/v4/v4.go)

## 简易的签名算法 SIG-AUTH
在理解 `AWS4` 的基础上，我们如果自己项目内部要做签名算法的话，就比较容易了，基础原理都一样，但是细节上会更简单很多，因为少了很多 ACL Scope 的校验，具体规则如下:

### 1. 分配一组安全凭证
开始一样会分配一组安全凭证，包含一个 `AccessKey` 和 `SecretKey`, 而且 `SecretKey` 一定要足够复杂，因为作为开放 sdk 的话，签名的具体算法是公开的，因此作为加密的 key `SecretKey` 组成部分要足够复杂才能有效防止字典暴力破解

### 2. 将签名信息放在 Authorization 标头
一样将签名的相关信息放在 Authorization 标头，格式为:
```text
Authorization: SIG-AUTH Key={AccessKey}, Sign={Signature}, Timestamp={timestamp}, Version=1
```
花括号内是可变的参数值。除开头的 scheme 部分外，其余各参数由逗号隔开，顺序不做要求，参数名称前的空白字符会被忽略。各参数定义为:
- Authorization scheme 固定为 SIG-AUTH 
- Key 是请求者的 `AccessKey `
- Sign 是基于请求内容和 `SecretKey` 生成的签名 Signature
- Timestamp 是生成签名时的 UNIX 时间戳，单位是秒。
- Version 表示签名算法的版本，当前固定值为 1 。可省略，省略时默认为 1 

后端服务器将根据签名算法，校验 Sign 的值是否正确，并要求 Timestamp 在允许的误差范围内（默认为 300 秒）

### 3. 对请求体做规范化，生成待签名串 StringToSign
字符集统一使用 UTF-8 。签名使用 HMAC-SHA256 算法，通过 `SecretKey` 对待签名串进行哈希计算得到。待签名串根据请求的内容生成，格式为：
> 一样会针对请求体做规范化，生成待签名串，不过这边对请求体做规范化会比 `AWS4` 简单非常多

```text
TIMESTAMP
METHOD
PATH
QUERY_VALUES
BODY_VALUES (optional)
END  (constant)
```
每个部分间用换行符（\n）分割，各部分的值为：
1. TIMESTAMP 是生成签名时的 UNIX 时间戳，需和 Authorization 头里的 Timestamp 参数值一样。
2. METHOD 是 HTTP 请求的 METHOD ，如 GET/POST/PUT 。
3. PATH 请求的路径，没有路径部分时，使用 `/`。 比如请求地址是 `http://temp.org/the/path/` 则路径为 `/the/path/`, 地址是 `http://temp.org/` 或 `http://temp.org`, 路径均为 `/`。
4. QUERY_VALUES 是 URL 的 query string 部分拼接后的值。 先按参数名称的 UTF-8 字节顺序升序，将参数排列好，需使用稳定的排序算法，这样若有同名参数，其顺序不会被打乱； 然后排序后的参数的值紧密拼接起来（无分隔符）； 若一个参数没有值，如“?a=&b=2”或“?a&b=2”中的“a”，则用参数名称代替值拼入。 没有 query string 时，整个 QUERY 部分使用一个空字符串。
5. BODY_VALUES 若是 application/x-www-form-urlencoded 请求，则处理方式同 QUERY 。 若是 application/json 请求，则为 JSON 原文，和 BODY 上送的一致，不做任何修改。 GET 请求时此部分省略（包含换行符均省略）。 不支持其他类型的请求。
6. 最后一行固定是“END”三个字符，末尾没有空行。

注意，这边有几个不同点:
1. UTF-8 字节顺序不是字典顺序，字节顺序下，英文大写字母在小写字母前面，比如 X 排序在 a 前面
2. 在规范化 QUERY_VALUES 和 application/x-www-form-urlencoded 模式的 BODY_VALUES 的时候，不是走 `urlencode(key)=urlencode(value)`, 而是直接取 value, 只不过当 value 为空的话，用 key 来代替。 然后如果是 application/json 请求，直接将 body 的 json 贴进去即可
3. 如果在 URL 上使用 ~auth 参数，此参数不参与签名计算。

### 4. 支持将 auth 信息放在查询字符串
当不方便定制请求头时，也可以将 Authorization 头的值，放在 URL 的 ~auth 参数上（记得 urlEncode ）。 ~auth 参数不参与签名计算。如果同时提供参数和请求头，则只读取请求头。 

此功能特别适用于 JSONP 请求，因为其不能定制 HTTP 头。

### 5. demo 例子
接下来我们测试几个例子，我有写了一个 SIG-AUTH 的验证的前端交互页面， 具体地址: [go-sigauth](https://github.com/kebingzao/go-sigauth)

#### 5.1 POST form 的例子
待签名的请求为:
```text
POST http://localhost:8012/sigauth/hello?a&c=3&b=2&z=4&X=中文
Content-Type: application/x-www-form-urlencoded
Body: p1=11&p3=33&p2=22
```
请求的 HTTP 报文的内容为:
- 请求的 QUERY 部分为 `a&c=3&b=2&z=4&X=中文`
- 请求的 BODY 部分是 `p1=11&p3=33&p2=22`

获取待签名串的步骤如下:
1. 拼接 TIMESTAMP ，值为 1701415043
2. 拼接 METHOD ，值为 POST
3. 拼接 PATH, 即 /sigauth/hello
4. 计算并追加 QUERY 部分, 得到参数表 `[a, c, b, z, X]`, 将参数根据名称按 UTF-8 字节顺序升序排列，并且使用稳定排序算法。 排列后为 `[X, a, b, c, z]`, 按排序后的参数顺序，得到参数的原始值为：`[中文, , 2, 3, 4]`, 其中有一个空白值为 a，用参数名称代替，最后得到 `中文a234`
5. 计算并追加 BODY 部分。由于是 `application/x-www-form-urlencoded` 的请求， BODY 部分的处理和 QUERY 规则一样，结果为： `112233`
6. 追加最后一行，固定值为 END

因此最后的签名串是:
```text
1701415043
POST
/sigauth/hello
中文a234
112233
END
```

通过 `SecretKey` 计算 HMAC-SHA256 值为： c203adfb66187114179529e959777a110ae3372ed7901f0ffe58ecc63288700f

拼接得到 Authorization 头，追加到请求头，最终请求为：
```text
POST http://localhost:8012/sigauth/hello?a&c=3&b=2&z=4&X=中文
Content-Type: application/x-www-form-urlencoded
Authorization: SIG-AUTH Key=testkey1, Sign=c203adfb66187114179529e959777a110ae3372ed7901f0ffe58ecc63288700f, Timestamp=1701415043, Version=1
Body: p1=11&p3=33&p2=22
```
使用前端校验的截图如下:

![1](2.png)

#### 5.2 POST json 的例子
如果将 Content-Type 改成 application/json，其他都不变，但是 追加 BODY 部分的时候，要直接整个 json 给:
```text
1701415712
POST
/sigauth/hello
中文a234
{"p1":11,"p3":33,"p2":22}
END
```
最好的计算截图

![1](3.png)

#### 5.3 GET 空白请求
由于是 GET 请求，待签名串由5部分构成，没有 BODY 部分；同时此请求没有参数，故 QUERY 部分为空字符串:
```text
1701415843
GET
/sigauth/hello

END
```
最终是:
```text
GET http://localhost:8012/sigauth/hello
Authorization: SIG-AUTH Key=testkey1, Sign=96edf2189c57df77a5e1e0ba8e8a13dc442ce7e310545ae56dab036376ac8f4c, Timestamp=1701415843, Version=1
```

![1](5.png)

#### 5.4 JSONP 请求
也可以走 JSONP 请求，将 auth 放到 query 中的 `~auth`, 具体验证如下
```text
GET http://localhost:8012/sigauth/hello?a&c=3&b=2&z=4&X=中文&callback=_jsonp1701415988865&~auth=SIG-AUTH%20Key%3Dtestkey1%2C%20Sign%3D193d0df954a203fe95181e6f6ea5848fab19f15a86e773cdc0126d6d31ae4fb5%2C%20Timestamp%3D1701415988%2C%20Version%3D1
Content-Type: application/json
Authorization: SIG-AUTH Key=testkey1, Sign=193d0df954a203fe95181e6f6ea5848fab19f15a86e773cdc0126d6d31ae4fb5, Timestamp=1701415988, Version=1
```

这边要注意一个细节， jsonp 后面的 callback 参数也要参与计算:
```text
1701415988
GET
/sigauth/hello
中文a23_jsonp17014159888654
END
```
在我的验证例子里面，jsonp 的 callback 是由当前请求的时间戳生成的，然后附加到 query params 中的:

![1](4.png)

后端验证服务器在处理 jsonp 的时候，返回值是不一样的，这个要记得处理, 以我的 golang demo 为例:
```text
// 返回结果值
func retrunRes(w http.ResponseWriter, r *http.Request, res Res) {
	urlValues := r.URL.Query()
	callback := urlValues.Get("callback")
	// 如果是 jsonp 格式的话，就返回对应格式
	if callback != "" {
		w.Write([]byte(fmt.Sprintf("%s(%s)", callback, res.resJsonString())))
	} else {
		w.Write([]byte(res.resJsonString()))
	}
}
```

## 总结和待优化
目前的这个签名算法，在 POST 上面只支持这两种 content-type
- 一个是表单的 application/x-www-form-urlencoded
- 一个是 json 格式的 application/json

并不支持 multipart/form-data 类型的请求，也就是如果有包含文件上传的话，是不支持，后面其实可以扩展，做的跟 `AWS4` 一样，可以针对二进制 payload 进行签名校验。

但是如果是要作为微服务内部通信的话，那肯定是够的，只要将这一组安全凭证，尤其是 `SecretKey` 弄的复杂一点, 比如位数高一点:
```text
// 基于 base32 编码，其将输入 n 字节，输出 n*8/5 个字符。
// 避免末尾的 padding 需要 n 可以被5整除。
// 若 n 不能被5整除，末尾的 padding （等号）会被自动去掉。
func randomBase32(n int) string {
	src := make([]byte, n)
	n, err := rand.Read(src)
	if err != nil {
		panic(err)
	}

	dst := make([]byte, base32.StdEncoding.EncodedLen(n))
	base32.StdEncoding.Encode(dst, src)

	res := string(dst)
	res = strings.TrimRight(res, "=")
	return res
}

// 生成一对 key 和 secret
func generateAccessKey(w http.ResponseWriter, r *http.Request) {
	res := NewRes()
	res.Data = struct {
		Key    string
		Secret string
	}{randomBase32(15), randomBase32(35)}
	retrunRes(w, r, res)
}
```

如果要作为跟 `AWS4` 一样用来做开放 SDK 的话，可以安全性再加强一点，比如增加域名白名单校验，ip 白名单校验， ACL 权限校验等等, 会更加的安全

---

参考资料:
- [SIG-AUTH demo 和 golang package 地址](https://github.com/kebingzao/go-sigauth)
- [AWS 使用签名版本 4 签名](https://docs.aws.amazon.com/zh_cn/AWSEC2/latest/WindowsGuide/ebsapis-using-sigv4.html)
- [AWS 身份验证方法](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/aws-signing-authentication-methods.html)
- [AWS API 请求签名的元素](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/signing-elements.html)
- [AWS 创建已签名的 AWS API 请求](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/create-signed-request.html)
- [AWS 请求签名示例](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/signature-v4-examples.html#signature-v4-examples-sdk)






