---
title: 页面跳转，浏览器地址栏地址保持不变的几种方式
date: 2019-05-30 16:38:33
tags: 
- js
- nginx
categories: 前端相关
---
## 前言
这边总结了一下，当页面跳转的时候，如果我们想要浏览器的地址栏地址保持不变的几种方式：
- js 实现
- iframe 实现
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

## 直接用 DNS CNAME 解析
其实上面那两种，现在正常的网页设计都很少在用了。 如果真的要用 A 网站显示 B 网站的内容，但是地址栏还是 A 网站的地址。那最好的方式就是将 A 网站的域名指向 B 网站的域名。
举个例子，其实现在很多的 CDN 都是这么用的，比如我们使用了 AWS 的 S3 服务，将我们的代码上传到 S3 的bucket， 然后我们使用了 AWS 的 CDN 服务（cloudfront 服务），这时候就会生成一个域名，域名是长这样子的：**d3jnxxxxxgltl.cloudfront.net**， 事实上这个域名也是可以配置成可以直接访问的，但是我们一般不怎么做，而是更愿意换成我们自己的域名，所以这时候就要把我们自己的域名指向这个 cloudfront 域名就可以了。
![1](3.png)

