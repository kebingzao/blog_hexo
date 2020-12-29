---
title: 使用 golang 批量检查站点 ssl 证书是否过期
date: 2020-12-29 17:32:29
tags: golang
categories: golang相关
---
## 前言
现在的 ssl 证书变成一年一换了， 所以运维人员每年都要在证书过期之前要去换证书，我们的业务涉及到二级域名以及下面的三级域名其实不少 (超过了 50 个), 而且在换证书的过程中，还会存在以下问题:
1. 服务部署在不同的云服务厂商，所以换证书的渠道可能有好几个(以 aws 来说，有些证书是在 EC2 上，有些用到了 cloudfront 加速的，还要去后台换), 更别说有好几个云服务厂商
2. 有时候不是简单的替换证书就完事了，大部分服务还要重启，比如 golang， php 服务一般也要重启 nginx， 所以因为量太大了， 有时候可能会忘掉
3. 验证的时候，https 还比较好验证， 如果是其他协议，比如 wss，那个就比较不好验证了。如果证书过期了，连接 wss 的时候， 浏览器就会报这个错

![png](1.png)

<!--more-->
基于以上的难点，虽然运维人员已经很尽责了，但是有时候还是会漏掉，导致业务出问题。事实上运维人员也有证书检测工具，不过作为业务方面的服务端人员，我们更关心证书替换的时候，我们涉及到的服务是否都正常。 因此我们需要在证书替换完之后，用脚本检验一下 ssl 连接是否都正常。

