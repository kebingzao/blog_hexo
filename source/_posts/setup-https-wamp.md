---
title: 如何在 WAMP 设置 https
date: 2019-04-10 11:51:29
tags: apache
categories: 前端相关
---
## 前言
之前有一个需求，就是要把某一个站点使用https，所以有时候需要在本地开发环境也要有https环境，因为本地是用 WAMP 搭的，所以就配了一下。 我当时的系统是 windows 7， 32 bit的。
## 创建自己定义的 ssl 证书
打开dos 窗口，定位到apache所在的bin目录：
<!--more-->
![1](1.png)
开始创建自己定义的ssl 证书：这时候会让你输入一个通行码，并确认一次，这个码后面导出的时候，会用到
````html
openssl genrsa -aes256 -out pass.key 2048
````
![1](2.png)
但是这时候弹出了一个错误：
![1](3.png)
少了一个 openssl 需要用到的动态链接库，那我们就去[这个地方](http://slproweb.com/products/Win32OpenSSL.html)下载一下 openssl 安装程序，
![1](4.png)
![1](5.png)
然后安装完以后可以从根目录把 **libeay32.dll** 和 **ssleay32.dll** 复制到 **D:\wamp\bin\apache\apache2.2.22\bin** 下：
![1](6.png)
这时候重新运行下，该命令：
![1](7.png)
发现已经可以了，并且需要输入并确认通行码，接下来把该证书导出来，并自定义证书名字，需要输入刚才定的通行码：
```html
openssl rsa -in pass.key -out test.key
```
![1](8.png)
接下来导出对应的证书crt文件，并添加具体的证书信息，比如国家，公司，服务器域名等等
```html
openssl req -new -x509 -nodes -sha1 -key test.key -out test.crt -days 999 -config C:\wamp\bin\apache\apache2.2.14\conf\openssl.cnf
```
![1](9.png)
这样子，自定义的这一对证书就有了， 就是在bin目录下的 **test.crt** 和 **test.key**
## 修改Apache配置
接下来在  **D:\wamp\bin\apache\apache2.2.22\conf**  新建一个 名为 **ssl** 的文件，存放访问https的证书和日记：
![1](10.png)
然后把刚才在bin目录下生成的那一对证书拷贝到ssl目录中，并新建一个logs的目录来存放日记文件：
![1](11.png)
接下来在这个 **D:\wamp\bin\apache\apache2.2.22\conf\extra** 目录下打开 **httpd-ssl.conf** 该文件，对一些设置进行修改，这个文件其实就是https下的配置。
![1](12.png)
1. 将
    ```html
    SSLSessionCache "shmcb:C:/Program Files/Apache Software Foundation/Apache2.2/logs/ssl_scache(512000)"
    ```
    替换成  
    ```html
    SSLSessionCache "shmcb:D:/wamp/bin/apache/apache2.2.22/conf/ssl/logs/ssl_scache(512000)"
    ```
2. 将 
    ```html
    SSLCertificateFile "C:/Program Files/Apache Software Foundation/Apache2.2/conf/server.crt"
    ```
    替换成 
    ```html
    SSLCertificateFile "D:/wamp/bin/apache/apache2.2.22/conf/ssl/test.crt"
    ```
3. 将 
    ```html
    SSLCertificateKeyFile "C:/Program Files/Apache Software Foundation/Apache2.2/conf/server.key"
    ```
    替换成
    ```html
    SSLCertificateKeyFile "D:/wamp/bin/apache/apache2.2.22/conf/ssl/test.key"
    ```
    第二第三点，其实就是证书的位置
4. 将 
    ```html
    SSLMutex "file:C:/Program Files/Apache Software Foundation/Apache2.2/conf/ssl/logs/ssl_mutex"
    ```
    替换成 
    ````html
    SSLMutex default
    ````
5. 修改虚拟主机名，将
    ```html
    DocumentRoot "C:/Program Files/Apache Software Foundation/Apache2.2/htdocs"
    TransferLog "C:/Program Files/Apache Software Foundation/Apache2.2/logs/access_log"
    ```
    替换成
    ```html
    DocumentRoot "D:/wamp/www/ssl"
    ServerName test:443
    ServerAdmin admin@example.com
    ErrorLog "D:/wamp/bin/apache/apache2.2.22/conf/ssl/logs/ssl_error.log"
    TransferLog "D:/wamp/bin/apache/apache2.2.22/conf/ssl/logs/ssl_access.log"
    ```
    ![1](13.png)
6. 将
    ```html
    <Directory "C:/Program Files/Apache Software Foundation/Apache2.2/cgi-bin">
    SSLOptions +StdEnvVars
    </Directory>
    ```
    替换成
    ```html
    <Directory "D:/wamp/www/ssl">
        SSLOptions +StdEnvVars
         Options Indexes FollowSymLinks MultiViews
         AllowOverride All
         Order allow,deny
         allow from all
    </Directory>
    ```
    ![1](14.png)
7. 将
    ```html
    CustomLog "C:/Program Files/Apache Software Foundation/Apache2.2/logs/ssl_request_log" \"%t %h %{SSL_PROTOCOL}x %{SSL_CIPHER}x \"%r\" %b"
    ```
    替换成
    ```html
    CustomLog "D:/wamp/logs/ssl_request.log" \
              "%t %h %{SSL_PROTOCOL}x %{SSL_CIPHER}x \"%r\" %b"
    ```
    ![1](15.png)

这样子 **httpd-ssl.conf** 就调整好了。接下来打开 **D:\wamp\bin\apache\apache2.2.22\conf** 的 **httpd.conf** 文件, 修改两个地方， 去掉前面的 # 号：
![1](16.png)
![1](17.png)
这样就打开了 ssl 模块了，但是这样还不够，接下来还要打开 php 模块扩展组件 **php_openssl**：
![1](18.png)
而 apache 也要打开 **ssl_module**：
![1](19.png)
然后将 **D:\wamp\bin\apache\apache2.2.22\bin** 下面的 **libeay32.dll** 和 **ssleay32.dll** 拷贝到 **C:\Windows\System32**。这样子基本上就设置就完成了。
## 测试
接下来就可以测试 https了， 因为我们设置的https的根目录是 **www\ssl**, 因此需要在 www 目录下新建一个 ssl 目录：
![1](20.png)
index.html 是一些比较简单的内容：
```html
<html>
<body>
<font size="5" color="red">test SSL successful</font>
</body>
</html>
```
最后输入 **https://test**
![1](21.png)
发现有错，没有响应，查看一下，错误日记:
![1](22.png)
发现我们当初在创建证书设置的域名跟本地服务器的域名不匹配，导致无法加载:
![1](23.png)
因此要用host指定一下:
![1](24.png)
然后再试一下:
![1](25.png)
发现成功了。
ps： 后面试了一下，发现不需要一定要 KBZ.COM 的域名，因为域名不匹配其实只是个 warn，他还不是错误。所以我可以用任何的域名都可以，比如我 host 这个域名
```html
192.168.40.33 ssl.demo.com
```
然后在 **www/ssl** 放上对应的html文件，也是可以访问的。



