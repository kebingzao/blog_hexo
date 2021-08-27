---
title: 在进行七牛或者S3文件上传的时候，怎么保证上传文件的完整性
date: 2021-08-26 13:54:19
tags: 
- aws
- s3
- 七牛
categories: golang相关
---
## 前言
我们的业务有上传服务，不管是我们自己的文件，还是用户生成的文件， 都会传到第三方的存储云服务中。 如果是国内用户，我们用的是七牛云服务， 如果是国外用户，我们用的是 aws 的 s3 存储服务。

如果是存放用户上传文件，一般我们建立的 bucket 就是私有桶。 正常情况下如果上传成功的话，一般文件都是完整的。 但是不排除网络丢包的情况， 那么我们怎么去保证我们上传的文件，一定是完整的呢。

最简单的方法，就是在上传到云服务之前，先把本地的文件进行 md5 计算，得到一个 md5 的 hash 值。 然后上传到云服务之后， 再去云服务请求这个文件的 md5 的值，两者对比，如果一致的话，那么文件就是完整的。
<!--more-->
## 获取本地文件的 md5 值
这个就很简单了，各种语言都可以实现，就以 golang 为例，代码:
```text
func GetPackageFileHash(filePath) (string, error) {
	fileBytes,err := ioutil.ReadFile(filePath)
	if err != nil{
		return "", err
	}
	h := md5.New()
	h.Write(fileBytes)
	md5s = hex.EncodeToString(h.Sum(nil))
	return md5s, nil
}
```
这样子就可以了。

