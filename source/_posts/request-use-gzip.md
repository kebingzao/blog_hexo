---
title: http 请求的时候使用 gzip 压缩来减少流量消耗
date: 2021-02-03 11:57:39
tags: nginx
categories: nginx相关
---
## 前言
我们知道一般无论是 CDN 还是 服务端的接口({% post_link nginx-gzip %})，在响应返回的时候，都会配置 gzip 来压缩响应数据，以达到减少体积，加快网络传输的效能。 但是 gzip 压缩只用于 response 的响应数据，一般来说针对 pc 端的站点来说，足够了。

但是有时候针对手机 app 来说，光是 response 用 gzip 压缩还不够， 因为 request 请求的时候，如果有时候 body 体带上很多数据的话，还是会导致体积很大，从而消耗流量。 手机的流量可比 pc 的流量贵多了。 因此对于有些比较注重流量消耗的 app 来说，最好也要做到 request 请求的时候，也用上 gzip 来压缩数据，然后服务端在 nginx 或者程序的网关那边，再根据请求头 `Content-Encoding` 是否为 `gzip` 来判断是否要对请求的数据进行 gzip 解压缩。

<!--more-->

## 实操
接下来我将通过用 golang 语言来模拟客户端的请求和服务端的响应是否要带上 gzip 头部，来模拟这一过程，代码很简单，而且已经极简了，客户端代码就是 `client.go`, 服务端代码就是 `server.go`， 不需要用到第三方库。 (我的 golang 环境是 `1.13.8`)
### 客户端
`client.go`:
```go
package main

import (
	"bytes"
	"compress/gzip"
	"fmt"
	"io/ioutil"
	"net/http"
	"flag"
)

func main() {
	var requestUseGzip = flag.String("req_use_gzip", "1", "has request body use gzip")
	var responseUseGzip = flag.String("resp_use_gzip", "1", "has response body use gzip")
	flag.Parse()
	var requestBodyStr = `{"name":"zach ke","des":"woooooo boy", "response_use_gzip": "%s"}`
	data := []byte(fmt.Sprintf(requestBodyStr, *responseUseGzip))
	var httpRequest *http.Request
	var err error
	if *requestUseGzip == "1"{
		fmt.Println("request with gzip")
		var zBuf bytes.Buffer
		zw := gzip.NewWriter(&zBuf)
		if _, err := zw.Write(data); err != nil {
			fmt.Println("gzip is faild,err:", err)
		}
		zw.Close()
		httpRequest, err = http.NewRequest("POST", "http://localhost:9905/request", &zBuf)
		if err != nil {
			fmt.Println("http request is failed, err: ", err)
		}
		httpRequest.Header.Set("Content-Encoding", "gzip")
	} else {
		fmt.Println("request without gzip")
		reader := bytes.NewReader(data)
		httpRequest, err = http.NewRequest("POST", "http://localhost:9905/request", reader)
		if err != nil {
			fmt.Println("http request is failed, err: ", err)
		}
	}
	// 通过参数判断是否返回值要用 gzip 压缩
	if *responseUseGzip == "1" {
		httpRequest.Header.Set("Accept-Encoding", "gzip")
	}else{
		// 这个也要指定，不然 response 那边获取 content-length 头部会取不到值
		httpRequest.Header.Set("Accept-Encoding", "deflate")
	}
	client := &http.Client{}
	httpResponse, err := client.Do(httpRequest)
	if err != nil {
		fmt.Println("httpResponse is failed, err: ", err)
	}
	defer httpResponse.Body.Close()

	fmt.Println("respone content length=", httpResponse.ContentLength)
	if httpResponse.StatusCode == 200 {
		var respBody string
		switch httpResponse.Header.Get("Content-Encoding") {
		case "gzip":
			fmt.Println("response with gzip")
			reader, err := gzip.NewReader(httpResponse.Body)
			if err != nil {
				fmt.Println("gzip get reader err ", err)
			}
			data, err = ioutil.ReadAll(reader)
			respBody = string(data)
		default:
			fmt.Println("response without gzip")
			bodyByte, _ := ioutil.ReadAll(httpResponse.Body)
			respBody = string(bodyByte)
		}
		fmt.Println("resp data=", respBody)
	}
}
```
可以看到，我这边通过 flag 来控制本次请求中， request 是否要 gzip 压缩，要求 response 是否也要 gzip 压缩返回。 所以总共有以下四种可能:
1. request gzip 压缩， response gzip 压缩
2. request gzip 压缩， response 不 压缩
3. request 不 压缩， response gzip 压缩
4. request 不 压缩， response 不 压缩

