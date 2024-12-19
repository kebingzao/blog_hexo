---
title: 浅谈 - 如何安全加固你的 HTTPS 站点
date: 2024-12-19 14:55:20
tags: 
- nginx
- security
- https
categories: 服务浅谈系列
---
## 前言
当你的站点上了 HTTPS 站点之后，恭喜你，你已经一只脚安全着陆了，但是另一只脚还悬空呢，能否着陆取决你如何安全的配置你的 HTTPS 站点

你可以用 [SSL Server Test](https://www.ssllabs.com/ssltest/analyze.html) 来测试一下你的 HTTPS 站点是否真的安全，达到了 `A+` 的安全级别。

接下来谈一谈如何对你的 HTTPS 站点进行安全加固，有哪些加固的策略呢?

## 1. 连接安全性和加密

### 1.1 启用更强的 TLS 加密 (TLS 1.2 或者 1.3)
传输层安全（TLS）及其前身安全套接字层（SSL），通过在浏览器和 web 服务器之间提供端到端加密来促进机密通信。没有 TLS，就谈不上什么安全。TLS 是 HTTP 安全性的基础。

TLS 1.0 和 TLS 1.1 的加密方式已经可以被破解了， 所以很容易招到攻击。 所以我们要将 TLS 的版本指定为 TLS 1.2 或者 TLS 1.3, 抛弃旧版本，具体可以看之前的实践:
- {% post_link set-tls-12 %}
- {% post_link web-use-tls12 %}
<!--more-->

### 1.2 抛弃弱的密码套件
除了要将 TLS 的 Protocol 版本升级到 1.2 及以上之外， Cipher (加密套件) 也要支持强安全的套件，不安全的加密套件，容易被攻击者破解，从而解密两端加密信息，影响网站安全。

比如下面的加密套件是比较安全的:
1. **ECDHE-ECDSA-AES256-GCM-SHA384**
   - 使用 ECDHE 进行密钥交换，支持前向保密。
   - ECDSA 用于认证，AES 256 位 GCM 用于加密，SHA384 用于消息认证。
2. **ECDHE-RSA-AES256-GCM-SHA384**
   - 类似于上一个套件，但使用 RSA 进行认证。
3. **ECDHE-ECDSA-AES128-GCM-SHA256**
   - 使用 ECDHE 进行密钥交换，支持前向保密。
   - ECDSA 用于认证，AES 128 位 GCM 用于加密，SHA256 用于消息认证。
4. **ECDHE-RSA-AES128-GCM-SHA256**
   - 类似于上一个套件，但使用 RSA 进行认证。

还有一些加密套件是不建议使用的
1. **RC4 加密套件**
   - 由于已知的弱点，RC4 已被弃用。
2. **DES 或 3DES 加密套件**
   - DES 的密钥长度太短，容易被破解。3DES 由于其较低的安全性和效率，也不再推荐使用。
3. **不支持前向保密的套件**
   - 如 RSA 密钥交换的套件，虽然依然被广泛使用，但不支持前向保密。
4. **MD5 作为哈希函数的套件**
   - MD5 已被证明不够安全，不应在任何加密套件中使用。

在配置加密套件时，确保优先选择支持前向保密的套件，并定期更新配置以应对新的安全威胁。

不过有时候要注意的是，有时候比较强的加密套件，会有一定的系统兼容问题，比如服务端开启了 TLS 1.2, 但是客户端是 window 7, 可能就会需要兼容一些稍微弱一点的加密，比如
```text
DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256
```
所以还是要看系统的兼容性的取舍。关于配置 SSL，可以通过 [SSL Configuration Generator](https://ssl-config.mozilla.org/) 来协助配置

### 1.3 HTTP 严格传输安全 (HSTS)
HTTP 严格传输安全（HTTP Strict Transport Security，HSTS）是一种网络安全策略机制，帮助保护网站免受一些形式的网络攻击，特别是中间人攻击（MITM）。它通过告知浏览器仅通过 HTTPS 连接访问该站点，而不是通过不安全的 HTTP 连接，从而增强了网站的安全性。

#### 工作原理
1. **HSTS 头部**：网站通过在响应头中包含 `Strict-Transport-Security` 来启用 HSTS。该头部指示浏览器在未来的一段时间内（由 `max-age` 参数指定）只使用 HTTPS 访问该网站。
2. **子域名保护**：HSTS 可以通过 `includeSubDomains` 参数扩展到网站的所有子域名，从而确保整个域名空间的安全。
3. **预加载列表**：网站可以选择加入浏览器的 HSTS 预加载列表，该列表包含了默认启用 HSTS 的网站，即使用户从未访问过这些网站。这需要网站所有者向浏览器厂商提交请求。

#### 优点
- **防止降级攻击**：HSTS 防止攻击者将用户从 HTTPS 降级到不安全的 HTTP。
- **中间人攻击防护**：通过强制使用加密连接，HSTS 能有效防止中间人攻击。

#### 配置示例

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

- `max-age=31536000`：浏览器应在未来一年（以秒为单位）强制使用 HTTPS。
- `includeSubDomains`：该策略适用于所有子域名。
- `preload`：指示网站已请求加入浏览器的 HSTS 预加载列表。

HSTS 是一种强大的工具，可以显著提升网站的安全性，但需要谨慎配置，以避免因误配置导致的访问问题。

关于 HSTS 的更多安全实战，可以看: {% post_link header-hsts %}

### 1.4 HTTP 公钥固定
HTTP 公钥固定（HTTP Public Key Pinning，HPKP）是一种安全机制，允许HTTPS网站通过HTTP标头指定其客户端（如浏览器）应该信任的公钥。这样，即使攻击者能够获取到伪造的证书，也无法成功冒充该网站，因为伪造证书的公钥不会匹配网站指定的公钥。

#### 工作原理
1. **固定公钥**：网站通过在响应头中包含 `Public-Key-Pins` 或 `Public-Key-Pins-Report-Only` 来指定一个或多个可信公钥的指纹（通常是SHA-256哈希）。
2. **有效期**：响应头中还包含一个 `max-age` 参数，指定浏览器应该在多长时间内记住这些公钥。
3. **备选公钥**：为防止主公钥丢失或被撤销，网站通常会指定一个或多个备选公钥。
4. **报告机制**：如果使用 `Public-Key-Pins-Report-Only`，浏览器不会阻止连接，但会将违规情况报告给指定的URI。

#### 安全性
HPKP 旨在防止中间人攻击和证书伪造。然而，由于配置错误可能导致网站无法访问，HPKP 需要谨慎配置。

出于这些原因，HPKP 的使用逐渐减少，并在一些浏览器中被弃用。现代网站更多地依赖其他机制，如严格传输安全（HSTS）来增强安全性。

### 1.5 全站 HTTPS
主站点通过 HTTPS 安全地服务，但是在 HTTP 上加载一些文件（images、js、css）。这是一个巨大的安全漏洞，破坏了 HTTPS 提供的安全性。

混合内容（Mixed Content）是一种安全漏洞。当一个通过 HTTPS 提供的网页加载了通过不安全的 HTTP 提供的资源（如图片、脚本、样式表等）时，就会出现混合内容问题。这种情况削弱了 HTTPS 的安全性，可能导致多种安全风险：
1. **会话劫持**：攻击者可以截获和篡改通过 HTTP 加载的资源，进而窃取用户的会话 cookie 或其他敏感信息。
2. **内容注入**：攻击者可以通过不安全的 HTTP 通道注入恶意代码，劫持用户的浏览器会话，或者在网页中植入恶意内容。
3. **中间人攻击（MITM）**：由于 HTTP 通信未加密，攻击者可以在传输过程中拦截和修改数据，HTTPS 本应防止这种攻击。

所以要全面使用 HTTPS，确保所有资源（包括图片、脚本、样式表等）都通过 HTTPS 加载。

### 1.6 CAA（Certification Authority Authorization）
CAA（Certification Authority Authorization）是一种 DNS 记录类型，用于指定哪些证书颁发机构（CAs）被授权为特定域名颁发 SSL/TLS 证书。CAA 记录通过增加对证书颁发过程的控制，帮助域名所有者提高其域名的安全性。

#### 工作原理
1. **DNS 记录配置**：域名所有者可以在其域名的 DNS 记录中添加 CAA 记录，明确指定允许哪些 CAs 为该域名颁发证书。
2. **CA 检查**：在颁发证书之前，证书颁发机构会查询域名的 CAA 记录，以确认其是否被授权为该域名颁发证书。
3. **防止未经授权的颁发**：如果 CAA 记录中未授权某个 CA，该 CA 就不应为该域名颁发证书。这有助于防止由于配置错误或恶意行为导致的未经授权的证书颁发。

#### CAA 记录的字段

- **标签（Tag）**：通常是 `issue`、`issuewild` 或 `iodef`。
  - `issue`：指定被授权颁发证书的 CA。
  - `issuewild`：指定被授权颁发通配符证书的 CA。
  - `iodef`：指定在检测到违反 CAA 记录的情况下，CA 应发送报告的电子邮件地址或 URL。

- **值（Value）**：CA 的名称或电子邮件地址/URL，用于报告用途。

#### 示例

以下是一个简单的 CAA 记录示例：

```
example.com.  CAA 0 issue "letsencrypt.org"
example.com.  CAA 0 iodef "mailto:admin@example.com"
```

- 上述记录允许 `letsencrypt.org` 为 `example.com` 颁发证书。
- 如果有任何违反 CAA 规则的情况，CA 将发送报告到 `admin@example.com`。

通过使用 CAA 记录，域名所有者可以更好地控制其域名的证书颁发过程，减少因错误或恶意行为导致的安全风险。

更多关于 CAA 的安全设置，可以看: {% post_link ssl-caa %}

### 1.7 OCSP Stapling
OCSP Stapling（在线证书状态协议捆绑）是一种安全增强策略，用于提高 SSL/TLS 证书验证过程的效率和安全性。

#### 工作原理
1. **传统 OCSP 查询**：通常情况下，浏览器在加载网站时会向证书颁发机构（CA）的 OCSP 服务器发送请求，以验证网站证书的状态（如是否被吊销）。这会导致额外的延迟，并可能暴露用户的浏览历史。
2. **OCSP Stapling**：通过 OCSP Stapling，服务器在建立 SSL/TLS 连接时，将来自 CA 的 OCSP 响应捆绑（“staple”）到其发送给客户端的握手消息中。这样，客户端无需单独联系 CA 的 OCSP 服务器即可验证证书状态。

#### 优势
- **性能提升**：减少了客户端与 CA 服务器之间的网络请求，从而降低了页面加载时间。
- **隐私保护**：客户端无需直接与 CA 服务器通信，减少了潜在的隐私泄露。
- **可靠性增强**：即使 CA 的 OCSP 服务器暂时不可用，客户端仍可以通过捆绑的 OCSP 响应验证证书状态。

#### 实施
服务器管理员需要在服务器配置中启用 OCSP Stapling。大多数现代 Web 服务器软件（如 Apache 和 Nginx）都支持此功能，并可以通过适当的配置选项启用。

OCSP Stapling 是一种推荐的安全实践，特别是在性能和隐私保护方面具有显著优势。它可以与其他安全机制（如 HSTS）结合使用，提供更全面的安全保障。

### 1.8 使用完整的证书链
使用完整的证书链可以提高 HTTPS 连接的安全性和可靠性。完整的证书链确保客户端能够验证服务器证书的信任性，从而建立安全的连接。

#### 什么是完整的证书链？
完整的证书链包括：
1. **服务器证书**：由证书颁发机构（CA）直接为你的域名颁发的证书。
2. **中间证书**：连接服务器证书和根证书的中间证书。中间证书用于将信任从根证书传递到服务器证书。
3. **根证书**：由 CA 自己签名的顶级证书，通常预安装在操作系统和浏览器中。

#### 为什么完整的证书链很重要？
1. **信任验证**：浏览器和客户端需要验证服务器证书的信任性。没有完整的证书链，客户端可能无法验证证书，从而导致连接失败。
2. **兼容性**：某些客户端（尤其是移动设备和某些旧版本的浏览器）可能无法自动下载缺失的中间证书，完整的证书链可以确保这些客户端能够成功建立连接。
3. **用户体验**：确保用户不会因为证书验证失败而看到安全警告，这有助于提高用户信任和网站的可靠性。

#### 如何配置完整的证书链？
1. **获取中间证书**：从你的证书颁发机构获取所有相关的中间证书。
2. **配置服务器**：在服务器配置中，确保将服务器证书和所有中间证书组合成一个证书链文件，并在配置中引用该文件。
   - **Nginx 示例**：
     ```nginx
     ssl_certificate /path/to/fullchain.pem;
     ssl_certificate_key /path/to/privatekey.pem;
     ```

   - **Apache 示例**：
     ```apache
     SSLCertificateFile /path/to/server.crt
     SSLCertificateChainFile /path/to/chain.pem
     SSLCertificateKeyFile /path/to/privatekey.key
     ```

通过确保使用完整的证书链，可以提高网站的安全性和兼容性，为用户提供更好的体验。


## 2. 内容安全
上面第一部分主要是证书和连接的安全， 接下来讲一下怎么对 HTTPS 的内容进行安全加固

### 2.1 内容安全策略 CSP
内容安全策略（Content Security Policy，CSP）是一种网络安全标准，用于防止多种类型的攻击，特别是跨站脚本（XSS）和数据注入攻击。CSP 通过定义允许加载的内容源，帮助网站管理员控制哪些资源可以在网页上执行。

#### 工作原理
CSP 是通过 HTTP 响应头或 `<meta>` 标签来实现的。网站管理员可以指定一组策略，指示浏览器允许加载哪些类型的资源以及从哪些来源加载。

#### 常见的 CSP 指令
- **default-src**：指定默认的内容源，如果其他特定指令未定义时使用。
- **script-src**：限制可以执行脚本的来源。
- **style-src**：限制可以应用样式表的来源。
- **img-src**：限制可以加载图像的来源。
- **font-src**：限制可以加载字体的来源。
- **connect-src**：限制可以发起连接（如 AJAX 请求）的来源。
- **frame-src**：限制可以嵌入框架的来源。

#### 示例
下面是一个基本的 CSP 示例，它只允许从同一来源加载脚本和样式：

```http
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self'; img-src 'self' https://example.com;
```

- `'self'` 表示允许从与文档相同的来源加载资源。
- `https://example.com` 表示允许从指定的域加载图像。

#### 优势
- **防止 XSS 攻击**：通过限制脚本来源，CSP 可以有效防止恶意脚本的注入和执行。
- **减少数据注入风险**：通过控制资源的来源，CSP 可以防止未经授权的数据注入。
- **增强内容安全**：CSP 提供了一个强大的工具来定义和执行内容加载策略，从而提高网站的整体安全性。

CSP 是一种强大的安全机制，能够显著减少常见攻击的风险。为了实现最佳效果，网站管理员应根据具体需求仔细设计和测试 CSP 策略。

关于更多关于 CSP 的安全实战，可以看: {% post_link csp %}

### 2.2 X-Frame-Options
`X-Frame-Options` 是一种 HTTP 响应头，用于提高网站的安全性，防止点击劫持（clickjacking）攻击。点击劫持是一种恶意技术，攻击者通过在透明或不可见的框架中加载另一个网站的页面，诱使用户在不知情的情况下点击按钮或链接。

#### `X-Frame-Options` 的指令
`X-Frame-Options` 头部可以设置以下指令：

1. **DENY**：
   - 页面不能在框架、iframe 或 object 中显示，无论是来自同源还是异源。
2. **SAMEORIGIN**：
   - 页面可以在相同来源的框架中显示，但不能在其他来源的框架中显示。
3. **ALLOW-FROM uri**（注意：此选项已被大多数浏览器废弃）：
   - 页面可以在指定的来源的框架中显示。由于安全性和兼容性问题，这个选项不再被广泛支持。

#### 示例
在 Nginx 或 Apache 中配置 `X-Frame-Options` 头部的示例：

- **Nginx**：

  ```nginx
  add_header X-Frame-Options "SAMEORIGIN";
  ```

- **Apache**：

  ```apache
  Header always set X-Frame-Options "SAMEORIGIN"
  ```

#### 作用
- **防止点击劫持**：通过限制页面在框架中的加载，`X-Frame-Options` 可以有效防止点击劫持攻击。
- **增强安全性**：这种简单的安全措施有助于保护用户和网站免受某些类型的网络攻击。

虽然 `X-Frame-Options` 提供了基本的防御，但现代网站通常结合使用内容安全策略（CSP）中的 `frame-ancestors` 指令来实现更灵活和强大的框架加载控制。

关于更多关于 `X-Frame-Options` 指令的安全实战，可以看: {% post_link web-forbidden-iframe-embed %}

### 2.3 X-XSS-Protection
`X-XSS-Protection` 是一种 HTTP 响应头，用于启用或禁用浏览器的内置跨站脚本（XSS）过滤器。XSS 是一种常见的安全漏洞，攻击者可以通过在网页中注入恶意脚本来窃取用户信息或劫持用户会话。

#### `X-XSS-Protection` 的指令
1. **0**：
   - 禁用 XSS 过滤器。浏览器将不会对检测到的 XSS 攻击进行任何干预。
2. **1**：
   - 启用 XSS 过滤器（这是大多数浏览器的默认设置）。当浏览器检测到 XSS 攻击时，会尝试清理页面以移除恶意脚本。
3. **1; mode=block**：
   - 启用 XSS 过滤器，并在检测到攻击时阻止页面加载，而不是尝试清理页面。
4. **1; report=<reporting-URI>**（注意：此功能并不广泛支持）：
   - 启用 XSS 过滤器，并在检测到攻击时将详细信息报告到指定的 URI。

#### 示例
在 Nginx 或 Apache 中配置 `X-XSS-Protection` 头部的示例：

- **Nginx**：

  ```nginx
  add_header X-XSS-Protection "1; mode=block";
  ```

- **Apache**：

  ```apache
  Header set X-XSS-Protection "1; mode=block"
  ```

#### 作用
- **防止 XSS 攻击**：通过启用浏览器的内置 XSS 过滤器，`X-XSS-Protection` 头部可以帮助减轻某些类型的 XSS 攻击。
- **增强用户安全**：在浏览器支持的情况下，这个头部提供了一层额外的安全保护。

需要注意的是，现代的防御策略倾向于使用内容安全策略（CSP）来提供更强大和灵活的 XSS 防御，因为 CSP 可以更全面地控制哪些脚本可以执行。`X-XSS-Protection` 头部主要用于提供一种额外的、较为基础的保护措施。

### 2.4 Cache-Control
在配置 HTTPS 网站的缓存策略时，`Cache-Control` 头部是一个关键的工具。它可以帮助管理浏览器和中间缓存的行为，确保敏感信息不被不当缓存。

在 HTTPS 环境中使用 `Cache-Control` 头部时需要注意的事项：

1. **敏感数据不缓存**：
   - 对于包含敏感信息的页面（如用户账户页面、支付页面等），应避免缓存。可以使用以下指令：
     ```http
     Cache-Control: no-store, no-cache, must-revalidate, private
     ```

2. **公共资源缓存**：
   - 对于不包含敏感信息的公共资源（如静态文件：CSS、JavaScript、图像等），可以利用缓存来提高性能：
     ```http
     Cache-Control: public, max-age=31536000
     ```
   - 这里的 `max-age` 可以根据资源的更新频率进行调整。

3. **确保一致性**：
   - 使用 `must-revalidate` 确保缓存过期后，必须从源服务器重新验证：
     ```http
     Cache-Control: no-cache, must-revalidate
     ```

4. **防止中间缓存**：
   - 对于需要确保仅在客户端缓存的内容，可以使用 `private` 指令：
     ```http
     Cache-Control: private, max-age=0, no-cache
     ```

5. **避免缓存混淆**：
   - 使用唯一的 URL 或查询参数来区分不同版本的资源，以防止缓存混淆。

6. **结合其他安全头**：
   - 配合使用其他安全头（如 HSTS、CSP）进一步增强安全性。

7. **敏感响应头**：
   - 确保敏感响应头（如 `Set-Cookie`）与 `Cache-Control: no-store` 一起使用，以防止敏感信息被缓存。

通过合理配置 `Cache-Control` 头部，可以有效控制缓存行为，保护敏感信息，同时优化资源加载性能。

关于更多浏览器缓存的用法，可以看: {% post_link browser-cache %}

### 2.5 X-Content-Type-Options
`X-Content-Type-Options` 是一个 HTTP 响应头，用于防止浏览器对响应的内容类型进行错误的 MIME 类型猜测（MIME sniffing）。这个头部可以帮助防止某些类型的攻击，例如跨站脚本（XSS）攻击。

#### `X-Content-Type-Options` 的指令
- **nosniff**：
  - 当设置为 `nosniff` 时，浏览器被指示不要对响应的内容类型进行猜测，而是严格按照服务器发送的 `Content-Type` 头部来处理内容。这有助于确保浏览器不会错误地执行可能包含恶意代码的文件。

#### 示例
在 Nginx 或 Apache 中配置 `X-Content-Type-Options` 头部的示例：

- **Nginx**：

  ```nginx
  add_header X-Content-Type-Options "nosniff";
  ```

- **Apache**：

  ```apache
  Header set X-Content-Type-Options "nosniff"
  ```

#### 作用
- **防止 MIME 类型混淆**：通过禁止浏览器的 MIME 类型猜测，`X-Content-Type-Options: nosniff` 可以防止浏览器将非脚本文件当作脚本执行。
- **增强安全性**：特别是在处理用户上传的文件时，这个头部可以减少某些类型的 XSS 攻击风险。

通过使用 `X-Content-Type-Options` 头部，网站可以更好地控制内容的处理方式，减少潜在的安全漏洞。

### 2.6 Subresource Integrity
子资源完整性（Subresource Integrity，SRI）是一种安全特性，允许浏览器在加载外部资源（如脚本或样式表）时验证其完整性。SRI 通过使用加密哈希值来确保资源在传输过程中未被篡改。

#### 工作原理
1. **生成哈希**：在使用外部资源时，开发者计算该资源的哈希值（通常使用 SHA-256、SHA-384 或 SHA-512）。
2. **添加 SRI 属性**：在 HTML 中，开发者将生成的哈希值添加到资源的 `<script>` 或 `<link>` 标签中，使用 `integrity` 属性。
3. **浏览器验证**：当浏览器加载资源时，它会计算资源的哈希值并与 `integrity` 属性中提供的哈希值进行比较。如果匹配，资源被认为是完整的并执行；如果不匹配，资源加载将被阻止。

#### 示例
假设你有一个外部 JavaScript 文件 `example.js`，可以这样使用 SRI：

```html
<script src="https://example.com/example.js"
        integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxc8bY5P7K4pD3rN7k5e3h5Q5f5e5f5"
        crossorigin="anonymous"></script>
```

- `integrity` 属性包含资源的哈希值。
- `crossorigin` 属性通常与 SRI 一起使用，以确保跨域资源共享（CORS）策略允许资源的完整性检查。

#### 优势
- **防止篡改**：SRI 确保外部资源在传输过程中未被篡改，增加了网站的安全性。
- **保护用户**：即使外部资源被攻击者替换为恶意版本，SRI 也能阻止恶意代码的执行。

SRI 是一种简单而有效的方法，可以显著提高网站的安全性，特别是在使用第三方资源时。开发者应尽可能为所有外部资源使用 SRI，以防止潜在的安全威胁。

### 2.7 Iframe Sandbox
iframe 在 WWW 上随处可见。主要用于装载第三方内容。这些 iframe 有很多方法来伤害托管网站，包括运行脚本和插件和重新引导访问者。sandbox 属性允许对 iframe 中可以进行的操作进行限制。

`iframe` 的 `sandbox` 属性是一种用于增强安全性的 HTML 特性。它通过限制 `iframe` 中内容的能力来减少潜在的安全风险，特别是在嵌入第三方内容时。

#### `sandbox` 属性的功能
当你为 `iframe` 设置 `sandbox` 属性时，浏览器会应用一组默认的限制，阻止 `iframe` 中的内容执行某些操作。可以通过指定一组允许的特性来解除部分限制。

#### 默认限制
启用 `sandbox` 属性后，`iframe` 中的内容将默认无法：
- 执行脚本。
- 提示用户下载文件。
- 使用表单提交。
- 修改父页面的内容。
- 访问浏览器的某些功能（如弹出窗口）。

#### 解除限制的特性
通过为 `sandbox` 属性添加特性，可以有选择地解除某些限制：
- **`allow-scripts`**：允许 `iframe` 中的内容执行脚本。
- **`allow-same-origin`**：允许 `iframe` 中的内容被视为与加载它的页面同源。
- **`allow-forms`**：允许 `iframe` 中的内容提交表单。
- **`allow-popups`**：允许 `iframe` 中的内容打开弹出窗口。
- **`allow-modals`**：允许使用 `alert()` 和 `prompt()` 等对话框。
- **`allow-orientation-lock`**：允许锁定屏幕方向。
- **`allow-pointer-lock`**：允许使用鼠标锁定 API。
- **`allow-presentation`**：允许进入全屏模式。
- **`allow-top-navigation`**：允许 `iframe` 导航其顶层浏览上下文。

#### 示例
```html
<iframe src="https://example.com" sandbox="allow-scripts allow-same-origin"></iframe>
```

在这个示例中，`iframe` 可以执行脚本并被视为同源，但仍然受到其他限制。

#### 优势
- **安全性增强**：通过限制 `iframe` 的功能，可以减少跨站脚本（XSS）和点击劫持等攻击的风险。
- **细粒度控制**：开发者可以根据需求精确控制 `iframe` 的行为。

`iframe sandbox` 是一种强大的工具，尤其在嵌入不受信任的第三方内容时，能够显著提升网页的安全性。

### 2.8 Server Clock
服务器时钟（Server Clock）指的是服务器的系统时间和日期设置。保持服务器时钟的准确性对于网络安全和许多计算机系统的正常运行至关重要。

#### 重要性
1. **加密协议**：许多加密协议（如 TLS/SSL）依赖于准确的时间戳来验证证书的有效性和防止重放攻击。如果服务器时钟不准，可能导致证书被错误地视为过期或未生效，从而导致连接失败。
2. **日志记录**：准确的时间戳对于日志记录和事件追踪至关重要。它有助于在发生安全事件时进行有效的审计和调查。
3. **调度任务**：服务器上许多任务（如备份、批处理任务）依赖于正确的时间调度。如果时钟不准确，可能导致任务在错误的时间执行。
4. **身份验证**：一些身份验证协议（如 Kerberos）依赖于时间同步来防止票据重放攻击和确保身份验证请求的有效性。

#### 维护服务器时钟的准确性
1. **网络时间协议（NTP）**：使用 NTP 是保持服务器时钟准确的常见方法。NTP 是一种网络协议，用于通过网络将计算机时钟与协调世界时（UTC）同步。
2. **定期检查和校准**：定期检查服务器时钟的准确性，并根据需要进行校准，以确保其与标准时间源保持一致。
3. **使用可靠的时间源**：选择可靠的 NTP 服务器作为时间源，例如国家授时中心或知名的公共 NTP 服务（如 `pool.ntp.org`）。

通过确保服务器时钟的准确性，可以增强系统的安全性和可靠性，避免因时间不准确导致的各种问题。

## 3. 减少信息披露
为什么要减少信息披露
1. **减少攻击面**：通过隐藏服务器软件和版本信息，可以防止攻击者轻松识别使用的特定软件版本及其已知漏洞。
2. **增加攻击难度**：攻击者在不了解服务器详细信息的情况下，需要花费更多时间和资源来探测和发动攻击。
3. **良好的安全实践**：最小化信息披露是安全领域的一个基本原则，旨在减少潜在的攻击向量。

### 3.1 Server Banner
减少服务器横幅（Server Banner）中的信息披露可以提高安全性。服务器横幅通常包含有关服务器软件及其版本的信息，这些信息可能被攻击者用来识别潜在的漏洞或攻击向量。

#### 如何减少服务器横幅的信息披露
- **Nginx**：
  - 修改 `nginx.conf` 文件，添加或修改以下行以隐藏版本号：
    ```nginx
    server_tokens off;
    ```

- **Apache**：
  - 修改 `httpd.conf` 文件，添加或修改以下行以隐藏版本号和操作系统信息：
    ```apache
    ServerTokens Prod
    ServerSignature Off
    ```

- **其他服务器软件**：检查相应的文档，寻找配置选项以隐藏或修改服务器标识。

通过减少信息披露，结合其他安全措施，可以显著增强服务器的整体安全性。

### 3.2 Web Framework Information
许多 web 框架设置 HTTP 头，识别框架或版本号。除了满足用户的好奇心，而且主要作为技术堆栈的广告，这几乎没有什么作用。这些头是不标准的，对浏览器渲染站点的方式没有影响。

我们应去除应用程序响应中有关所使用的 Web 框架（如 Django、Ruby on Rails、ASP.NET 等）及其版本的信息。这种信息披露可能会被攻击者利用来识别特定框架的已知漏洞，从而发动攻击。

#### 为什么要减少 Web 框架信息披露
1. **降低攻击风险**：攻击者常常利用已知的框架漏洞进行攻击。如果框架和版本信息被隐藏，攻击者将难以确定可利用的漏洞。
2. **增加攻击难度**：隐藏框架信息迫使攻击者采用更复杂的方法来探测和攻击应用程序。
3. **遵循安全最佳实践**：最小化信息披露是安全领域的基本原则，有助于减少潜在的攻击向量。

#### 如何减少 Web 框架信息披露
从服务器响应中删除这些标头: `X-Powered-By`, `X-Runtime`, `X-Version` 和 `X-AspNet-Version`。

通过减少 Web 框架信息的披露，可以显著提高应用程序的安全性，减少被攻击者利用的风险。

## 4. Cookie 安全
确保 Cookie 的安全性是保护用户数据和维护应用程序完整性的重要方面。以下是一些关键的安全加固措施，可以帮助提高 Cookie 的安全性：

1. **使用 `Secure` 标志**：
   - 确保敏感的 Cookie（如会话 Cookie）仅通过 HTTPS 传输，防止在不安全的 HTTP 连接上传输时被拦截。
   - 设置 `Secure` 标志：
     ```http
     Set-Cookie: sessionId=abc123; Secure
     ```

2. **使用 `HttpOnly` 标志**：
   - 防止客户端脚本（如 JavaScript）访问 Cookie，减少跨站脚本（XSS）攻击的风险。
   - 设置 `HttpOnly` 标志：
     ```http
     Set-Cookie: sessionId=abc123; HttpOnly
     ```

3. **使用 `SameSite` 属性**：
   - 控制 Cookie 在跨站请求中是否被发送，减少跨站请求伪造（CSRF）攻击的风险。
   - 可选值：
     - `Strict`：Cookie 不会在跨站请求中发送。
     - `Lax`：Cookie 在一些跨站请求中发送（如导航到目标站点的 GET 请求）。
     - `None`：Cookie 在所有请求中发送（需要 `Secure` 标志）。
   - 设置 `SameSite` 属性：
     ```http
     Set-Cookie: sessionId=abc123; SameSite=Lax
     ```

4. **设置合理的过期时间**：
   - 为 Cookie 设置适当的 `Expires` 或 `Max-Age` 属性，限制其有效期，减少长期暴露的风险。
   - 示例：
     ```http
     Set-Cookie: sessionId=abc123; Max-Age=3600
     ```

5. **限制 Cookie 的作用域**：
   - 使用 `Domain` 和 `Path` 属性限制 Cookie 的作用域，仅在必要的情况下发送。
   - 示例：
     ```http
     Set-Cookie: sessionId=abc123; Domain=example.com; Path=/account
     ```

6. **加密敏感数据**：
   - 对敏感的 Cookie 数据进行加密，以防止在传输过程中被窃取或篡改。

7. **定期更新和审计**：
   - 定期审查 Cookie 的使用和配置，确保符合最新的安全标准和最佳实践。

通过实施这些安全措施，可以显著提高 Cookie 的安全性，保护用户数据免受各种网络攻击的威胁。

## 总结
通过合理的配置上述的加固建议，可以让你的 HTTPS 更加的安全!!

---
参考资料:
- [如何让HTTPS站点评级达到A+? 还得看这篇HTTPS安全优化配置最佳实践指南](https://mp.weixin.qq.com/s/ggU_5sIZXQGhdZkUOa1waw)
- [HTTPS 安全最佳实践（一）之SSL/TLS部署](https://blog.myssl.com/ssl-and-tls-deployment-best-practices/)
- [HTTPS 安全最佳实践（二）之安全加固](https://blog.myssl.com/https-security-best-practices/)