## 脚本
所以我们打算用 golang 写一个简单的脚本，去检测我们的所有的业务站点的 ssl 连接，判断证书是否正常。 幸运的是，在 github 上有找到了这么一个脚本: [go-check-certs](https://github.com/timewasted/go-check-certs), 非常符合我们的需求，逻辑也非常的简单， 代码量也很少，就一个文件， 而且也不需要去依赖其他的第三方包，所以我这边直接将代码贴出来，然后再分析:

```go
// Copyright 2013 Ryan Rogers. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.

package main

import (
	"crypto/tls"
	"crypto/x509"
	"flag"
	"fmt"
	"io/ioutil"
	"log"
	"strings"
	"sync"
	"time"
)

const defaultConcurrency = 8

const (
	errExpiringShortly = "%s: ** '%s' (S/N %X) expires in %d hours! **"
	errExpiringSoon    = "%s: '%s' (S/N %X) expires in roughly %d days."
	errSunsetAlg       = "%s: '%s' (S/N %X) expires after the sunset date for its signature algorithm '%s'."
)

type sigAlgSunset struct {
	name      string    // Human readable name of signature algorithm
	sunsetsAt time.Time // Time the algorithm will be sunset
}

// sunsetSigAlgs is an algorithm to string mapping for signature algorithms
// which have been or are being deprecated.  See the following links to learn
// more about SHA1's inclusion on this list.
//
// - https://technet.microsoft.com/en-us/library/security/2880823.aspx
// - http://googleonlinesecurity.blogspot.com/2014/09/gradually-sunsetting-sha-1.html
var sunsetSigAlgs = map[x509.SignatureAlgorithm]sigAlgSunset{
	x509.MD2WithRSA: sigAlgSunset{
		name:      "MD2 with RSA",
		sunsetsAt: time.Now(),
	},
	x509.MD5WithRSA: sigAlgSunset{
		name:      "MD5 with RSA",
		sunsetsAt: time.Now(),
	},
	x509.SHA1WithRSA: sigAlgSunset{
		name:      "SHA1 with RSA",
		sunsetsAt: time.Date(2017, 1, 1, 0, 0, 0, 0, time.UTC),
	},
	x509.DSAWithSHA1: sigAlgSunset{
		name:      "DSA with SHA1",
		sunsetsAt: time.Date(2017, 1, 1, 0, 0, 0, 0, time.UTC),
	},
	x509.ECDSAWithSHA1: sigAlgSunset{
		name:      "ECDSA with SHA1",
		sunsetsAt: time.Date(2017, 1, 1, 0, 0, 0, 0, time.UTC),
	},
}

var (
	hostsFile   = flag.String("hosts", "./host.txt", "The path to the file containing a list of hosts to check.")
	warnYears   = flag.Int("years", 0, "Warn if the certificate will expire within this many years.")
	warnMonths  = flag.Int("months", 0, "Warn if the certificate will expire within this many months.")
	warnDays    = flag.Int("days", 0, "Warn if the certificate will expire within this many days.")
	checkSigAlg = flag.Bool("check-sig-alg", true, "Verify that non-root certificates are using a good signature algorithm.")
	concurrency = flag.Int("concurrency", defaultConcurrency, "Maximum number of hosts to check at once.")
)

type certErrors struct {
	commonName string
	errs       []error
}

type hostResult struct {
	host  string
	err   error
	certs []certErrors
}

func main() {
	flag.Parse()

	if len(*hostsFile) == 0 {
		flag.Usage()
		return
	}
	if *warnYears < 0 {
		*warnYears = 0
	}
	if *warnMonths < 0 {
		*warnMonths = 0
	}
	if *warnDays < 0 {
		*warnDays = 0
	}
	if *warnYears == 0 && *warnMonths == 0 && *warnDays == 0 {
		*warnDays = 30
	}
	if *concurrency < 0 {
		*concurrency = defaultConcurrency
	}

	processHosts()
}

func processHosts() {
	done := make(chan struct{})
	defer close(done)

	hosts := queueHosts(done)
	results := make(chan hostResult)

	var wg sync.WaitGroup
	wg.Add(*concurrency)
	for i := 0; i < *concurrency; i++ {
		go func() {
			processQueue(done, hosts, results)
			wg.Done()
		}()
	}
	go func() {
		wg.Wait()
		close(results)
	}()

	for r := range results {
		if r.err != nil {
			log.Printf("%s: %v\n", r.host, r.err)
			continue
		}
		for _, cert := range r.certs {
			for _, err := range cert.errs {
				log.Println(err)
			}
		}
	}
}

func queueHosts(done <-chan struct{}) <-chan string {
	hosts := make(chan string)
	go func() {
		defer close(hosts)

		fileContents, err := ioutil.ReadFile(*hostsFile)
		if err != nil {
			return
		}
		lines := strings.Split(string(fileContents), "\n")
		for _, line := range lines {
			host := strings.TrimSpace(line)
			if len(host) == 0 || host[0] == '#' {
				continue
			}
			select {
			case hosts <- host:
			case <-done:
				return
			}
		}
	}()
	return hosts
}

func processQueue(done <-chan struct{}, hosts <-chan string, results chan<- hostResult) {
	for host := range hosts {
		select {
		case results <- checkHost(host):
		case <-done:
			return
		}
	}
}

func checkHost(host string) (result hostResult) {
	result = hostResult{
		host:  host,
		certs: []certErrors{},
	}
	conn, err := tls.Dial("tcp", host, nil)
	if err != nil {
		result.err = err
		return
	}
	defer conn.Close()

	timeNow := time.Now()
	checkedCerts := make(map[string]struct{})
	for _, chain := range conn.ConnectionState().VerifiedChains {
		for certNum, cert := range chain {
			if _, checked := checkedCerts[string(cert.Signature)]; checked {
				continue
			}
			checkedCerts[string(cert.Signature)] = struct{}{}
			cErrs := []error{}

			// Check the expiration.
			if timeNow.AddDate(*warnYears, *warnMonths, *warnDays).After(cert.NotAfter) {
				expiresIn := int64(cert.NotAfter.Sub(timeNow).Hours())
				if expiresIn <= 48 {
					cErrs = append(cErrs, fmt.Errorf(errExpiringShortly, host, cert.Subject.CommonName, cert.SerialNumber, expiresIn))
				} else {
					cErrs = append(cErrs, fmt.Errorf(errExpiringSoon, host, cert.Subject.CommonName, cert.SerialNumber, expiresIn/24))
				}
			}

			// Check the signature algorithm, ignoring the root certificate.
			if alg, exists := sunsetSigAlgs[cert.SignatureAlgorithm]; *checkSigAlg && exists && certNum != len(chain)-1 {
				if cert.NotAfter.Equal(alg.sunsetsAt) || cert.NotAfter.After(alg.sunsetsAt) {
					cErrs = append(cErrs, fmt.Errorf(errSunsetAlg, host, cert.Subject.CommonName, cert.SerialNumber, alg.name))
				}
			}

			result.certs = append(result.certs, certErrors{
				commonName: cert.Subject.CommonName,
				errs:       cErrs,
			})
		}
	}

	return
}
```
逻辑其实很好理解， 就是读取一个存放各个 `域名:ip` 的文件(这个就是我们要检测域名的存放的文件)，跟我们的 host 文件一样，也是一行一个，如果是 `#` 开头，那么就是注释，直接跳过。 我这边改了一行代码，就是默认读取同目录下的 `host.txt` 文件， 这样子就不需要每次都要在命令行指定
```go
hostsFile   = flag.String("hosts", "./host.txt", "The path to the file containing a list of hosts to check.")
```

他还提供几个指定有效时间范围内的参数 `-years`, `-months`, `-days` ， 所以我们可以自定义检查出 `几天/几个月/几年` 之内会过期的证书，提前预知。 默认是 30 天， 也就是如果站点的证书是在 30 天内过期的话，那么就会输出到控制台。 当然如果证书过期的话，也会输出。

他的原理其实就是用 tcp 去连接:
```go
conn, err := tls.Dial("tcp", host, nil)
```
然后得到连接详情里面的证书信息，然后匹配证书里面的过期时间，就可以了。 类似于得到我们浏览器的这个:

![png](2.png)

## 测试
接下来我们测试一下，在 host.txt 文件加上(为了隐私，相关二级域名全部替换成 foo):
```text
m.foo.com:443
comet1.foo.com:6977
```
如果是 https， 端口一般是 443， 如果是 wss 的话，就看具体开启的 ssl 端口是哪个，本例的 wss 端口就是 6977 端口。

执行一下，发现没有结果:
```text
check-certs>go run main.go
```
这个是因为没有特殊指定 `year`， `month`， `day` 参数，所以默认 day 是 30天，而我们的证书是有效的，并且有效期大于 30 天，不在 30 天之内，所以就没有在要输出的错误信息里面。 因此我们将时间改长一点，改成 2 年之内有过期就输出来:
```text
check-certs>go run main.go -years=2
2020/12/29 16:54:58 m.foo.com:443: '*.foo.com' (S/N B4BE823F8FC3AD98) expires in roughly 333 days.
2020/12/29 16:54:58 comet1.foo.com:6977: '*.foo.com' (S/N B4BE823F8FC3AD98) expires in roughly 333 days.
```
这时候可以看到这两个域名都满足了条件，并且输出到控制台了。距离过期还剩下 333 天。

接下来我们将 `m.foo.com:443` 这个域名的证书人为的设置为过期，看看会不会检测出来:
```go
check-certs>go run main.go -years=2
2020/12/29 16:59:33 m.foo.com:443: x509: certificate has expired or is not yet valid
2020/12/29 16:59:33 comet1.foo.com:6977: '*.foo.com' (S/N B4BE823F8FC3AD98) expires in roughly 333 days.
```
可以看到，有提示证书过期了，这个其实是在进行 tcp 连接的时候，就连接不上了， 然后报的错误了。 

## 总结
所以通过这种方式，我们可以批量的检测我们的业务站点的 ssl 证书是否替换成功。 甚至还可以提前知道有哪些证书快要过期了。 而且如果你的 host.txt 文件很长的话，还可以设置 并发检测参数 `concurrency`, 这个默认是 8，基本上可以符合需求了。

---
参考资料
- [go-check-certs](https://github.com/timewasted/go-check-certs)