### 服务端
`server.go`:
```go
package main

import (
	"compress/gzip"
	"fmt"
	"io/ioutil"
	"net/http"
	"bytes"
)

func handler(resp http.ResponseWriter, req *http.Request) {
	var bodyDataStr string
	// 是否需要解 gzip 压缩
	fmt.Println("resquest content length=", req.ContentLength)
	if req.Header.Get("Content-Encoding") == "gzip" {
		fmt.Println("request with gzip")
		body, err := gzip.NewReader(req.Body)
		if err != nil {
			fmt.Println("unzip is failed, err:", err)
		}
		defer body.Close()
		data, err := ioutil.ReadAll(body)
		if err != nil {
			fmt.Println("read all is failed.err:", err)
		}
		bodyDataStr = string(data)
	} else {
		fmt.Println("request without gzip")
		data, err := ioutil.ReadAll(req.Body)
		defer req.Body.Close()
		if err != nil {
			fmt.Println("read resp is failed, err: ", err)
		}
		bodyDataStr = string(data)
	}
	fmt.Println("request json string=", bodyDataStr)

	respJson := []byte(`{code: "1",msg: "success"}`)

	// 通过头部的 Accept-Encoding 判断返回值是否要用 gzip 压缩
	if req.Header.Get("Accept-Encoding") == "gzip" {
		fmt.Println("response with gzip")
		// 添加 gzip 头部
		resp.Header().Set("Content-Encoding", "gzip")
		var zBuf bytes.Buffer
		zw := gzip.NewWriter(&zBuf)
		if _, err := zw.Write(respJson); err != nil {
			zw.Close()
			fmt.Println("gzip is faild,err:", err)
		}
		zw.Close()
		//fmt.Println("gzip content:", string(zBuf.Bytes()))
		resp.Write(zBuf.Bytes())
	}else{
		fmt.Println("response without gzip")
		// 正常不压缩 返回
		resp.Write(respJson)
	}
}

func main() {
	fmt.Println("http://localhost:9905/request")
	http.HandleFunc("/request", handler)
	http.ListenAndServe(":9905", nil)
}

```

### 运行起来
首先先运行服务端:
```go
go run server.go
```
接下来针对以上 4 种情况，分别运行 客户端 程序，然后查看一下服务端的输出

#### 1. 都压缩
客户端指令:
```text
~ >go run client.go
request with gzip
respone content length= 50
response with gzip
resp data= {code: "1",msg: "success"}
```
服务端输出:
```text
resquest content length= 85
request with gzip
request json string= {"name":"zach ke","des":"woooooo boy", "response_use_gzip": "1"}
response with gzip
```

#### 2. request 压缩， response 不压缩
客户端指令:
```text
~ >go run client.go -resp_use_gzip=0 -req_use_gzip=1
request with gzip
respone content length= 26
response without gzip
resp data= {code: "1",msg: "success"}

```
服务端输出:
```text
resquest content length= 85
request with gzip
request json string= {"name":"zach ke","des":"woooooo boy", "response_use_gzip": "0"}
response without gzip
```

#### 3. request 不压缩， response 压缩
客户端指令:
```text
~ >go run client.go -resp_use_gzip=1 -req_use_gzip=0
request without gzip
respone content length= 50
response with gzip
resp data= {code: "1",msg: "success"}
```
服务端输出:
```text
resquest content length= 64
request without gzip
request json string= {"name":"zach ke","des":"woooooo boy", "response_use_gzip": "1"}
response with gzip
```

#### 4. 都不压缩
客户端指令:
```text
~ >go run client.go -resp_use_gzip=0 -req_use_gzip=0
request without gzip
respone content length= 26
response without gzip
resp data= {code: "1",msg: "success"}
```
服务端输出:
```text
resquest content length= 64
request without gzip
request json string= {"name":"zach ke","des":"woooooo boy", "response_use_gzip": "0"}
response without gzip
```

