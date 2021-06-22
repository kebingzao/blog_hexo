---
title: 记一次 aws s3 公开桶的索引泄露问题
date: 2021-06-21 17:41:19
tags: 
- cloudfront
- aws
- s3
- security
categories: web安全
---
## 前言
之前有个好心的白帽子有发了一个邮件过来，说我们有一个 aws s3 的桶有索引泄露的安全缺陷，让我们赶紧处理一下。

吓的我赶紧试了一下，按照他的 POC 截图来操作，结果还真是泄露了索引:
```text
$ aws s3 ls s3://someCdnBucket
2015-08-15 11:01:02     276216 xxx.js
2015-09-01 20:26:19     101945 xxx2.js
2017-03-10 09:15:02      31751 xxx.css
...
```

确实可以通过 s3 的 cli2 的命令行工具，然后随便注册一个私人的 aws 账号，就可以通过 `aws s3 ls s3://{bucketName}` 来得到我这个项目的 bucket 里面的所有的文件索引了。

这个确实很严重， 因为如果知道了文件索引， 意味着这个公开桶的所有资源都可以被下载下来， 不幸中的万幸就是， 这个 bucket 是一个作为静态 CDN 源的公开桶， 只需要知道文件的 url 就可以访问， 里面的文件并没有用户的隐私数据。

而且这个 researcher 还表示了一个担忧， 既然我可以下载这些文件， 那么我可以上传或者从这个 bucket 删除文件吗?? 带着他这个疑问， 我这边也开始了排查。
<!--more-->
## 排查
当然排查之前，我们要先安装 aws cli 工具并配置

### 安装 aws cli 工具并配置
我这边是直接安装最新的 AWS CLI 的 版本 2: [在 Windows 上安装、更新和卸载 AWS CLI 版本 2](https://docs.aws.amazon.com/zh_cn/cli/latest/userguide/install-cliv2-windows.html)

安装完，要执行 `aws configure` 进行账号配置， 需要注意的是， 这个配置不能是 我们项目 的 aws 账号，不然根本就有权限了， 根本说明不了问题。 所以配置了一个我早期私人测试的 aws 账号，并且为了测试，创建了一个 bucket 为 `test-xxx`, 并且上传了一张为 `曲线1.png` 的图片。

执行一下对应的指令:
```text
aws s3 ls  s3://test-xxx
```
可以正常显示这个 aws 下面的所有的 bucket，并且可以列出这个 bucket 的文件索引

![1](1.png)

接下来我们试下能不能用这个私人的 aws 账号去显示这个 `someCdnBucket` bucket 的文件索引。
```text
$ aws s3 ls s3://someCdnBucket
2015-08-15 11:01:02     276216 xxx.js
2015-09-01 20:26:19     101945 xxx2.js
2017-03-10 09:15:02      31751 xxx.css
...
```

事实证明，是可以的， 这个 researcher 的操作没错， 任何一个 aws 账号都可以拉取这个 bucket 的文件索引， 从而将这个 bucket 都拷贝下来。

说明文件索引的访问权限肯定是开放的。 那么其他的权限呢， 比如 删除权限， 上传权限？？ 测一下

```text
admin@admin-PC MINGW64 /d/Program Files/Amazon/AWSCLIV2
$ ./aws.exe s3api delete-object --bucket someCdnBucket --key xxx.js

An error occurred (AccessDenied) when calling the DeleteObject operation: Access Denied
```
```text
admin@admin-PC MINGW64 /d/Program Files/Amazon/AWSCLIV2
$ ./aws.exe s3api put-object --bucket someCdnBucket --key zach1.jpg --body e:/tmp/zach1.jpg

An error occurred (AccessDenied) when calling the PutObject operation: Access Denied

```
发现删除 和 上传操作都是不行的。 很明显没有权限。 而如果换成是这个账号本来就有权限的 `test-xxx`, 那么是可以的

```text
admin@admin-PC MINGW64 /d/Program Files/Amazon/AWSCLIV2
$ ./aws.exe s3api delete-object --bucket test-xxx --key 曲线1.png

admin@admin-PC MINGW64 /d/Program Files/Amazon/AWSCLIV2
$ ./aws.exe s3 ls s3://test-xxx
```
说明删除成功

```text
admin@admin-PC MINGW64 /d/Program Files/Amazon/AWSCLIV2
$ ./aws.exe s3api put-object --bucket test-xxx --key zach1.jpg --body e:/tmp/zach1.jpg
{
    "ETag": "\"743a89a2687d7f634956844c5b6c2720\""
}

admin@admin-PC MINGW64 /d/Program Files/Amazon/AWSCLIV2
$ ./aws.exe s3 ls s3://test-xxx                                                    
2021-05-12 16:27:36      82575 zach1.jpg
```
上传成功。

所以可以肯定如果不是我们项目 的 aws 账号， 那么肯定是不会对这个 `someCdnBucket` 有写入权限的。

### 权限设置的原因
那么是什么原因导致了这个 bucket 会索引泄露了， 我们对比了另一个也是存放 cdn 静态资源的 bucket， 他就不会有这个索引泄露问题。

后面经过排查，发现是这个 s3 bucket 的访问控制列表的一个配置有问题

![1](2.png)

在所有人(公有访问权限) 这一栏的 对象这边不能是 `列出`， 而是要勾掉变成 `-`, 就是禁止。

改掉之后就可以了， 然后用 cli 请求也是正常了
```text
admin@admin-PC MINGW64 /d/Program Files/Amazon/AWSCLIV2
$ ./aws.exe s3 ls s3://someCdnBucket

An error occurred (AccessDenied) when calling the ListObjectsV2 operation: Access Denied
```

而且直接访问这个 bucket 所对应的 s3 的域名链接也是会出现 `AccessDenied`

![1](3.png)

## 总结
所以即使是设置为 公开桶的时候， 访问权限也要设置好。 而且不用的 bucket 最好就删掉， 如果这个 bucket 有对应的域名和 dns 解析的话，也要把对应的 dns 解析删掉掉， 不然就会出现子域名接管的安全问题，这个也是我之前踩过的一个坑，就是 s3 bucket 所对应的子域名被接管了，具体看 {% post_link s3-subdomain-takeover %}。