## 获取七牛的文件的 md5 值
七牛也有接口可以获取已上传文件的 md5 值: [文件HASH值（qhash）](https://developer.qiniu.com/dora/1297/file-hash-value-qhash)

> 他这边有地区限制, 该功能目前支持华东、华南、华北、北美、东南亚区域的存储 bucket

简单的来说，就是在原来的资源的连接 url 最后再补上 `?qhash/md5` 字串，这样子就会返回该文件的 md5 值了。
```text
http://dn-odum9helk.qbox.me/resource/gogopher.jpg?qhash/md5

{
    "hash": "6085d57b5ee06ac7f793fa5f0f052cb5",
    "fsize": 214513
}
```

不过要注意的是，该方式仅适用于公开桶， 不适用于私有桶， 私有桶的文件的 md5 值的请求，就跟我们请求私有桶的下载连接一样，也要对整个请求进行签名才行。 只不过这时候的资源文件的名称就会由 `gogopher.jpg` 变成 `gogopher.jpg?qhash/md5`, 而本例就是用的私有桶，用的就是这种方式。

![](1.png)

搞笑的是，关于私有桶怎么请求 md5 值，他的文档竟然都没有提到， 还是我提交工单给他们， 他们工程师才说的。

而且之前在处理这个事情的时候， 他们也有开放一个接口来获取资源的元信息: [资源元信息查询](https://developer.qiniu.com/kodo/1308/stat), 上面文档返回值有 `md5`， 但是我用了他们的 golang 的 SDK 里面的方式， 发现响应返回的结构体， 只有一些必须返回的，比如 hash, fsize 等， 并没有 md5 值
```text
type Entry struct {
	Hash     string `json:"hash"`
	Fsize    int64  `json:"fsize"`
	PutTime  int64  `json:"putTime"`
	MimeType string `json:"mimeType"`
	Customer string `json:"customer"`
}
```
也不知道是不是我使用方式不当，反正我请求下来， 是没有看到 md5 值。 所以就用了上面第一种方式来得到 md5 值。

## 获取 S3 的文件的 md5 值
关于怎么保证 S3 上传文件的完整性的话，他们是有官方文档的: [如何检查已上载到 Amazon S3 的对象的完整性？](https://aws.amazon.com/cn/premiumsupport/knowledge-center/data-integrity-s3/), 不过他们的处理方式是在上传的时候， 先计算文件的 md5 值，然后在上传的时候，将 `Content-MD5` 当做 header 头部带过去， 上传的时候， Amazon S3 根据提供的 Content-MD5 值检查对象。如果值不匹配，则会收到错误消息。

这个是在上传结算的时候， 由 s3 那边来判断是否完整。 其实是可以达到我们的需求的。 但是为了跟七牛的完整性校验保持一致。 还是采用了上传完成之后，再获取 md5 值进行匹配的方式来处理。

如果要获取资源的 md5 的话，S3 也有提供一个接口，可以获取这个资源的 etag: [HeadObject](https://docs.aws.amazon.com/AmazonS3/latest/API/API_HeadObject.html), 正常情况下，这个 etag 就是这个文件的 md5 值。 但是在某些情况下， 他并不等于文件的 md5 值.

{% blockquote Etag https://docs.aws.amazon.com/AmazonS3/latest/API/RESTCommonResponseHeaders.html %}
实体标签表示对象的特定版本。 ETag 仅反映对象内容的更改，而不反映其元数据。 ETag 可能是也可能不是对象数据的 MD5 摘要。它是否取决于对象是如何创建的以及它是如何加密的，如下所述：

通过 AWS 管理控制台或通过 PUT 对象、POST 对象或复制操作创建的对象：
- 由 SSE-S3 或明文加密的对象具有作为其数据的 MD5 摘要的 ETag。
- 由 SSE-C 或 SSE-KMS 加密的对象具有不是其对象数据的 MD5 摘要的 ETag。

无论加密方法如何，由分段上传或分段复制操作创建的对象都具有不是 MD5 摘要的 ETag。
{% endblockquote %}

简单的来说，正常情况下，ETag 应该是跟资源的 md5 一致。 但是不同的加密上传会导致有可能 ETag 会不一致。 尤其是分段上传肯定是不一致的。 而且我也试过了， 直接在 AWS 管理控制台上传的资源算出来的 ETag 也跟资源本身的 md5 也不一致。

不过对于我们的业务来说，不是分段上传， 加密对象也不是 SSE-C 或 SSE-KMS， 所以 ETag 和 md5 是一致 。 所以我们的场景是可以用这个方法来得到 ETag，然后校验的。

## 举个栗子
接下来写了完整的 demo 来说明一下， 我这边就直接贴代码了， 尽可能代码简介一点。 整个 demo 的文件有几个 (用 golang 写的)
```text
- main.go   入口文件
- qiniu.go  七牛上传的相关函数
- aws.go    s3 上传的 aws 权限验证
- s3.go     s3 上传的相关函数
- utils.go  工具方法
- text.txt  demo 中用来上传的实例文件
```

结构非常简单清晰，而且用 golang 来写可以让代码巨简洁，没有一行是多余的。 接下来开始介绍各个文件。

### 工具方法文件 utils.go

```text
package main

import (
	"fmt"
	"io/ioutil"
	"crypto/md5"
	"encoding/hex"
	"time"
	"net"
	"net/http"
)

const (
	DefaultResponseTimeout  = 30
	DefaultKeepAliveTimeout = 30
)

func log (msg string) {
	fmt.Println(msg + "\n")
}

// 获取文件的 md5
func Md5bytes(bytes []byte) (md5s string) {
	h := md5.New()
	h.Write(bytes)
	md5s = hex.EncodeToString(h.Sum(nil))
	return
}
// http 请求
func HttpGet(url string) ([]byte, error) {
	var body []byte
	timeout := time.Duration(DefaultResponseTimeout) * time.Second
	keepalive := time.Duration(DefaultKeepAliveTimeout) * time.Second

	transport := &http.Transport{
		ResponseHeaderTimeout: timeout,
		Dial: func(network, addr string) (net.Conn, error) {
			dialer := net.Dialer{
				Timeout:   timeout,
				KeepAlive: keepalive,
			}
			return dialer.Dial(network, addr)
		},
		DisableKeepAlives: true,
	}

	client := &http.Client{
		Transport: transport,
	}

	resp, err := client.Get(url)
	if err != nil {
		return body, err
	}
	defer resp.Body.Close()

	body, err = ioutil.ReadAll(resp.Body)
	if err != nil {
		return body, err
	}

	return body, nil
}

// 获取文件的 md5 值
func GetFileMd5Hash(fileBytes []byte) (string) {
	hash := Md5bytes(fileBytes)
	log(fmt.Sprintf("get file hash: %v", hash))
	return hash
}
```
封装了几个等下要用到的方法。 不再多说，注释写的也很清楚。

### 七牛上传文件 qiniu.go
```text
package main

import (
	"bytes"
	"fmt"
	"io/ioutil"
	"github.com/qiniu/api.v6/conf"
	qio "github.com/qiniu/api.v6/io"
	"github.com/qiniu/api.v6/rs"
	"github.com/qiniu/api.v6/url"
	"encoding/json"
	"errors"
)

const (
	// 七牛上传的有效期， 设置为一个小时
	QiniuUploadExpireTime = 3600
	// 七牛的 bucket 名称
	QiniuBucket = "test-example"
	// 这个 bucket 的自定义域名，生成下载地址，或者请求 md5 值的时候，会用到
	QiniuDomain = "https://test-example-qn.foo.com/"
	QiniuAccessKey = "_gDUs_24.......................ABw_Uo"
	QiniuSecretKey = "Zs-........................-ar-SQk"
)

type QiNiuHandler struct {
	PutPolicy rs.PutPolicy
	Name      string
	rsCli     rs.Client
	DoMain    string
}

// QiniuInit 七牛初始化
func QiniuInit() *QiNiuHandler {
	return newQiNiu()
}

func newQiNiu() *QiNiuHandler {
	conf.ACCESS_KEY = QiniuAccessKey
	conf.SECRET_KEY = QiniuSecretKey

	QiNiuUpload := &QiNiuHandler{}
	name := QiniuBucket
	QiNiuUpload.PutPolicy = rs.PutPolicy{
		Scope: name,
	}
	QiNiuUpload.Name = name
	QiNiuUpload.rsCli = rs.New(nil)
	QiNiuUpload.DoMain = QiniuDomain

	return QiNiuUpload
}

func (q *QiNiuHandler) UploadFile(fileName string, file []byte) (url string, err error) {
	log(fmt.Sprintf("start upload qiniu : %v", fileName))

	b := ioutil.NopCloser(bytes.NewReader(file))
	var ret qio.PutRet
	var extra = &qio.PutExtra{}
	// 上传之前先判断是否文件名已经存在，如果已经存在，那么就删掉
	err = q.Delete(fileName)
	if err != nil {
		log(fmt.Sprintf("q.Delete failed: %v", err.Error()))
	}

	uptoken := q.PutPolicy.Token(nil)
	log(fmt.Sprintf("qiniu upload name : %v, token : %v", fileName, uptoken))
	err = qio.Put(nil, &ret, uptoken, fileName, b, extra)
	if err != nil {
		//上传产生错误
		log(fmt.Sprintf("upload qiniu failed: %v", err.Error()))
		return url, err
	}
	//上传成功，处理返回值
	url = GenDownloadUrl(q.DoMain, fileName, QiniuUploadExpireTime, true)
	log(fmt.Sprintf("end upload qiniu : %v", fileName))
	return url, nil
}

//删除文件
func (q *QiNiuHandler) Delete(fileName string) error {
	err := q.rsCli.Delete(nil, q.Name, fileName)
	if err != nil {
		return err
	}
	return nil
}

//获取文件的元信息，比如 hash 值 (这个方法暂时本项目用不到)
// https://developer.qiniu.com/kodo/1308/stat
func (q *QiNiuHandler) Stat(fileName string) (rs.Entry, error) {
	eInfo, err := q.rsCli.Stat(nil, q.Name, fileName)
	if err != nil {
		return  eInfo, err
	}
	log(fmt.Sprintf("get qiniu file stat : %v, info: %v", fileName, eInfo))
	return eInfo, nil
}
// 获取文件的 md5 值
func (q *QiNiuHandler) GetFileHash(fileName string) (string, error) {
	// https://developer.qiniu.com/dora/1297/file-hash-value-qhash
	// 不过因为是私有空间，所以要访问的话，还得再进行签名
	log(fmt.Sprintf("start get qiniu hash  %v", fileName))
	fileName = fmt.Sprintf("%v?qhash/md5", fileName)
	// 如果要访问私有空间的 md5 值，那么就要用下载的那个 token 来加密链接，然后请求
	url := GenDownloadUrl(q.DoMain, fileName, QiniuUploadExpireTime, false)
	body, err := HttpGet(url)
	if err != nil {
		log(fmt.Sprintf("get qiniu md5 fail, request fail: %v:", url))
		return "", nil
	}
	result := make(map[string]interface{})
	err = json.Unmarshal([]byte(body), &result)
	if err != nil {
		log(fmt.Sprintf("get qiniu md5 fail: %v json %v:", url, string(body)))
	}
	log(fmt.Sprintf("url: %v, result: %v", url, result))
	// 如果文档不存在的话，就会为空
	if result["hash"] == nil {
		return "", errors.New(result["error"].(string))
	}
	if result["hash"].(string) == "" {
		log(fmt.Sprintf("get qiniu md5 fail: %v not find hash param %v:", url, string(body)))
	}
	hash := result["hash"].(string)
	log(fmt.Sprintf("get qiniu hash success, %v: %v", url, hash))
	return hash, nil
}
// 生成下载的链接，主要是生成 token
func GenDownloadUrl(domain string, key string, expire uint32, needEscape bool) string {
	urlStr := domain + url.Escape(key)
	if !needEscape {
		urlStr = domain + key
	}
	policy := rs.GetPolicy{}
	policy.Expires = expire
	return policy.MakeRequest(urlStr, nil)
}
```

注释也写的很清楚， 就是上传和获取 md5，直接用他们的 sdk 方法就可以搞定了。 也不多说

### s3 上传文件 aws.go 和 s3.go
因为 s3 是属于 aws 的其中一个服务， 而 aws 的 accessKey 和 secretKey 是适用于所有的 aws 产品的，不仅仅是 s3。 所以这边将权限初始化分开，会显得架构更加的清晰。

首先是 aws.go 就是权限初始化:
```text
package main

import (
	"github.com/crowdmob/goamz/aws"
)

type Amzs struct {
	Auth aws.Auth
}

const (
	AWSAccessKey= "ABBBBBBBUAAAAAAAAA"
	AWSSecretKey= "Ie..................7sux4z+FtH"
)

var (
	Amz       *Amzs
	AmzRegoin map[string]aws.Region = map[string]aws.Region{
		"USEast":  aws.USEast,
		"USWest":  aws.USWest,
		"USWest2": aws.USWest2,
	}
)

func AmzInit() {
	Amz = newAmzs()
}

func newAmzs() *Amzs {
	amz := &Amzs{
		Auth: aws.Auth{AccessKey: AWSAccessKey, SecretKey: AWSSecretKey},
	}
	return amz
}

func getRegion(region string) aws.Region {
	var r aws.Region
	var ok bool

	if r, ok = AmzRegoin[region]; !ok {
		r = AmzRegoin["USWest"]
	}
	return r
}
```
他有很多个区域， 一般我们的bucket， 要么是在美东，要么是在美西， 不需要所有的 region 都列出来。

接下来是 s3.go:
```text
package main

import (
	"github.com/crowdmob/goamz/aws"
	"github.com/crowdmob/goamz/s3"
	"time"
	"strings"
	"fmt"
)

const (
	S3Bucket = "test-s3"
	S3BucketAcl= "public-read"
	S3ConnectTimeout= 15
	S3ReadTimeout= 240
	// 本次的 bucket 是在美东区域
	S3BucketRegion = "USEast"
)

type S3Handler struct {
	Bucket *s3.Bucket
	Name   string
	Region aws.Region
	Acl    s3.ACL
}

func S3Init() *S3Handler {
	// 先初始化 aws 的权限设置
	AmzInit()
	// 然后再初始化 s3 的配置
	return newS3()
}

func newS3() *S3Handler {
	region := getRegion(S3BucketRegion)
	sn := s3.New(Amz.Auth, region)
	sn.ConnectTimeout = time.Duration(S3ConnectTimeout) * time.Second
	sn.ReadTimeout = time.Duration(S3ReadTimeout) * time.Second

	s := &S3Handler{
		Name:   S3Bucket,
		Acl:    s3.ACL(S3BucketAcl),
		Region: region,
		Bucket: sn.Bucket(S3Bucket),
	}
	return s
}
// 上传文件
func (m *S3Handler) UploadFile(fileName string, data []byte) (url string, err error) {
	log(fmt.Sprintf("start upload s3 : %v", fileName))
	contType := "application/octet-stream"
	option := &s3.Options{}
	err = m.Bucket.Put(fileName, data, contType, m.Acl, *option)
	if err != nil {
		log(fmt.Sprintf("upload s3 failed, file: %v, error : %v", fileName, err.Error()))
		return url, err
	}
	url = m.Bucket.URL(fileName)
	log(fmt.Sprintf("end upload s3 : %v", fileName))
	return url, nil
}
// 获取 s3 文件的 meta 资源数据， 直接获取 header 的数据
func (m *S3Handler) GetFileMetaData(fileName string) (map[string]interface{}, error) {
	log(fmt.Sprintf("get s3 meta : %v", fileName))
	resp, err := m.Bucket.Head(fileName, nil)
	metaMap := make(map[string]interface{})
	if err != nil {
		log(fmt.Sprintf("get s3  meta failed, file: %v, error : %v", fileName, err.Error()))
		return metaMap, err
	}
	for k, v := range resp.Header {
		metaMap[k] = v
	}
	log(fmt.Sprintf("meta: %v, e-tag: %v", metaMap, metaMap["Etag"]))
	return metaMap, nil
}

// 获取文件的 md5 值, 这里其实就是 etag
// 这边要特别注意的是， 通过 s3 上传的文件，并不是所有文件的 md5 值都是跟 etag 一致的。
// 除了多块上传的时候， etag 肯定是跟源文件的 md5 不一致的情况， 在普通单文件上传时候，采取不同的加密方式，也会导致有时候 etag 会跟实际文件的 md5 不一样。
// 比如我就试过直接在 AWS Management Console 直接上传文件的时候， 他的 etag 就不是源文件的 md5 值。
// 不过对于本项目来说， 因为是通过 PUT 的方式来进行单文件上传的，并且也没有设置其他的加密方式，所以刚好 etag 跟源文件的 md5 值一样。
// 具体关于什么时候 etag 会跟 md5 不一致的文档： https://docs.aws.amazon.com/AmazonS3/latest/API/RESTCommonResponseHeaders.html
// 还有一种校验 s3 上传文件完整性的情况，就是在上传的时候， 在头部传入 Content-MD5 的值，这个值是我们自己算的，算法： https://aws.amazon.com/cn/premiumsupport/knowledge-center/data-integrity-s3/ ， 通过这种方式也可以判断上传文件的完整性， 不过为了保证跟七牛的体验一致，还是采用上传后 md5 校验的方式来进行判断。
func (m *S3Handler) GetFileHash(fileName string) (string, error) {
	log(fmt.Sprintf("start get s3 hash  %v", fileName))
	metaMap, err := m.GetFileMetaData(fileName)
	hash := ""
	if err != nil {
		log(fmt.Sprintf("get s3 hash failed, file: %v, error : %v", fileName, err.Error()))
		return "", err
	}
	if metaMap["Etag"] != nil {
		hash = metaMap["Etag"].([]string)[0]
		// 替换掉 双引号
		hash = strings.Replace(hash, `"`, ``, -1)
	}
	log(fmt.Sprintf("get s3 hash success, %v: %v", fileName, hash))
	return hash, nil
}
```
注释也很清楚， 不再多说。

### 最后执行的 main 文件
```text
package main

import (
	"io/ioutil"
	"fmt"
)

const (
	UploadFileName = "test.txt"
)

type UploadHandler interface {
	UploadFile(FileName string, file []byte) (downloadUrl string, err error)
	GetFileHash(FileName string) (string, error)
}

func main() {
	fmt.Println("start===")
	testQiNiu()
	testS3()
}
// 测试七牛上传
func testQiNiu (){
	uploadClient := QiniuInit()
	commonHandle(uploadClient)
}
// 测试 s3 上传
func testS3 (){
	uploadClient := S3Init()
	commonHandle(uploadClient)
}
// 通用方法处理
func commonHandle (uploadClient UploadHandler){
	// read file
	b, err := ioutil.ReadFile("./" + UploadFileName)
	newName := "new-test.txt"
	originHash := GetFileMd5Hash(b)
	// 接下来上传, 可以选择直接用原文件名称，也可以重命名
	downloadUrl, err := uploadClient.UploadFile(newName, b)
	if err != nil {
		log(err.Error())
	}
	log(fmt.Sprintf("download url: %v", downloadUrl))
	// 得到 本地文件 的 md5 值
	localHash, err := uploadClient.GetFileHash(newName)
	if err != nil {
		log(err.Error())
	}
	log(fmt.Sprintf("local md5: %v, online md5: %v", originHash, localHash))
	// 接下来比较两者的 md5 值是否一致
	if originHash == localHash {
		log("file match!!!!")
	}else{
		log("file not match!!!!")
	}
}
```
做了一个 interface， 这样子就可以将通用的处理流程都抽出来。 显得代码更加的干净了。

### 跑起来看下效果
可以看到，刚开始是跑七牛上传， 然后接下来是跑 s3 上传的。 先看七牛的 log:

```text
start===
get file hash: e10adc3949ba59abbe56e057f20f883e

start upload qiniu : new-test.txt

qiniu upload name : new-test.txt, token : _gDUs_24d-itH7Kk0EFYwU5uxnEpiYSSpdABw_Uo:SSGy00RIKSMhI1ymNPRrSnUA-U0=:eyJzY29wZSI6ImFpci10ZXN0IiwiZGVhZGxpbmUiOjE2MzAwNDgzMTB9

end upload qiniu : new-test.txt

download url: https://test-example-qn.foo.com/new-test.txt?e=1630048310&token=_gDUs_24d-itH7Kk0EFYwU5uxnEpiYSSpdABw_Uo:mCz4fbDYN1SzIZa5E3gzeTzpf2U

start get qiniu hash  new-test.txt

url: https://test-example-qn.foo.com/new-test.txt?qhash/md5&e=1630048310&token=_gDUs_24d-itH7Kk0EFYwU5uxnEpiYSSpdABw_Uo:ZVL1JRqSn8zPb9DM2HodgwBCldM, result: map[fsize:6 hash:e10adc3949ba59abbe56e057f20f883e]

get qiniu hash success, https://test-example-qn.foo.com/new-test.txt?qhash/md5&e=1630048310&token=_gDUs_24d-itH7Kk0EFYwU5uxnEpiYSSpdABw_Uo:ZVL1JRqSn8zPb9DM2HodgwBCldM: e10adc3949ba59abbe56e057f20f883e

local md5: e10adc3949ba59abbe56e057f20f883e, online md5: e10adc3949ba59abbe56e057f20f883e

file match!!!!
```

log 也写的很详细了， 反正就是先上传，然后再获取 md5， 最后匹配。

然后接下来看 s3 上传的 log:

```text
get file hash: e10adc3949ba59abbe56e057f20f883e

start upload s3 : new-test.txt

end upload s3 : new-test.txt

download url: https://s3.amazonaws.com/test-s3/new-test.txt

start get s3 hash  new-test.txt

get s3 meta : new-test.txt

meta: map[Accept-Ranges:[bytes] Content-Length:[6] Content-Type:[application/octet-stream] Date:[Fri, 27 Aug 2021 06:12:01 GMT] Etag:["e10adc3949ba59abbe56e057f20f883e"] Last-Modified:[Fri, 27 Aug 2021 06:12:00 GMT] Server:[Amazo
nS3] X-Amz-Id-2:[O3OUrzoLr8Z0RbAOaLa0/DI3q+KfBJrCa87mlVg2KCbS3uwozv5nCe9Yf+gtDL9w5A97hurS6rU=] X-Amz-Request-Id:[YX6T7015025YYNZF]], e-tag: ["e10adc3949ba59abbe56e057f20f883e"]

get s3 hash success, new-test.txt: e10adc3949ba59abbe56e057f20f883e

local md5: e10adc3949ba59abbe56e057f20f883e, online md5: e10adc3949ba59abbe56e057f20f883e

file match!!!!
```

一样测试结果肯定是匹配的。