其实判断是否要用 gzip 压缩很简单，通过 request 或者 response 的头部设置即可:
1. 如果自身设置 `Content-Encoding` 为 `gzip`, 说明你的数据是 gzip 压缩过的， 对方要进行 gzip 解压才行
2. 如果你希望对方给你的数据是 gzip 压缩过的，那么就设置头部 `Accept-Encoding` 为 `gzip`


## gzip 压缩后体积更大的情况
不知道以上有一种情况，你们有没有发现过，就是无论是 request 还是 response 的数据， gzip 压缩过后的字节体积比原先更多了，反而是不压缩的字节体积更小，为了更直观，我列个表格:

| - |请求 body 字节| 响应 body 字节|
|---|---|---|
|都压缩| 85 | 50 |
|只压缩 request| 85 | 26 |
|只压缩 response| 64 | 50 |
| 都不压缩 | 64 | 26 |

其实这样子是对的，事实上 gzip 压缩的时候，在原数据比较小的时候，反而压缩后的体积会更大， 这也是为啥 nginx 要有一个 `gzip_min_length` 参数来控制压缩的最小字节数，一般我们是设置为 1k (默认值是 20)， 具体可以看 {% post_link gzip-min-length %}

接下来我们将 请求的 body 和 响应的 body 都调大一点
```text
	respJson := []byte(`{code: "1",msg: "success", des: "12asdfsafsafsagaegewgsdfsdfsdfsdfgghhhhhhhhhhsfgegrfsdffffffffffadffwefwefefwafrrrrrrrrrrrrrwgsdfsdfsdfsdfgghhhhhhhhhhsfgegrfsdffffffffffadffwefwefefwafrrrrrrrrrwgsdfsdfsdfsdfgghhhhhhhhhhsfgegrfsdffffffffffadffwefwefefwafrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrr"}`)
```
```text
	var requestBodyStr = `{"name":"zach ke","des":"woooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boy", "response_use_gzip": "%s"}`
```
可以看到 都压缩后的体积是: 
```text
request with gzip
respone content length= 111
response with gzip
resp data= {code: "1",msg: "success", des: "12asdfsafsafsagaegewgsdfsdfsdfsdfgghhhhhhhhhhsfgegrfsdffffffffffadffwefwefefwafrrrrrrrrrrrrrwgsdfsdfsdfsdfgghhhhhhhhhhsfgegrfsdffffffffffadffwefwefefwafrrrrrrrrrwgsdfsdfsdfsdfgghhhhhhhh
hhsfgegrfsdffffffffffadffwefwefefwafrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrr"}
```
```text
resquest content length= 89
request with gzip
request json string= {"name":"zach ke","des":"woooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo
 boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boy", "response_use_gzip": "1"}
response with gzip
```
分别是请求体积是 89， 响应体积 111。 

而如果是没有压缩的，那么就是：
```text
request without gzip
respone content length= 343
response without gzip
resp data= {code: "1",msg: "success", des: "12asdfsafsafsagaegewgsdfsdfsdfsdfgghhhhhhhhhhsfgegrfsdffffffffffadffwefwefefwafrrrrrrrrrrrrrwgsdfsdfsdfsdfgghhhhhhhhhhsfgegrfsdffffffffffadffwefwefefwafrrrrrrrrrwgsdfsdfsdfsdfgghhhhhhhh
hhsfgegrfsdffffffffffadffwefwefefwafrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrr"}
```
```text
resquest content length= 383
request without gzip
request json string= {"name":"zach ke","des":"woooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo
 boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boywoooooo boy", "response_use_gzip": "0"}
response without gzip
```
分别是请求体积是 383， 响应体积 343。  可以看到这个压缩率就不错了。 

## 总结
基本上数据量越多的话， 用 gzip 能省的流量越多，而且还能够加快网络传输，何乐而不为。 (事实上，还可以设置压缩比例和压缩算法，我这边为了演示，只使用默认的配置)

不过request 请求的话，如果用 gzip 的话，会影响性能的，数据量越大的话， 压缩和解压所消耗的时间也会越长，所以一般 pc 端的站点也不需要， 只有一些对流量消耗比较敏感的 app 可以考虑用这种方式。