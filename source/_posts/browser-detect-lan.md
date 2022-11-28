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
还有一种方式就是通过 webrtc 的 stun/turn server 去获取当前浏览器的局域网 ip，然后将这个 ip 扔给客户端，让客户端去 ping，如果通，说明就在同一个局域网。

但是我后面试了一下，发现这种方式会有问题， 他返回的 host 信息是有问题的，变成这样子:
```javascript
{"candidate":"candidate:1316551156 1 udp 2113937151 493c5821-465b-41d5-9cfe-f0b703d3add6.local 57516 typ host generation 0 ufrag gjxD network-cost 999","sdpMid":"0","sdpMLineIndex":0}
```
局域网 ip 就变成这样子 `493c5821-465b-41d5-9cfe-f0b703d3add6.local`

后面查了一下，原来 chrome 在 75 版本之后，使用了一种 mDNS 的技术，将 local ip 匿名了: [Google Chrome - Anonymize local IPs exposed by WebRTC prevents live view in cloud console](https://support.imperosoftware.com/support/solutions/articles/44001790065-google-chrome-anonymize-local-ips-exposed-by-webrtc-prevents-live-view-in-cloud-console)

导致我们没办法看到这个局域网 ip。 这边附上操作代码:

```javascript
  const findIP = (onNewIP) => {      
    const myPeerConnection = window.RTCPeerConnection ||
      window.mozRTCPeerConnection ||
      window.webkitRTCPeerConnection
    const pc = new myPeerConnection({
      iceServers: [
        {urls: 'stun:stun.l.google.com:19302'},
        {urls: 'stun:stun.services.mozilla.com:3478'},
        {urls: 'stun:stun.qq.com:3478'}
      ] })
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
    document.body.append(`----get ip: ${ip}`)
  }

  findIP(addIP)
```
这种方式虽然没办法得到局域网 ip， 但是却可以得到外网 ip， 以下就是我利用上述的程序得到的 外网 ip `125.xx.xx.250` :
```javascript
 {"candidate":"candidate:842163049 1 udp 1677729535 125.xx.xx.250 55956 typ srflx raddr 0.0.0.0 rport 0 generation 0 ufrag 12LL network-cost 999","sdpMid":"0","sdpMLineIndex":0}
```

这个在 chrome 是有配置项的: `chrome://flags/#enable-webrtc-hide-local-ips-with-mdns`, 默认是开启的。 如果将其设置为 disable 后， 那么是不会启用 mDNS 技术来隐藏真实的 local ip， 那么直接就是局域网 ip 了

![1](5.png)

```javascript
{"candidate":"candidate:1316551156 1 udp 2122260223 192.168.40.51 56234 typ host generation 0 ufrag n4Ma network-id 1","sdpMid":"0","sdpMLineIndex":0}
```

### 有一种例外情况可以忽略 mDNS
查阅了相关资料，发现有一种例外情况也可以显示原始的局域网 ip:

{% blockquote WebRTC 公开的私有 IP 地址更改为 mDNS 主机名 https://groups.google.com/g/discuss-webrtc/c/6stQXi72BEU %}
对应用程序的影响

当该功能处于活动状态时，ICE 候选主机中的私有 IP 地址将被替换为 mDNS 主机名，例如1f4712db-ea17-4bcf-a596-105139dfd8bf.local 。
目前，除具有 getUserMedia 权限的站点外，该功能对所有站点都有效，假定这些站点具有较高的用户信任度。 
{% endblockquote %}

那就是在有启用 `getUserMedia` 权限的站点 (要是 https 的站点)， 所以将上述的代码最后一行改为: 
```javascript
  navigator.mediaDevices.getUserMedia({ video: false, audio: true }).then( stream => {
    findIP(addIP)
  }).catch(err => {
    findIP(addIP)
  })
```

这时候就会弹出授权框

![1](6.png)

点击允许之后，就会出现局域网 ip 了 (而且下一次不需要再授权了，只要有权限，那么就会有不匿名的局域网 ip)

![1](7.png)

## 4. 跟客户端建立起 webrtc 连接，如果是在一个局域网的话，会有 localCandidate 信息
目前最稳妥的方式，就是跟客户端建立起 webrtc 连接， 这时候如果真的是在同一个局域网下的话，就可以得到 localCandidate，同时他的值是 `host` 的话， 那么就可以认为是在同一个局域网下了。

具体操作就是在 ice 连接成功的情况下， 这时候我们可以获取 webrtc 的统计数据(这个是一个异步)， 通过 RTCPeerConnection 对象的 getStats 方法:
```javascript
const report = await this.#pc.getStats()
// 遍历report，按照 type 来分类
/** @type {Record<string, any[]>} */
const data = {};
report.forEach((item) => {
    if (data[item.type]) {
        data[item.type].unshift(item)
    } else {
        data[item.type] = [item]
    }
});
return data
```
然后这时候得到统计数据之后, 就可以去找到当前连接的通道 `transport`, 然后在 `candidate-pair` 找到这个通道,  然后接下来在 `local-candidate` 找到这个 id, 最后一步就是判断是不是 host 了

具体的代码如下:
```javascript
/**
 * 通过 webrtc 的 stat 数据来判断当前连接的通道是否是局域网
 * @param stat
 * @returns {boolean}
 */
checkIsLocalCandidateType(stat){
  const originRaw = stat._raw
  const throwErr = (msg) => {
    throw { msg: `[checkIsLocalCandidateType]: ${msg}` }
  }
  const findLocalCandidate = (targetCandidate) => {
    if(targetCandidate.length){
      let localCandidateId = targetCandidate[0]["localCandidateId"]
      // 然后接下来在 local-candidate 找到这个 id
      if(originRaw["local-candidate"]){
        let targetLocalCandidate = originRaw["local-candidate"].filter(item => item.id === localCandidateId)
        if(targetLocalCandidate.length){
          let candidateType = targetLocalCandidate[0]["candidateType"]
          // 最后一步就是判断是不是 host 了
          logger.debug(`[WebRtcProcess] => checkIsLocalCandidateIdFromWebRtcStats: ${candidateType}`)
          return candidateType === "host"
        }else{
          throwErr("local-candidateType not found")
        }
      }else{
        throwErr("local-candidate not found")
      }
    }else{
      throwErr("targetCandidate not found")
    }
  }
  // 找到当前连接的通道, 这个是 chrome 的流程
  if(originRaw.transport?.length){
    const selectedCandidatePairId = originRaw.transport[0].selectedCandidatePairId
    // 然后在 candidate-pair 找到这个通道
    if(originRaw["candidate-pair"]?.length){
      let targetCandidate = originRaw["candidate-pair"].filter(item => item.id === selectedCandidatePairId)
      return findLocalCandidate(targetCandidate)
    }else{
      throwErr("candidate-pair not found")
    }
  }else{
    // firefox 是没有 transport 参数的，要额外处理
    if(originRaw["candidate-pair"]?.length){
      let targetCandidate = originRaw["candidate-pair"].filter(item => item["selected"] === true && item["state"] === "succeeded")
      return findLocalCandidate(targetCandidate)
    }
    // 有些旧的浏览器比如 safari 11.1.2，就会出现找不到 local-candidate 参数的情况
  }
  return false
}
```
这个是在 ice 连接之后， 第一次获取 getStats 方法的时候，就可以判断， 因为 ice 连接之后，说明当前的 webrtc 肯定是有连接的 transport 通道的， 只要当前的这一条通道的 `local-candidate` 的值是 host， 那么就肯定是在同一个局域网下。

通过这种方式绝大部分情况下是可以判断浏览器和客户端是在同一个局域网下，但是有一种情况下会误判， 就是当前是局域网，但是网络连通不好，导致不是 host 先连上，而是穿透(srflx)或者转发(relay)先连上 transport, 这时候第一次的 getStats 就不会是 host 了，因为他当前的连接通道确实不是 host。

同时需要注意一点的是，正常情况下这个 host 的 `RTCIceCandidate` 也是不会显示 局域网 ip 的， 而且他不是用 mDNS 加密，而是直接为空:
```javascript
{
    "id": "RTCIceCandidate_1wdP0rnA",
    "timestamp": 1651027428646,
    "type": "local-candidate",
    "transportId": "RTCTransport_audio_1",
    "isRemote": false,
    "networkType": "unknown",
    "ip": "",
    "address": "",
    "port": 63261,
    "protocol": "udp",
    "candidateType": "host",
    "priority": 2113937151
  }
```

也是只有在启用 `getUserMedia` 权限，才会显示局域网 ip 出来。

## 5. 一样建立 webrtc 连接，但是双方只接受 host ice， 如果还能连上，那么肯定是在同一个局域网
上述的情况是连接上 webrtc 之后，才去判断是不是走 host 的 transport 通道，再去判断当前是否是局域网连接。

但是如果我们在传输交换 ice 的时候，只传和接收 局域网 ice 的话， 那么能连接上 webrtc 的情况下，肯定是局域网。 因为在只有 host 模式下 ice 可以连接上，说明两端肯定是在同一个局域网下

所以就要改成双方在发送 ice 的时候，只发送 host 模式下的 ice, 类似于这样子
```text
{"sdpMLineIndex":0,"sdpMid":"0","candidate":"candidate:559267639 1 udp 2122202367 ::1 45061 typ host generation 0 ufrag yvH5 network-id 2"}
{"sdpMLineIndex":0,"sdpMid":"0","candidate":"candidate:2065939615 1 udp 2122260223 192.168.197.13 47089 typ host generation 0 ufrag yvH5 network-id 3 network-cost 10"}
```

然后在 `this.#pc.addIceCandidate(new RTCIceCandidate(item), () => {` 添加的时候，就会只有 host 的 ice 了。

这种情况下，对于一端浏览器端，一端其他端(windows，android， ios)，这个其他端因为有更详细的底层 sdk 的 api， 所以可以做到只发送 host 的 ice。 

但是对于两端都是浏览器端的情况，因为预置的 `RTCIceTransportPolicy` 只有两个值， relay 和 all， 并没有类似于 `only-local` 的值，没办法控制只发送 host ice 的情况

{% blockquote mozilla https://w3c.github.io/webrtc-pc/#rtcicetransportpolicy-enum %}
enum RTCIceTransportPolicy {
  "relay",
  "all"
};
{% endblockquote %}

但是我们可以在接收添加 ice 的时候，过滤掉非 host 的 ice，比如这样子:
```text
/**
 * 添加ICE
 * pc.addIceCandidate
 */
addIce = (data) => {
    return new Promise((resolve) => {
        const RTCIceCandidate = window.RTCIceCandidate || (window as any).mozRTCIceCandidate || (window as any).webkitRTCIceCandidate
        data.forEach(item => {
            // 只接收 host 的 ice
            if(JSON.stringify(item).includes("typ host")){
                this.#pc.addIceCandidate(new RTCIceCandidate(item), () => {
                    logger.debug(`[PeerConnection]: add ice success: ${JSON.stringify(item)}`)
                    this.#emitter.emit('addIceStatus', 'success')
                }, (err) => {
                    logger.error(`[PeerConnection]: add ice fail: ${err}`)
                    this.#emitter.emit('addIceStatus', 'fail')
                })
            }
        })
        resolve(data)
    });
}
```

这样子，如果 ice 可以连接上，那么肯定两端是在同一个局域网中。


## 总结
本文介绍了几种浏览器判断跟客户端是否在同一个局域网的方式， 前三种都有一定的问题， 第四种一定情况下，在网络比较差的情况下，会有误判。 只有第五种是可以 100% 判断是在同一个局域网下的情况。

---

参考资料:
- [WebRTC 公开的私有 IP 地址更改为 mDNS 主机名](https://groups.google.com/g/discuss-webrtc/c/6stQXi72BEU)
- [Google Chrome - 匿名化 WebRTC 公开的本地 IP 可防止云控制台中的实时视图](https://support.imperosoftware.com/support/solutions/articles/44001790065-google-chrome-anonymize-local-ips-exposed-by-webrtc-prevents-live-view-in-cloud-console)

