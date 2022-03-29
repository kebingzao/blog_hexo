---
title: 浏览器判断某一个 ip 是否与其在同一个局域网的几种方式
date: 2022-03-29 19:17:44
tags: 
- js
- 浏览器
categories: 前端相关
---
## 前言
前段时间有个需求， 就是我们有做一个 web 的投屏端， 可以将另一个客户端(比如 android，ios，win，mac) 投屏到 web 站点来。 但是期间因为涉及到引流， 所以针对投屏的客户端是否在同一个局域网下要做不同的判断，如果在同一个局域网下，那么就可以免费使用，如果不是的话，就会有其他的引导。

所以我们得到客户端的 ip 地址之后，需要判断当前浏览器是否跟这一台客户端在同一个局域网下。 有几种判断方式

> 以下的测试数据，都是基于 chrome 98 的浏览器， 早期的浏览器可能会有不同的表现

## 1. 让客户端开启一个本地端口，然后浏览器去请求这个端口
最简单的方式就是让这个客户端开启一个本地端口，就是类似于 ip+port 的方式，可以是 http ，也可以是 websocket 的方式。 然后让浏览器去请求这个端口。

如果可以请求成功，那么就说明是在同一个局域网。 不过这边有几个问题:
<!--more-->
### 1.1. 在 https 下会有问题
如果当前的站点是 https 的话，那么在 https 下请求 http 协议的 ip 形式的接口的话， 浏览器是会直接 block 掉的 (请求不会到服务端这边)

![1](1.png)

如果是 ws 协议的 ip 地址的 websocket 的话，也是一样会被 block 掉 (请求不会到服务端这边)

![1](2.png)

那如果是 https 或者是 wss 下的 ip 形式的接口呢 (客户端是没有合格的证书的，就算加上 tls ，也是用自制证书的 ip 地址的方式)，是不是就可以? 答案肯定是不行的， 直接就报证书错误了 (除非事先这个接口已经在浏览器那边事先手动允许访问了)

![1](3.png)

所以总结一下：
1. 在 https 的站点下， 非 tls 的 http 和 ws 的 ip 地址的接口请求，会直接 block 掉
2. 在 https 的站点下， tls 的 https 和 wss 的 ip 地址的接口请求，会报证书错误，也是 block

### 1.2. https 请求 localhost 的例外情况
那有没有例外呢，在我的测试中，还真有一个例外， 就是如果是本机的客户端的服务，以 localhost 的 host 方式，不管是 `http://localhost:3000` 还是 `ws://localhost:3000`, 都可以在 https 的站点下 work， 不过两者有差别:
1. 如果是 localhost 的 http 请求的话，浏览器也会 block (测试的时候，之所以 block 是因为 CORS 限制)，不过他只会 block 返回值，但是请求是有出去的，也就是说返回值是 200，只不过 js 没法取的返回值，但是服务端是有收到这个请求的

![1](4.png)

2. 如果是 localhost 的 ws 请求的话， 哪怕是在 https 的站点，也是可以 work 的。就跟在 http 站点中表现一样

关于这一点其实我也有点疑惑，正常来说， localhost 的形式跟 ip 地址的形式应该差别不大才对， 但是实验的结果却是 ip 地址的不行，localhost 就可以， 差别可能就是 ip 地址的不仅仅是本机，同一个局域网下的机器都有，但是 localhost 却是限于本机上开启的服务， 可能相对来说也会比较安全吧。

事实上我们还真有一个服务是在 https 站点下调用 localhost 服务接口的，就是我们 win 端应用在进行第三方 google 登录授权的时候，就是跳转到我们的 https 下的官网站点的 web 页面进行授权的， 然后授权完成之后，就会调用该应用创建的 localhost 的服务接口，将信息传递到 win 端的应用， 从而完成整个第三方授权过程。

### 1.3. 在 http 下也会有问题
在 https 下有问题， 其实在 http 下也会有问题的， chrome 可能在 `2022-05-17` 之后， http 协议的远程站点，就不能再请求本地的专用网络了。包括 ip 形式的， localhost 形式的。

之前我还针对过这个情况，进行过一次兼容: {% post_link chrome-private-cors-error %}

综上所述， 通过请求 ip+port 的方式，请求客户端暴露的接口来判断跟客户端是处于同一个局域网的策略，是行不通的

## 2. 根据 ip 网段来判断是否是同一个局域网
还是一种方式，就是既然可以得到 客户端的 ip (可以通过很多途径，比如去服务端取，或者直接两者连接同一个 websocket 通道，然后进行信息转发之类的)， 那能不能直接通过这个 ip 是不是局域网 ip 网段来直接确定是不是在同一个局域网。

我们知道从 `0.0.0.0` 到 `255.255.255.255` 这些网段中，其中有一些网段是给局域网的，包括:
- a类网 `10.0.0.0~10.255.255.255`
- b类网 `172.16.0.0~172.31.255.255`
- c类网 `192.168.0.0~192.168.255.255`

那能不能直接根据客户端传过来的 ip 地址来判断是不是在同一个局域网内，比如这样子判断:
```javascript
checkIpInLAN: function (ip) {
    return ip && (ip.indexOf("10.") === 0 || ip.indexOf("172.") === 0 || ip.indexOf("192.168") === 0);
},
```
其实也是不准确的，因为这边混淆了一个概念，以上确实是属于局域网的网段， 但是该网段不一定跟当前的浏览器在同一个局域网。

