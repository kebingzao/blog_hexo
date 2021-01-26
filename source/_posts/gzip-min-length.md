---
title: gzip 配置 gzip_min_length 来判断 response 是否要 gzip 压缩
date: 2021-01-26 13:29:27
tags: nginx
categories: nginx相关
---
## 前言
之前服务端的 nginx 服务有配置 gzip 压缩: {% post_link nginx-gzip %}, 但是在之前有一次服务优化中， 发现同一个服务的不同接口的请求，有些 response 有 gzip 压缩，有些 response 没有 gzip 压缩 ??

![1](1.png) 

<!--more-->

![1](2.png) 

可以看到，同一个服务的不同接口，有的 response 有 gzip 压缩， 有的却没有， 这个啥原因?

## 真相
因为我们的服务有用负载均衡，比如腾讯云的 CLB 或者 aws 的 ELB 服务，刚开始以为 gzip 配置是在云平台那边配置的。 后面发现负载均衡也只是负责转发到具体的业务服务器而已，并不会进行处理。 所以 gzip 的配置肯定是在业务服务器上面配的， 所以去找了一下， 有找到 nginx 的 gzip 配置:
```php
	# Gzip Settings
	gzip on;
	gzip_disable "msie6";
	gzip_vary on;
	gzip_proxied any;
	gzip_min_length 1000;
	gzip_comp_level 4;
	gzip_buffers 16 8k;
	gzip_http_version 1.1;
	gzip_types text/plain text/css application/json application/javascript application/x-javascript text/xml application/xml application/xml+rss text/javascript image/svg+xml;
```
然后发现了一个可能有问题的配置，就是:
```php
gzip_min_length 1000;
```
这个配置 `gzip_min_length` 的参数值为正整数，单位为字节，也可用 k 表示千字节，比如写成 1024 与 1k 都可以，效果是一样的，表示当资源大于1k时才进行压缩，资源大小取响应头中的 `Content-Length` 进行比较，经测试如果响应头不存在 `Content_length` 信息，该限制参数对于这个响应包是不起作用的；另外此处参数值不建议设的太小，因为设的太小，一些本来很小的文件经过压缩后反而变大了，官网没有给出建议值，在此建议1k起，因为小于1k的也没必要压缩，并根据实际情况来调整设定。 

而它的默认值是 20 字节。 如果为 0，那就全部都压缩。

而我们设置的是 1000 字节，也就是如果请求返回的 `Content-Length` 大于 1000， 那么就会启用 gzip 压缩并返回。 如果小于 1000， 那么就会正常返回。

事实上上面第二张图之所以没有返回的时候用 gzip 压缩，就是因为 response 返回的 `Content-Length` 小于 1000， 所以不会启用 gzip 压缩返回。









