---
title: 页面跳转，浏览器地址栏地址保持不变的几种方式
date: 2019-05-30 16:38:33
tags: 
- js
- nginx
- 腾讯云
- cloudfront
categories: 前端相关
---
## 前言
这边总结了一下，当页面跳转的时候，如果我们想要浏览器的地址栏地址保持不变的几种方式：
- js 实现
- iframe 实现
- nginx 转发代理
- DNS CNAME 指向

## js 实现
原理很简单，就是通过 ajax 去请求这个站点的页面，然后使用 **document.write** 重新渲染页面。
代码如下：
<!--more-->
```html
<html>
<head>
	<title>demo</title>
</head>
<body>
	<script>
        function createXMLHttpRequest(){
            if(window.XMLHttpRequest){
                XMLHttpR = new XMLHttpRequest();
            }else if(window.ActiveXObject){
                try{
                    XMLHttpR = new ActiveXObject("Msxml2.XMLHTTP");
                }catch(e){
                    try{
                        XMLHttpR = new ActiveXObject("Microsoft.XMLHTTP");
                    }catch(e){
                    }
                }
            }
        }
        function sendRequest(url){
            createXMLHttpRequest();
            XMLHttpR.open("GET",url,true);
            XMLHttpR.onreadystatechange = processResponse;
            XMLHttpR.send(null);
        }
        function processResponse(){
            if(XMLHttpR.readyState === 4 && XMLHttpR.status === 200){
                document.write(XMLHttpR.responseText);
            }
        }
        sendRequest("http://test.example.com");
	</script>
</body>
</html>
```
但是这种方法要注意一个问题，会有跨域问题：
![1](1.png)
所以我们要去 nginx (我们用的 web server) 那边添加 CORS 的跨域头部：
```html
location / {
    add_header Access-Control-Allow-Origin '*';
    add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
    add_header Access-Control-Allow-Headers 'origin, content-type';
}
```
注意这边直接 origin 为 * ，后面可以设置具体的域名。 这样就可以访问了
![1](2.png)
这样就把站点的首页拉下来了。不过这种方式有几个不好的方法：
1. 因为有跨域，所以站点的页面要开启 CORS，这样子如果对来源不做限制的话，其实是不安全的。
2. 因为用了document.write对页面进行了重写，所以对 SEO 非常不友好。
3. 如果这个页面有外链的话，如果是相对路径的话，跳转会因为找不到而报错。

## iframe 的方式实现
这个之前就很常见了，直接看代码：
```html
<iframe id="frameOne" name="frameOne" frameborder="0" width="100%" scrolling="auto" height="100%" src="http://www.example.com">
</iframe>
```
用 iframe 的缺点如下：
1. 也有跨域的问题
2. 对 SEO 也不友好
3. iframe 销毁是无法全部释放内存的，频繁创建销毁会导致内存溢出
4. iframe 会阻塞主页面的加载，所以会感觉到页面加载变慢了
5. 目标页面不能有**X-Frame-Options** 这个头部，不然 iframe 会加载不了， 具体：{% post_link web-forbidden-iframe-embed %}

## nginx 实现反向代理
还有一种方式，就是将请求代理转发到另一个域名：
```html
location / {
         proxy_set_header Host www.baidu.com;
         proxy_set_header X-Real-IP $remote_addr;
         proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
         proxy_pass https://www.baidu.com;
}
```
这样就可以将这个站点，代理转发到百度了。
![1](4.png)

## DNS CNAME 解析
无论是 js 还是 iframe 的实现，现在正常的网页设计都很少在用了。 如果真的要用 A 网站显示 B 网站的内容，但是地址栏还是 A 网站的地址。那最好的方式就是将 A 网站的域名指向 B 网站的域名。
举个例子，其实现在很多的 CDN 都是这么用的，比如我们使用了 AWS 的 S3 服务，将我们的代码上传到 S3 的bucket， 然后我们使用了 AWS 的 CDN 服务（cloudfront 服务），这时候就会生成一个域名，域名是长这样子的：**d3jnxxxxxgltl.cloudfront.net**， 事实上这个域名也是可以配置成可以直接访问的，但是我们一般不怎么做，而是更愿意换成我们自己的域名，所以这时候就要把我们自己的域名指向这个 cloudfront 域名就可以了。
![1](3.png)

### 实操
目标就是 foo.com 上显示 boo.com 的内容。但是地址栏上还是 foo.com。
难点： boo.com 有加速，而且国内用 腾讯云 cos 加速， 国外用 cloudfront 加速。
所以要分两步：
#### 腾讯云 cos 操作
1. 登入腾讯云后台
2. 进入 boo.com 所在的 bucket 目录的配置
3. 打开 **域名管理** 这个页面
4. 到下面的 **自定义加速域名** 点击 **添加域名**
![1](5.png)
5. 输入 foo.com，然后会生成一个 新的加速的站点，比如 **foo.com.cdn.dnsv1.com**
![1](6.png)
6. 接下来再把这个域名给 foo.com 那边的运营人员，让他们在 DNS 后台 (比如 aws 的 router 53)， 将 **foo.com** 指向 **foo.com.cdn.dnsv1.com**
![1](7.png)

不过需要注意一点的是：腾讯云 cos 这边的配置也是需要条件的，首先这个域名（比如 本例的 foo.com ）要在国内有备案, 而且如果有使用加速，或者要使用 https 服务的话，那么是要在我们这边上传 foo.com 的 https 证书的（所以他们要把他们的 https 的证书给我们）。
#### aws cloudfront 操作
cloudfront 的原理跟 cos 差不多，也是生成一个对应加速的地址。也就是同一个 bucket，然后再生成一个对应域名的 cloudfront：
![1](8.png)
也就是同一个 bucket 可以生成两个不同的cloudfront 加速域名，然后把第二个给对方，然后让对方的运营人员去进行 CNAME 指向：
![1](9.png)

### CNAME 实操总结
这样子就可以实现通过 CNAME 的方式，来使得对方的域名可以显示你的站点了。 而且还可以实现 CDN 的加速效果。 相比之下 nginx 转发虽然也可以实现，但是多绕了一层，而且还不能加速。

## 总结
最优结果当然是用 CNAME 的方式!!！
