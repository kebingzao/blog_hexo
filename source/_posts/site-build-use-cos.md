---
title: 站点加载速度优化，国内采用腾讯云COS存放
date: 2019-05-31 13:52:48
tags: 
- aws
- 腾讯云
categories: 
- 前端相关
---
## 前言
之前有做过官网的优化：{% post_link www-history-12 %}，我们优化了官网在国内的加载速度。接下来也想优化一下，另一个站点在国内的加载速度，因为测试也在反馈，加载速度超级慢的。
## 实作
我查看了一下，现在 router 53 该站点，比如 boo.foo.com 的配置：
![1](1.png)
<!--more-->
国外用 cloudfront 没毛病， 但是国内用网宿加速，那就坑爹了。 不仅需要首次回源，而且还经常出问题。所以坚决改成跟官网一样的方式，就是打包的时候，同时放一份在腾讯云的 cos 上。
但是相对于官网还是有一些不同，官网他把资源存放分三部：
1. html 文件放源服务器
2. 资源文件国外放s3
3. 资源文件国内放cos

但是这个站点，不需要三个地方，因为他的资源文件是没有日期命名的，每次都是生成新的hash名字，然后覆盖，所以只需要放两个地址：
1. 国外全部放s3
2. 国内全部放cos

## 模拟一个预发布来实践
因为接下来这个站点可能要发版，为了接下来实作的时候，不影响现有的用户，所以我们就模拟了一个跟线上一样的构建环境，也可以做预发布环境(pre-boo.foo.com)。包括域名，bucket都是新的，但是都是线上的配置一样的。
### aws 的配置
首先在S3上创建一个新的bucket，叫 pre-boo.foo.com，
![1](2.png)
在 cloudfront 上创建一个 cloudfront 节点，并把回源地址指向这个s3 bucket
![1](3.png)
![1](4.png)
这样 s3 就配置完了。
### 配置cos域名
在腾讯云cos上创建一个 bucket：
![1](5.png)
绑定一个新的域名
![1](6.png)
### 在router 53 配置区域分配
![1](7.png)
按照地区分发，如果是中国区，就将域名 cname 到腾讯 cos 的那个域名。如果是其他地址，就 cname 到 aws 的 cloudfront 域名。
### 重新配置 gitlab 触发 webhook
![1](8.png)
### 在Jenkins那边配置对应的触发任务
![1](9.png)
这样子基础设置就都完成了，接下来就是写构建脚本了。简单的来说，就是打包完之后，上传的时候，s3 传一份， cos 那边传一份，然后调用 api 刷新各自的缓存。
### 具体上传
其实就在原来的基础上，因为原来只传 s3。 然后在 s3 传完之后， 再把相同的文件，传到 腾讯云的 cos 就行了， 看构建log是这样的：
构建成功之后，就传到 cos：
```html
Bucket name: pre-boo-foo-com-1256079527
set_gzip: 0
Local path: /data/server/wwwroot/pre-boo.foo.com/
INFO:qcloud_cos.cos_client:put object, url=:http://pre-boo-foo-com-1256079527.cos.ap-guangzhou.myqcloud.com/index.html ,headers=:{'Cache-Control': 'max-age=31536000'}
INFO:qcloud_cos.cos_client:put object, url=:http://pre-boo-foo-com-1256079527.cos.ap-guangzhou.myqcloud.com/static/manifest.json ,headers=:{'Cache-Control': 'max-age=31536000'}
INFO:qcloud_cos.cos_client:put object, url=:http://pre-boo-foo-com-1256079527.cos.ap-guangzhou.myqcloud.com/static/img/3.a66c45d.png ,headers=:{'Cache-Control': 'max-age=31536000'}
```
成功之后，再传到 s3：
```html
upload: ../../data/server/wwwroot/pre-boo.foo.com/index.html to s3://pre-boo.foo.com/index.html
Completed 9.2 KiB/14.2 MiB (18.4 KiB/s) with 118 file(s) remaining
Completed 26.2 KiB/14.2 MiB (27.1 KiB/s) with 118 file(s) remaining
upload: ../../data/server/wwwroot/pre-boo.foo.com/static/img/3.61a4b7a.png to s3://pre-boo.foo.com/static/img/3.61a4b7a.png
```
最后再重新刷新 cloudfront 和 cos 的缓存： 注意，一定要请缓存
```html
clean cloudfront cache
{
    "Location": "https://cloudfront.amazonaws.com/2018-11-05/distribution/E3D4UOWxxxxx/invalidation/I33J3B4Exxxxx",
    "Invalidation": {
        "Id": "I33J3B4Exxxxx",
        "Status": "InProgress",
        "CreateTime": "2019-05-29T11:28:05.750Z",
        "InvalidationBatch": {
            "Paths": {
                "Quantity": 1,
                "Items": [
                    "/index.html"
                ]
            },
            "CallerReference": "cli-1559129284-289782"
        }
    }
}
clean tx-cos cache
{u'count': 2, u'task_id': u'15591292865xxxxxxx'}
```
这样就可以了，不过我们之前测试还遇到了一个情况，就是上传上去之后，发现请求的时候，报了这个错误：
![1](10.png)
从这个来看的话，应该是 cos 报的错，因为 bucket name 就是 cos 的bucket名字。 所以为了判断是cos配置的问题，还是都有的问题。
我先从 host 到 cloudfront，然后刷新看下情况，发现是正常的。 所以可以肯定是因为 cos 的配置文件。 后面查了一下，发现问题出在这边，我们把这个bucket设置为默认源站：
![1](11.png)
这样是不行的，这样会有权限的问题， 所以要改成静态资源源站:
![1](12.png)
这时候还不会有马上生效，因为 cos 本身是有cdn的， 所以还要刷新一下 cos 的cdn，然后就可以了， 恢复正常了。
## 最后修改线上的站点的配置
为了避免构建的时候，cos 的桶出问题会影响用户，所以 router 53 中国区先不改。还是先指向网宿的加速。然后等构建完之后，s3 传一份， cos 也传一份。
然后接下来测试 cloudfront 有没有问题，还是用 host 的方式：
![1](13.png)
然后改一下本地的 hosts 文件(cloudfront 对应的 ip 地址一定时间是会变的，所以当我们ping出来之后，要赶紧测试，不能等太久，不然后面有可能就切到其他域名去了)：
```html
54.230.159.28 boo.foo.com
```
测试了一下，没问题。接下来 host cos 的地址，看看正不正常：
![1](14.png)
```html
117.41.244.111 boo.foo.com
```
当然这个 cos 的 cdn 的域名，也要在 cos 那边配置好才行。
![1](15.png)
测试发现也是正常的。既然国内，国外都正常，接下来就是把 router 53 的指向改过来就行了。
![1](16.png)
## 备注
这边要注意的一点是，我们不需要再对资源文件，比如 html，js，css 再手动进行 gzip 压缩了， cdn 服务器如果判断这些资源本身是没有gzip压缩的，那么它会在传输中，对资源进行 gzip 压缩，不需要我们自己去处理。