因为同样是局域网的网段，但是相互之间可能就是不通的，比如我们公司提供给客人的wifi，和我们内部人员自己使用的 wifi，就是都属于 `192.168.x.x` 的局域网网段， 但是两者其实是不通， 也就是两者不在同一个局域网下。

所以这种方式也是没办法判断浏览器和客户端是在同一个局域网下的

## 3. 通过 webrtc 的 stun/turn 得到局域网 ip，然后扔给客户端，让客户端去 ping
还有一种方式就是通过 webrtc 的 turn server 去获取当前浏览器的局域网 ip，然后将这个 ip 扔给客户端，让客户端去 ping，如果通，说明就在同一个局域网。

但是我后面试了一下，发现这种方式会有问题， 正常的 webrtc 连接，在进行 sdp 和 ice 的交换完之后，成功连接的时候，如果这时候是在同一个局域网的话， ice 会返回 host 的凭证:
```javascript
{"candidate":"candidate:1316551156 1 udp 2122260223 192.168.40.51 56234 typ host generation 0 ufrag n4Ma network-id 1","sdpMid":"0","sdpMLineIndex":0}
```
这时候我们可以看到确实有出现过 host 信息，里面有浏览器的局域网 ip : `192.168.40.51`

但是如果只是本地去连接 stun/turn server 的话，他返回的 host 信息是有问题的，变成这样子:
```javascript
{"candidate":"candidate:1316551156 1 udp 2113937151 493c5821-465b-41d5-9cfe-f0b703d3add6.local 57516 typ host generation 0 ufrag gjxD network-cost 999","sdpMid":"0","sdpMLineIndex":0}
```
局域网 ip 就变成这样子 `493c5821-465b-41d5-9cfe-f0b703d3add6.local`

后面查了一下，原来 chrome 在 75 版本之后，使用了一种 mDNS 的技术，将 local ip 匿名了: [Google Chrome - Anonymize local IPs exposed by WebRTC prevents live view in cloud console](https://support.imperosoftware.com/support/solutions/articles/44001790065-google-chrome-anonymize-local-ips-exposed-by-webrtc-prevents-live-view-in-cloud-console)

导致我们没办法看到这个局域网 ip。

所以只是单纯的通过 stun/turn server 去检测的话， chrome 的 local ip 会被匿名， 我们也无从得知。

这边附上操作代码:

```javascript
  const findIP = (onNewIP) => {      
    const myPeerConnection = window.RTCPeerConnection ||
      window.mozRTCPeerConnection ||
      window.webkitRTCPeerConnection
    const pc = new myPeerConnection({
      iceServers: [{
        urls: ["turn:turn.example.com:3478"],
        credential: "8+aKeo7wxxxxGVkdwzqSs=",
        username: "16486xxxxx9"
      },] })
    const noop = () => {}
    const localIPs = {}
    const ipRegex = /([0-9]{1,3}(\.[0-9]{1,3}){3}|[a-f0-9]{1,4}(:[a-f0-9]{1,4}){7})/g

    const ipIterate = (ip) => {
      if (!localIPs[ip]) onNewIP(ip)
      localIPs[ip] = true
    }
    pc.createDataChannel('') // create a bogus data channel
    pc.createOffer(sdp => {
      sdp.sdp.split('\n').forEach(line => {
        if (line.indexOf('candidate') < 0) return
        line.match(ipRegex).forEach(ipIterate)
      })
      pc.setLocalDescription(sdp, noop, noop)
    }, noop) // create offer and set local description
    pc.onicecandidate = (ice) => { // listen for candidate events
      if (
        !ice ||
        !ice.candidate ||
        !ice.candidate.candidate ||
        !ice.candidate.candidate.match(ipRegex)
      ) return
      ice.candidate.candidate.match(ipRegex).forEach(ipIterate)
    }
  }

  const addIP = (ip) => {
    console.log('got ip: ', ip)
  }

  findIP(addIP)
```

值得说明一点的是，在连接 turn/stun server 的时候，我看到网上都是用 `stun:stun.services.mozilla.com` 这种公共的 stun 服务， 我试了一下得不到 host 信息， 换成自己搭的 turnserver， 后面就可以得到 host 信息，虽然被匿名了

还有就是，这种方式虽然没办法得到局域网 ip， 但是却可以得到外网 ip， 以下就是我利用上述的程序得到的 外网 ip `125.xx.xx.250` :
```javascript
 {"candidate":"candidate:842163049 1 udp 1677729535 125.xx.xx.250 55956 typ srflx raddr 0.0.0.0 rport 0 generation 0 ufrag 12LL network-cost 999","sdpMid":"0","sdpMLineIndex":0}
```

## 4. 跟客户端建立起 webrtc 连接，如果是在一个局域网的话，会有 localCandidate 信息
目前最稳妥的方式，就是跟客户端建立起 webrtc 连接， 这时候如果真的是在同一个局域网下的话，就可以得到 localCandidate，那么就可以认为是在同一个局域网下了。

## 总结
本文介绍了几种浏览器判断跟客户端是否在同一个局域网的方式， 前三种都有一定的问题， 只有第四种才能准确的判断是在同一个局域网下。



