---
title: 浅谈之-在浏览器中输入一个网址后发生了什么
date: 2018-07-01 23:42:57
tags: js
categories: 前端相关
---
这个是一道非常经典的面试题，也是我在面试前端和后端经常会问的一个问题。因为这道题可以很大程度上考验一个开发者的技术深度和广度，而不仅仅限于平时日常工作中做的那些东西。
这道题严格上来说并没有标准答案，因为每一个环节都可以做非常深的延展。我这边也是参考一些网上的答案，然后配合自己的见解，给出一个我自己认为的一个比较全的答案。
主要分为以下几个环境：
1. 浏览器解析该URL得到IP地址
2. 浏览器根据解析得到的IP地址向服务器发送一个 HTTP 请求
3. 服务器收到请求并进行处理，最后返回响应
4. 浏览器对该响应进行解码，渲染显示
5. 断开 TCP 连接，并将资源进行本地缓存
<!--more-->
---

# 1 浏览器解析该URL得到IP地址
## 1.1 浏览器解析输入的字符串是 URL 还是搜索的关键字？
浏览器可以通过解析输入的字符串来判断当前输入的是URL还是搜索的关键字。当协议或主机名不合法时，即认为输入的不是URL，而是关键字，这时候浏览器会将地址栏中输入的文字传给默认的搜索引擎。比如如果是Chrome，那么就是google搜索引擎。大部分情况下，在把文字传递给搜索引擎的时候，URL会带有特定的一串字符，用来告诉搜索引擎这次搜索来自这个特定浏览器。
## 1.2 转换非 ASCII 的 Unicode 字符
浏览器检查输入是否含有不是 a-z， A-Z，0-9， - 或者 . 的字符，如果有非ASCII的字符的话，浏览器会对主机名部分使用 [Punycode](https://en.wikipedia.org/wiki/Punycode) 编码,  如果输入的是中文域名就会经过这一个步骤，这时候就会先解析成可用於DNS系统的编码，以便于后续的DNS查询。
## 1.3 检查 HSTS 列表
浏览器检查自带的“预加载 HSTS（HTTP严格传输安全）”列表，这个列表里包含了哪些请求浏览器只使用HTTPS进行连接的网站。如果网站在这个列表里，浏览器会使用 HTTPS 而不是 HTTP 协议，否则，最初的请求会使用HTTP协议发送。
注意，一个网站哪怕不在 HSTS 列表里，也可以要求浏览器对自己使用 HSTS 政策进行访问。浏览器向网站发出第一个 HTTP 请求之后，网站会返回浏览器一个响应，请求浏览器只使用 HTTPS 发送请求。然而，就是这第一个 HTTP 请求，却可能会使用户受到 [downgrade attack](https://en.wikipedia.org/wiki/Moxie_Marlinspike#SSL_stripping) 的威胁，也是就俗称的 <font color=red>SSL 剥离攻击</font>, 这也是为什么现代浏览器都预置了 HSTS 列表。
以nginx为例：如果设置这个头部的话，那么就会要求浏览器使用HSTS策略：
{% codeblock lang:js %}
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
{% endcodeblock %}
## 1.4 DNS 查询
DNS（Domain Name System，域名系统）因特网上作为域名和IP(Internet Protocol Address)地址相互映射的一个分布式数据库，能够使用户更方便的访问互联网，而不用去记住能够被机器直接读取的IP数串。在整个互联网体系中，约定俗成的用于标识网络上设备的地址是IP，然而我们输入的是域名，因为域名更方便人们记忆，不然那么多网站，人怎么可能记住所有的IP地址。
通过主机名，最终得到该主机名对应的IP地址的过程叫做域名解析（或主机名解析）,域名解析服务器是基于UDP协议实现的一个应用程序，通常通过监听53端口来获取客户端的域名解析请求。
### 1.4.1 浏览器缓存
浏览器会缓存DNS记录一段时间。 有趣的是，操作系统没有告诉浏览器储存DNS记录的时间，这样不同浏览器会储存个自固定的一个时间（2分钟到30分钟不等）。所以这时候浏览器就会检查域名是否在缓存当中。
比如如果要查看 Chrome 当中的缓存， 打开 chrome://net-internals/#dns :
![1](what-happens-when-you-navigate-to-a-url/1.png)
### 1.4.2 系统缓存
如果浏览器缓存中没有，浏览器会做一个系统调用（windows里是<font color=red>gethostbyname</font>）。这样便可获得系统缓存中的记录进行查询。<font color=red>gethostbyname</font> 函数在试图进行DNS解析之前首先检查域名是否在本地 Hosts 里，Hosts 的位置 [不同的操作系统有所不同](https://en.wikipedia.org/wiki/Hosts_%28file%29#Location_in_the_file_system)。
如果hosts文件没有，就会去查系统本地缓存的其他DNS 记录， 要查看系统的 DNS 缓存记录，以 windows 为例就是 <font color=red>ipconfig /displaydns </font>：
![1](what-happens-when-you-navigate-to-a-url/2.png)
以下这些 DNS 缓存记录，就包含 Hosts 文件里面的记录了.
![1](what-happens-when-you-navigate-to-a-url/3.png)
如果 <font color=red>gethostbyname</font> 没有这个域名的缓存记录，也没有在 Hosts 里找到，它将会向 DNS 服务器发送一条 DNS 查询请求。DNS 服务器是由网络通信栈提供的，通常是本地路由器或者 ISP 的缓存 DNS 服务器，即下面的流程。
### 1.4.3 路由器缓存
如果在系统缓存里面还是没找到对应的IP，那么接着会发送一个请求到路由器上，然后路由器在自己的路由器缓存上查找记录，路由器一般也存有DNS信息（缓存你上过的网站，所以有时路由器需要进行DNS刷新）
### 1.4.4 ISP DNS缓存
如果本地路由器还是没有，这个请求就会被发送到ISP（注：Internet Service Provider，互联网服务提供商，就是那些拉网线到你家里的运营商，中国电信中国移动什么的），ISP也会有相应的ISP DNS服务器，一听中国电信就知道这个DNS服务器的规模肯定不会小，所以基本上都能在这里找得到。
ps：会跑到这里进行查询是因为你没有改动过"网络中心"的"ipv4"的DNS地址，万恶的运营上可以改动这个DNS服务器，换句话说他们可以让你的浏览器跳转到他们设定的页面上，这也就是人尽皆知的DNS和HTTP劫持。我们也可以自行修改DNS服务器来防止DNS被ISP污染。
### 1.4.5 递归查询
如果在ISP DNS服务器还没有查到的话（一般情况下在ISP DNS服务器中都可以找到），你的ISP的DNS服务器会将请求发向根域名服务器进行搜索。根域名服务器就是面向全球的顶级DNS服务器，共有13台逻辑上的服务器，从A到M命名，真正的实体服务器则有几百台，分布于全球各大洲。所以这些服务器有真正完整的DNS数据库。
![1](what-happens-when-you-navigate-to-a-url/4.png)
举个例子：  
例如 "web.airdroid.com" 这个域名在 本地域名服务器上找不到
1. 这时候本地域名服务器就会到根域名服务器查找，根域名服务器说这个是一个.com域名。
2. 然后本地域名服务器就跑到管理.com域名的服务器上进行进一步查询，顶级域名服务器说是 .airdroid二级域名。
3. 最后本地域名服务器再跑到管理 .airdroid这个二级域名所在的权限域名服务器，去查询 web这个三级域名的ip 地址。

所以域名结构为：三级域名.二级域名.一级域名。
这里的查询过程是包含递归查询和迭代查询的，客户端主机发送给本地服务器的查询是递归查询，而后面的三个查询是迭代查询。
如果到了根域名服务器还是找不到域名的对应信息，那只能说明一个问题：这个域名本来就不存在，它没有在网上正式注册过。或者卖域名的把它回收掉了（通常是因为欠费）。 这也就是为什么打开一个新页面会有点慢，因为本地没什么缓存，要这样递归地查询下去。
### 1.4.6 多IP域名DNS查询解决方案
这边还要注意一个细节，有些域名会涉及到多个ip， 比如 facebook.com, 有几种方法可以消除这个瓶颈：
1. [循环DNS](https://baike.baidu.com/item/DNS%E5%BE%AA%E7%8E%AF%E5%A4%8D%E7%94%A8) —— 单个域名、多个IP列表循环应对DNS查询
2. [负载均衡器](https://baike.baidu.com/item/%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1%E5%99%A8) —— 一个特定IP的负载均衡服务器（例如：反向代理服务器）负责监听请求并转发给后面的多个服务器集群的某一个，实现多个服务器负载均衡。 而且现在很多云服务都有提供负载均衡服务，比如AWS 的 ELB。
3. 地理DNS —— 根据用户所处地理位置，返回不同的IP（应用：CDN）
4. [anycast](https://baike.baidu.com/item/Anycast) —— 一个IP地址映射多个物理主机的路由技术

# 2 浏览器根据解析得到的IP地址向服务器发送一个 HTTP 请求
这个过程就会涉及到网络传输，就会涉及到网络5层模型：
![1](what-happens-when-you-navigate-to-a-url/5.png)
## 2.1  生成TCP数据包
当浏览器得到了目标服务器的 IP 地址，以及 URL 中给出来端口号（http 协议默认端口号是 80， https 默认端口号是 443），它会调用系统库函数 socket ，请求一个 TCP流套接字:
1. 这个请求首先被交给传输层，在传输层请求被封装成 TCP segment。目标端口会被加入头部，源端口会在系统内核的动态端口范围内选取（Linux下是ip_local_port_range)
2. TCP segment 被送往网络层，网络层会在其中再加入一个 IP 头部，里面包含了目标服务器的IP地址以及本机的IP地址，把它封装成一个IP packet。
3. 这个 TCP packet 接下来会进入链路层，链路层会在封包中加入 frame 头部，里面包含了本地内置网卡的MAC地址以及网关（本地路由器）的 MAC 地址。如果内核不知道网关的 MAC 地址，它必须进行 ARP 广播来查询其地址。

举个例子： 如果把一台主机比作一座房子，把进程比作房子里面的房间，Socket相当于房间的门。不管是房间的人要出来还是外面的人要去到某一个房间，都必须先通过Socket这一道门。
套接字的作用是实现传输层的多路复用和多路分解。在应用层可以同时运行多个进程，每个进程都需要通过传输层来收发分组，而传输层的TCP进程只有一个，当TCP进程收到一个分组后，该怎么确定应该转发给哪个进程呢？答案是通过套接字，这就是多路分解。
## 2.2 建立TCP 连接
这时候TCP数据包已经封装好了，接下来就是传输了。对于大部分家庭网络和小型企业网络来说，数据包会从本地计算机出发，经过本地网络，再通过调制解调器把数字信号转换成模拟信号，使其适于在电话线路，有线电视光缆和无线电话线路上传输。在传输线路的另一端，是另外一个调制解调器，它把模拟信号转换回数字信号，交由下一个 网络节点 处理。节点的目标地址和源地址将在后面讨论。
大型企业和比较新的住宅通常使用光纤或直接以太网连接，这种情况下信号一直是数字的，会被直接传到下一个 网络节点 进行处理。
最终数据包会到达管理本地子网的路由器。在那里出发，它会继续经过自治区域(autonomous system, 缩写 AS)的边界路由器，其他自治区域，最终到达目标服务器。一路上经过的这些路由器会从IP数据报头部里提取出目标地址，并将封包正确地路由到下一个目的地。IP数据报头部 time to live (TTL) 域的值每经过一个路由器就减1，如果封包的TTL变为0，或者路由器由于网络拥堵等原因封包队列满了，那么这个包会被路由器丢弃。
上面报文的发送和接受过程在 TCP 连接的三次握手过程中会发生很多次：
![1](what-happens-when-you-navigate-to-a-url/6.png)
1. Client首先发送一个连接试探，ACK=0 表示确认号无效，SYN = 1 表示这是一个连接请求或连接接受报文，同时表示这个数据报不能携带数据，seq = x 表示Client自己的初始序号（seq = 0 就代表这是第0号包），这时候Client进入syn_sent状态，表示客户端等待服务器的回复
2. Server监听到连接请求报文后，如同意建立连接，则向Client发送确认。TCP报文首部中的SYN 和 ACK都置1 ，ack = x + 1表示期望收到对方下一个报文段的第一个数据字节序号是x+1，同时表明x为止的所有数据都已正确收到（ack=1其实是ack=0+1,也就是期望客户端的第1个包），seq = y 表示Server 自己的初始序号（seq=0就代表这是服务器这边发出的第0号包）。这时服务器进入syn_rcvd，表示服务器已经收到Client的连接请求，等待client的确认。
3. Client收到确认后还需再次发送确认，同时携带要发送给Server的数据。ACK 置1 表示确认号ack= y + 1 有效（代表期望收到服务器的第1个包），Client自己的序号seq= x + 1（表示这就是我的第1个包，相对于第0个包来说的），一旦收到Client的确认之后，这个TCP连接就进入Established状态，就可以发起http请求了。

## 2.3 TLS 握手 
如果是https的，那么就会在TCP三次握手之后，再多一层 TLS 握手，具体握手的流程如下：
![1](what-happens-when-you-navigate-to-a-url/7.png)
### 2.3.1 Client Hello
握手第一步是客户端向服务端发送 Client Hello 消息，这个消息里包含了一个客户端生成的随机数 Random1、客户端支持的加密套件（Support Ciphers）和 SSL Version 等信息。
### 2.3.2 Server Hello
#### 2.3.2.1 Server Hello
第二步是服务端向客户端发送 Server Hello 消息，这个消息会从 Client Hello 传过来的 Support Ciphers 里确定一份加密套件，这个套件决定了后续加密和生成摘要时具体使用哪些算法，另外还会生成一份随机数 Random2。注意，至此客户端和服务端都拥有了两个随机数（Random1+ Random2），这两个随机数会在后续生成对称秘钥时用到。
![1](what-happens-when-you-navigate-to-a-url/8.png)
#### 2.3.2.2 Certificate
同时服务端还会将自己的证书下发给客户端，让客户端验证自己的身份，客户端验证通过后取出证书中的公钥。
![1](what-happens-when-you-navigate-to-a-url/9.png)
#### 2.3.2.3 Server Key Exchange
如果是DH算法，这里发送服务器使用的DH参数。RSA算法不需要这一步。
![1](what-happens-when-you-navigate-to-a-url/10.png)
#### 2.3.2.4 Certificate Request
另外还有一个就是 Certificate Request  报文，Certificate Request 是服务端要求客户端上报证书，这一步是可选的，对于安全性要求高的场景会用到。
### 2.3.3 Server Hello Done
Server Hello Done 通知客户端 Server Hello 过程结束。
![1](what-happens-when-you-navigate-to-a-url/11.png)
### 2.3.4 Certificate Verify
客户端收到服务端传来的证书后，先从 CA 验证该证书的合法性，验证通过后取出证书中的服务端公钥，再生成一个随机数 Random3，再用服务端公钥非对称加密 Random3生成 PreMaster Key。
### 2.3.5 Client Key Exchange
上面客户端根据服务器传来的公钥生成了 PreMaster Key，Client Key Exchange 就是将这个 key 传给服务端，服务端再用自己的私钥解出这个 PreMaster Key 得到客户端生成的 Random3。至此，客户端和服务端都拥有 Random1 + Random2 + Random3，两边再根据同样的算法就可以生成一份秘钥，握手结束后的应用层数据都是使用这个秘钥进行对称加密。为什么要使用三个随机数呢？这是因为 SSL/TLS 握手过程的数据都是明文传输的，并且多个随机数种子来生成秘钥不容易被暴力破解出来。客户端将 PreMaster Key 传给服务端的过程如下图所示：
![1](what-happens-when-you-navigate-to-a-url/12.png)
### 2.3.6 Change Cipher Spec(Client)
这一步是客户端通知服务端后面再发送的消息都会使用前面协商出来的秘钥加密了，是一条事件消息。
![1](what-happens-when-you-navigate-to-a-url/13.png)
### 2.3.7 Encrypted Handshake Message(Client)
这一步是服务端通知客户端后面再发送的消息都会使用加密，也是一条事件消息。
### 2.3.8 Encrypted Handshake Message(Server)
这一步对应的是 Server Finish 消息，服务端也会将握手过程的消息生成摘要再用秘钥加密，这是服务端发出的第一条加密消息。客户端接收后会用秘钥解密，能解出来说明协商的秘钥是一致的。
![1](what-happens-when-you-navigate-to-a-url/14.png)
### 2.3.9 Application Data
到这里，双方已安全地协商出了同一份秘钥，所有的应用层数据都会用这个秘钥加密后再通过 TCP 进行可靠传输。
## 2.4 发送http请求
TCP 通道已经建立了，那么就是发送http请求了。
下面就是一个请求头：
{% codeblock lang:js %}
Request URL:http://www.airdroid.com/
Request Method:GET

Accept:text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Accept-Encoding:gzip, deflate
Accept-Language:en-US,en;q=0.9,zh-CN;q=0.8,zh;q=0.7,zh-TW;q=0.6
Cache-Control:max-age=0
Connection:keep-alive
Cookie:_ga=GA1.2.1143934626.1528079240; versionCode=1805311653; 
Host:web.airdroid.com
If-Modified-Since:Fri, 22 Jun 2018 03:22:12 GMT
If-None-Match:"705914acbcfec78c6f6cbfbb4954025e"
Upgrade-Insecure-Requests:1
User-Agent:Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/67.0.3396.87 Safari/537.36
{% endcodeblock %}
这个get请求包含了主机（host）、用户代理(User-Agent)，用户代理就是自己的浏览器，它是你的"代理人"，Connection（连接属性）中的keep-alive表示浏览器告诉对方服务器在传输完现在请求的内容后不要断开连接，不断开的话下次继续连接速度就很快了。其他的顾名思义就行了。还有一个重点是Cookies，Cookies保存了用户的登陆信息，在每次向服务器发送请求的时候会重复发送给服务器。Chrome上的F12与Firefox上的firebug(快捷键shift+F5)均可查看这些信息。

http 请求报文和响应报文的格式：
{% codeblock lang:js %}
起始行：如 GET / HTTP/1.0 （请求的方法 请求的URL 请求所使用的协议）
头部信息：User-Agent Host等成对出现的值
主体
{% endcodeblock %}
# 3 服务器收到请求并进行处理，最后返回响应
## 3.1 http 服务器请求处理

HTTPD(HTTP Daemon)在服务器端处理请求/响应。最常见的 HTTPD 有 Linux 上常用的 Apache 和 nginx，以及 Windows 上的 IIS。
当HTTPD收到请求之后（假设当前请求的是 www.airdroid.com ），这时候服务器把请求拆分成以下几个参数：
1. HTTP 请求方法(GET, POST, HEAD, PUT, DELETE, CONNECT, OPTIONS, 或者 TRACE)。直接在地址栏中输入 URL 这种情况下，使用的是 GET 方法
2. 域名：www.airdroid.com
3. 请求路径/页面：/ (我们没有请求www.airdroid.com下的指定的页面，因此 / 是默认的路径)

然后开始进行以下校验：
1. 服务器验证其上已经配置了 www.airdroid.com 的虚拟主机
2. 服务器验证 www.airdroid.com 接受 GET 方法
3. 服务器验证该用户可以使用 GET 方法(根据 IP 地址，身份信息等)
4. 如果服务器安装了 URL 重写模块（例如 Apache 的 mod_rewrite 和 IIS 的 URL Rewrite），服务器会尝试匹配重写规则，如果匹配上的话，服务器会按照规则重写这个请求。
5. 服务器根据请求信息获取相应的响应内容，这种情况下由于访问路径是 "/" ,会访问首页文件（你可以重写这个规则，但是这个是最常用的）。
6. 服务器会使用指定的处理程序分析处理这个文件，假如 www.airdroid.com 使用 PHP，服务器会使用 PHP 解析 index 文件，并捕获输出，把 PHP 的输出结果返回给请求者。

假设服务器端使用nginx+php(fastcgi)架构提供服务：
首先是nginx读取配置文件，当Nginx在收到 浏览器 GET / 请求时，会读取http请求里面的头部信息，根据Host来匹配 自己的所有的虚拟主机的配置文件的server_name,看看有没有匹配的，有匹配那么就读取该虚拟主机的配置，发现如下配置：
{% codeblock lang:js %}
root /web/echo
{% endcodeblock %}
通过这个就知道所有网页文件的就在这个目录下， 例如访问 http://www.airdroid.com/index.html ,那么代表 <font color=red>/web/echo</font>下面有个文件叫index.html。
{% codeblock lang:js %}
index index.html index.htm index.php
{% endcodeblock %}
通过这个就能得知网站的首页文件是哪个文件，也就是我们在输入 http://www.airdroid.com/ ，nginx就会自动帮我们把index.html（假设首页是index.php 当然是会尝试的去找到该文件，如果没有找到该文件就依次往下找，如果这3个文件都没有找到，那么就抛出一个404错误）加到后面，那么添加之后的URL是/index.php,然后根据后面的配置进行处理。
{% codeblock lang:js %}
location ~ .*\.php(\/.*)*$ {
  root /web/echo;
  fastcgi_pass 127.0.0.1:9000;
  fastcgi_index index.php;
  astcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
  include fastcgi_params;
}
{% endcodeblock %}
这一段配置指明凡是请求的URL中匹配（这里是启用了正则表达式进行匹配） *.php后缀的（后面跟的参数）都交给后端的fastcgi进程进行处理。于是nginx把/index.php这个URL交给了后端的fastcgi进程处理，等待fastcgi处理完成后（结合数据库查询出数据，填充模板生成html文件）返回给nginx一个index.html文档，nginx再把这个index.html返回给浏览器，于是乎浏览器就拿到了首页的html代码，同时nginx写一条访问日志到日志文件中去。

## 3.2 服务器返回http响应。
{% codeblock lang:js %}
Status Code:
304 Not Modified

Connection:keep-alive
Date:Sat, 30 Jun 2018 09:33:37 GMT
ETag:"705914acbcfec78c6f6cbfbb4954025e"
Last-Modified:Fri, 22 Jun 2018 03:22:12 GMT
Server:AmazonS3
x-amz-id-2:u92O2vB1z97lMUTzlFp0iVLhoxG3mZI0PFzxgG+CYKle2PN0GjdPTQ8u0gWtKEHVWlbLV1kFXEk=
x-amz-request-id:C5D34C185F72C339
X-Dscp-Value:0
X-Via:1.1 jfzhdx94:3 (Cdn Cache Server V2.0), 1.1 dongzhanjiang13:4 (Cdn Cache Server V2.0)
{% endcodeblock %}
大部分都是返回 200， 也就是响应成功。 当前还有其他的状态码，具体状态码的含义看[这边](http://tool.oschina.net/commons?type=5)， 这边主要是列一下常用的：
- <font color=red>101 Upgrade</font> 协议升级，比如将ws连接就会返回一个 101
- <font color=red>200 Success</font> 成功
- <font color=red>301 Moved Permanently</font> 永久重定向，比如很多网站直接访问二级域名的时候，会变成www开头的三级域名，这个就是一个永久重定向， 比如：
{% codeblock lang:js %}
airdroid.com -> www.airdroid.com
{% endcodeblock %}
![1](what-happens-when-you-navigate-to-a-url/15.png)
至于为什么要这么做，为什么服务器一定要重定向而不是直接发会用户想看的网页内容呢？其中一个原因跟搜索引擎排名有关。你看，如果一个页面有两个地址，就像 http://www.airdroid .com/ 和 http://airdroid.com/ ，搜索引擎会认为它们是两个网站，结果造成每一个的搜索链接都减少从而降低排名。而搜索引擎知道301永久重定向是 什么意思，这样就会把访问带www的和不带www的地址归到同一个网站排名下。还有一个是用不同的地址会造成缓存友好性变差。当一个页面有好几个名字时，它可能会在缓存里出现好几次。

- <font color=red>304 Not Modified</font> 未修改，比如本地缓存的资源文件和服务器上比较时，发现并没有修改，服务器返回一个304状态码。说到304，那么就要说到浏览器缓存机制，包含强缓存和协商缓存，而304就是协商缓存，资源没有改变之后，服务端返回的状态码， 具体看[这边](https://segmentfault.com/a/1190000011219392)
- <font color=red>401 Not Auth</font>， 当前请求需要用户验证。该响应必须包含一个适用于被请求资源的 WWW-Authenticate 信息头用以询问用户信息。
- <font color=red>403 Forbidden</font>， 服务器已经理解请求，但是拒绝执行它。与401响应不同的是，身份验证并不能提供任何帮助
- <font color=red>404 Not Found</font> 资源找不到
- <font color=red>500 Internal Server Error</font> 服务器内部错误，基本上就是服务端的程序有bug
- <font color=red>502 Bad Gateway</font> 前面代理服务器联系不到后端的服务器时出现
- <font color=red>504 Gateway Timeout</font> 这个是代理能联系到后端的服务器，但是后端的服务器在规定的时间内没有给代理服务器响应

# 4 浏览器对该响应进行解码，渲染显示
响应主体采用特定方法压缩，整个响应以blob类型传输，响应头指示响应主体以何种方式压缩。
## 4.1 是否进行解压
浏览器会根据响应头部是否有 <font color=red>Content-Encoding: gzip</font>，将响应报文的主体进行 gzip解压。
## 4.2 展示形态
根据Content-type 来判断要将主体如果显示，比如报头中把Content-type设置为“text/html”。报头让浏览器将该响应内容以HTML形式呈现，而不是以文件形式下载它。浏览器会根据报头信息决定如何解释该响应，不过同时也会考虑像URL扩展内容等其他因素。
## 4.3 解析 
说解析之前，要先说下组成浏览器的组件有：
- <font color=red>用户界面</font> 用户界面包含了地址栏，前进后退按钮，书签菜单等等，除了请求页面之外所有你看到的内容都是用户界面的一部分
- <font color=red>浏览器引擎</font> 浏览器引擎负责让 UI 和渲染引擎协调工作
- <font color=red>渲染引擎</font> 渲染引擎负责展示请求内容。如果请求的内容是 HTML，渲染引擎会解析 HTML 和 CSS，然后将内容展示在屏幕上
- <font color=red>网络组件</font> 网络组件负责网络调用，例如 HTTP 请求等，使用一个平台无关接口，下层是针对不同平台的具体实现
- <font color=red>UI后端</font> UI 后端用于绘制基本 UI 组件，例如下拉列表框和窗口。UI 后端暴露一个统一的平台无关的接口，下层使用操作系统的 UI 方法实现
- <font color=red>Javascript 引擎</font> Javascript 引擎用于解析和执行 Javascript 代码
- <font color=red>数据存储</font> 数据存储组件是一个持久层。浏览器可能需要在本地存储各种各样的数据，例如 Cookie 等。浏览器也需要支持诸如 localStorage，IndexedDB，WebSQL 和 FileSystem 之类的存储机制

### 4.3.1 解析 HTML，生成 DOM 树
浏览器渲染引擎从网络层取得请求的文档，一般情况下文档会分成8kB大小的分块传输。HTML 解析器的主要工作是对 HTML 文档进行解析，生成解析树。
解析树是以 DOM 元素以及属性为节点的树。DOM是文档对象模型(Document Object Model)的缩写，它是 HTML 文档的对象表示，同时也是 HTML 元素面向外部(如Javascript)的接口。树的根部是"Document"对象。整个 DOM 和 HTML 文档几乎是一对一的关系。
HTML不能使用常见的自顶向下或自底向上方法来进行分析。主要原因有以下几点:
- 语言本身的“宽容”特性
- HTML 本身可能是残缺的，对于常见的残缺，浏览器需要有传统的容错机制来支持它们
- 解析过程需要反复。对于其他语言来说，源码不会在解析过程中发生变化，但是对于 HTML 来说，动态代码，例如脚本元素中包含的 document.write() 方法会在源码中添加内容，也就是说，解析过程实际上会改变输入的内容

解析结束之后，浏览器开始加载网页的外部资源（CSS，图像，Javascript 文件等）。此时浏览器把文档标记为可交互的（interactive），浏览器开始解析处于“推迟（deferred）”模式的脚本，也就是那些需要在文档解析完毕之后再执行的脚本。之后文档的状态会变为“完成（complete）”，浏览器会触发“加载（load）”事件。
注意解析 HTML 网页时永远不会出现“无效语法（Invalid Syntax）”错误，浏览器会修复所有错误内容，然后继续解析。
解析过程中，如果是遇到要加载外部css文件，浏览器会另外发出一个请求，来获取css文件。遇到图片资源，浏览器也会另外发出一个请求，来获取图片资源。这是异步请求，并不会影响html文档进行加载。但是当文档加载过程中遇到js文件，html文档会挂起渲染（加载解析渲染同步）的线程，不仅要等待文档中js文件加载完毕，还要等待解析执行完毕，才可以恢复html文档的渲染线程。
### 4.3.2 解析 CSS
- 根据 [CSS词法和句法](https://www.w3.org/TR/CSS2/grammar.html) 分析CSS文件和 style 标签包含的内容以及 style 属性的值
- 每个CSS文件都被解析成一个样式表对象（StyleSheet object），这个对象里包含了带有选择器的CSS规则，和对应CSS语法的对象
- CSS解析器可能是自顶向下的，也可能是使用解析器生成器生成的自底向上的解析器

### 4.3.3 解析 JS
浏览器UI线程：单线程，大多数浏览器（比如chrome）让一个单线程共用于执行javascrip和更新用户界面。
js阻塞页面：浏览器里的http请求被阻塞一般都是由js所引起，具体原因是js文件在下载完毕之后会立即执行，而js执行时候会阻塞浏览器的其他行为，有一段时间是没有网络请求被处理的，这段时间过后http请求才会接着执行，这段空闲时间就是所谓的http请求被阻塞。
js阻塞原因：之所以会阻塞UI线程的执行，是因为js能控制UI的展示，而页面加载的规则是要顺序执行，所以在碰到js代码时候UI线程会首先执行它。
## 4.4 页面渲染
- 通过遍历DOM节点树创建一个“Frame 树”或“渲染树”，并计算每个节点的各个CSS样式值
- 通过累加子节点的宽度，该节点的水平内边距(padding)、边框(border)和外边距(margin)，自底向上的计算"Frame 树"中每个节点的首选(preferred)宽度
- 通过自顶向下的给每个节点的子节点分配可行宽度，计算每个节点的实际宽度
- 通过应用文字折行、累加子节点的高度和此节点的内边距(padding)、边框(border)和外边距(margin)，自底向上的计算每个节点的高度
- 使用上面的计算结果构建每个节点的坐标
- 当存在元素使用 floated，位置有 absolutely 或 relatively 属性的时候，会有更多复杂的计算，详见http://dev.w3.org/csswg/css2/ 和 http://www.w3.org/Style/CSS/current-work
- 创建layer(层)来表示页面中的哪些部分可以成组的被绘制，而不用被重新栅格化处理。每个帧对象都被分配给一个层
- 页面上的每个层都被分配了Textures
- 每个层的帧对象都会被遍历，计算机执行绘图命令绘制各个层，此过程可能由CPU执行栅格化处理，或者直接通过D2D/SkiaGL在GPU上绘制
- 上面所有步骤都可能利用到最近一次页面渲染时计算出来的各个值，这样可以减少不少计算量
- 计算出各个层的最终位置，一组命令由 Direct3D/OpenGL发出，GPU命令缓冲区清空，命令传至GPU并异步渲染，帧被送到Window Server。

## 4.5 GPU 渲染
- 在渲染过程中，图形处理层可能使用通用用途的 CPU，也可能使用图形处理器 GPU
- 当使用 GPU 用于图形渲染时，图形驱动软件会把任务分成多个部分，这样可以充分利用 GPU 强大的并行计算能力，用于在渲染过程中进行大量的浮点计算。


所以总结起来就是： 解析HTML以构建DOM树 –> 构建渲染树 –> 布局渲染树 –> 绘制渲染树。
DOM树是由HTML文件中的标签排列组成，渲染树是在DOM树中加入CSS或HTML中的style样式而形成。渲染树只包含需要显示在页面中的DOM元素，像 head 元素或display属性值为none的元素都不在渲染树中。
在浏览器还没接收到完整的HTML文件时，它就开始渲染页面了，在遇到外部链入的脚本标签或样式标签或图片时，会再次发送HTTP请求重复上述的步骤。在收到CSS文件后会对已经渲染的页面重新渲染。加入它们应有的样式，图片文件加载完立刻显示在相应位置。在这一过程中可能会触发页面的重绘或重排。

# 5 断开 TCP 连接，并将资源进行本地缓存
## 5.1 断开TCP 连接
当资源通过TCP通道传输过来的时候，浏览器使用完之后就要断开通道了。
这时候就是四次挥手：
![1](what-happens-when-you-navigate-to-a-url/16.png)
1. 客户端进程发出连接释放报文，并且停止发送数据。释放数据报文首部，FIN=1，其序列号为seq=u（等于前面已经传送过来的数据的最后一个字节的序号加1），此时，客户端进入FIN-WAIT-1（终止等待1）状态。 TCP规定，FIN报文段即使不携带数据，也要消耗一个序号。
2. 服务器收到连接释放报文，发出确认报文，ACK=1，ack=u+1，并且带上自己的序列号seq=v，此时，服务端就进入了CLOSE-WAIT（关闭等待）状态。TCP服务器通知高层的应用进程，客户端向服务器的方向就释放了，这时候处于半关闭状态，即客户端已经没有数据要发送了，但是服务器若发送数据，客户端依然要接受。这个状态还要持续一段时间，也就是整个CLOSE-WAIT状态持续的时间。
3. 客户端收到服务器的确认请求后，此时，客户端就进入FIN-WAIT-2（终止等待2）状态，等待服务器发送连接释放报文（在这之前还需要接受服务器发送的最后的数据）。
4. 服务器将最后的数据发送完毕后，就向客户端发送连接释放报文，FIN=1，ack=u+1，由于在半关闭状态，服务器很可能又发送了一些数据，假定此时的序列号为seq=w，此时，服务器就进入了LAST-ACK（最后确认）状态，等待客户端的确认。
5. 客户端收到服务器的连接释放报文后，必须发出确认，ACK=1，ack=w+1，而自己的序列号是seq=u+1，此时，客户端就进入了TIME-WAIT（时间等待）状态。注意此时TCP连接还没有释放，必须经过2?MSL（最长报文段寿命）的时间后，当客户端撤销相应的TCB后，才进入CLOSED状态。
6. 服务器只要收到了客户端发出的确认，立即进入CLOSED状态。同样，撤销TCB后，就结束了这次的TCP连接。可以看到，服务器结束TCP连接的时间要比客户端早一些。

## 5.2 进行本地缓存
一个页面一般都要很多资源文件组成（HTML,CSS,JS, 图片）, 当用 TCP 请求加载下来之后。一般浏览器将这些资源用完之后，都会存在内存里面一段时间，这样子下次访问的时候，就不用重新请求了，直接返回 200 from memory cache。
如果响应头部有设置 Cache-Control max-age 的时候(一般都是一些静态资源，比如 css， 图片，js ，才会设置)，那么浏览器就会将这些资源存到硬盘里面。 如果缓存时间没有过期的时候（即max-age 或者 expires 还有效），这时候就会直接返回 200 from disk cache。
如果缓存期过了，但是本地还有缓存的话，这时候就会请求服务端，然后带上一些协商缓存的头部，比如 If-None-Match 或 If-Since-Modified， 如果服务器发现资源没有变的话，返回 304的话，这时候浏览器就继续用本地的缓存，并刷新 max-age 和 expire 头部。
对于一些动态脚本，比如 .php 之内的，就不会进行缓存了。

---

以上就是我所理解的整个过程，当然整个过程还可以再细化。但是我觉得这样子也差不多够了。
以下是一些参考资料：
[What-happens-when 的中文翻译](https://github.com/skyline75489/what-happens-when-zh_CN)
[What really happens when you navigate to a URL]([]http://igoro.com/archive/what-really-happens-when-you-navigate-to-a-url/)
[TCP的三次握手与四次挥手](https://blog.csdn.net/qzcsu/article/details/72861891)
[SSL/TLS 握手过程详解](https://www.jianshu.com/p/7158568e4867)
[一次完整的HTTP事务是怎样一个过程？](http://blog.51cto.com/linux5588/1351007)