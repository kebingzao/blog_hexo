---
title: webrtc 浏览器启用麦克风将其当做 audio track 传给对方
date: 2022-07-04 17:51:03
tags: 
- webrtc
- browser
categories: webrtc相关
---
## 前言
这段时间一直在处理浏览器 webrtc 投屏的事情，不管是其他端(android, ios, win, mac) 投给浏览器， 还是浏览器自己投给浏览器, 都会涉及到双向语音。

在这期间有踩了一些坑， 干脆就记录一下。

webrtc 浏览器要发送语音，那就是要启用麦克风来收集音频，然后传给对方。 因为浏览器可以作为投屏的投屏端(source)，以及接受投屏的客户端(node), 这两种情况下的处理方式是不一样的，所以就分开讲。

## 浏览器作为客户端(node)启用麦克风将语音传给对方的几种方式
> 本文不是科普文，所以对于 webrtc 的一些技术，不在这边进行基础讲解

浏览器作为客户端(node)启用麦克风将语音传给对方，在我的实作中， 其实有 3 种:
<!--more-->
### 1. 进行 webrtc 刚连接的时候，就收集音频
第一种最简单，就是在我初始化完 peerconnection 之后， 这时候还没有交换 SDP 的时候，就将后面要启用的音频先收集好，先传过去

所以代码也最简单

```javascript
let stream = await navigator.mediaDevices.getUserMedia({ video: false, audio: true })
// 初始化 peerconnection
this.#initPc(turnInfo)
stream.getTracks().forEach(track => {
  this.#pc.addTrack(track, stream)
})
```

这样子的好处就是，因为我一开始就收集音轨信息，并且在 SDP 交换的时候，就会带上本客户端的 audio track 了。 所以 source 端其实就可以得到这个 track 了， 直接取出来就可以了(`ontrack` 监听得到)。

然后对于客户端来说， 用户没有真正点击之前，可以设置 muted 静音， 等用户真的要语音之后， 再把 muted 去掉就行了。

当然坏处也很明显， 用户本来只是想投屏， 但是却要求要开启麦克风权限， 这个体验肯定是不好的(第一次授权的时候，要用户手动点击允许才行)

### 2. 临时要用语音的时候，再去开启麦克风权限
第一种的体验是不太好， 麦克风权限获取的时候，肯定是用户当下要用的时候，才去获取才对，如果用户不需要的话，不能去干扰用户。

所以我们可以这样子做，等用户想语音的时候，再去获取，但是这样子会有一个问题，当我 webrtc 连接之后，如果我要新加 media stream， 是需要重新交换 sdp 信息， 让 sdp 重新带上这个新加的 track， 这样子 source 端才能识别

所以代码会稍微复杂一点，不过只需要第一次添加 media stream， 需要去交换 sdp， 后面的开关就不需要了，直接 replace track 就行了。

```javascript
// target端的 sdp 数据, 用来开启 audio 的时候，重新交换 sdp
#targetSdpData = {}
// 本次 webrtc 连接是否有开启过 audio
#hadOpenAudio = false
```
```javascript
openLocalAudio(){
  return new Promise((resolve, reject) => {
    this.addLocalAudioTrack(!this.#hadOpenAudio).then( stream =>{
      //这时候要重新交换sdp 的内容，不然只有 addtrack 是不行
      // 当然一次 webrtc 连接只需要交换过一次就行了
      if(this.#hadOpenAudio){
        resolve(stream)
      }else{
        this.#resendSdp().then(() => {
          this.#hadOpenAudio = true
          resolve(stream)
        }, err => reject(err))
      }
    }, err => {
      reject(err)
    })
  })
}

// 重新交互 sdp， client 开启麦克风的时候， 需要将麦克风的 sender 信息重新交互给 target, 不然 track 没法同步过去
async #resendSdp(){
  const SDPAnswer = await this.setSdp(this.#targetSdpData)
  const setRemoteSDPResult = await this.#mqttPub('toTarget', 'webrtc.setRemoteSDP', [SDPAnswer.sdp])
}

// 设置 sdp
setSdp(offerSDP) {
  const sdpConstraints = {
      'mandatory': {
          'OfferToReceiveAudio': true,
          'OfferToReceiveVideo': true
      }
  }

  const RTCSessionDescription = window.RTCSessionDescription || window.mozRTCSessionDescription || window.webkitRTCSessionDescription
  const offer = new RTCSessionDescription({ "sdp": offerSDP, "type": "offer" })

  let sdpAnswer = null
  return this.#pc.setRemoteDescription(offer).then(() => {
      logger.debug(`[PeerConnection]: setRemoteDescription success`)
      return this.#pc.createAnswer(sdpConstraints)
  }).then((answer) => {
      logger.debug(`[PeerConnection]: createAnswer success`)
      sdpAnswer = answer
      return this.#pc.setLocalDescription(answer)
  }).then(() => {
      logger.debug(`[PeerConnection]: setLocalDescription success`)
      return sdpAnswer
  }).catch((error) => {
      logger.error(`[PeerConnection]: set sdp err: ${error}, sdp: ${offerSDP}`)
      throw error
  })
}

 /**
 * 开启本地音频
 * @returns {Promise<void>}
 */
addLocalAudioTrack(isFirst) {
  return new Promise((resolve, reject) => {
    // doc: https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getUserMedia
    navigator.mediaDevices.getUserMedia({ video: false, audio: true }).then( stream => {
      // https://github.com/webrtc/samples/blob/gh-pages/src/content/peerconnection/upgrade/js/main.js
      // 从这边来看的话，如果之前没有开启过 audio track，就连接上 webrtc 的话， 那么 sdp 的 sender 是不会包含 audio 信息，所以这时候客户端添加 addTrack 的话是没有效果的。 要重新同 target 端进行 sdp 的交换才行
      // 除非一开始在 sdp 交换之前就 addtrack。 否则在连接之后，再进行 addtrack 就会有这个问题
      let audioTrack = stream.getAudioTracks()[0]
      if(isFirst){
        // 第一次直接 add 进去，生成 sender
        this.#localAudioTrackSender = this.#pc.addTrack(audioTrack, stream)
      }else{
        // 如果不是第一次，就用 replace 的方式。 不然反复开启，每次都 addtrack 的话，就会导致第二次开启的时候，声音传不过去
        this.#localAudioTrackSender?.replaceTrack(audioTrack)
      }
      resolve(stream)
    }).catch(e => {
      // 错误列表如下
      /**
       * AbortError  -> 硬件阻止
       * NotAllowedError -> 用户拒绝
       * NotFoundError -> 找不到麦克风
       * NotReadableError -> 硬件不可读
       * SecurityError -> 安全问题
       * TypeError -> 其他类型错误
       */
      switch(e.name) {
        case "NotFoundError":
          logger.error(`[getUserMedia] error: Unable to open your call because no camera and/or microphone were found.`)
          break
        case "NotAllowedError":
          logger.error(`[getUserMedia] error: because reject!!!!!`)
          break
        default:
          logger.error(`[getUserMedia] error: Error opening your camera and/or microphone: ${e.message}`)
          break;
      }
      logger.error(`[getUserMedia] error: ${e}`)
      reject({ code: -2,  msg: e })
    })
  })
}

/**
 * 关闭本地音频
 */
stopLocalAudioTrack() {
  this.#localAudioTrackSender?.track?.stop()
}

```
然后这个 `this.#targetSdpData` 其实就是之前 source 返回的 sdp 数据，这个要保存起来，后面生成新的 answer 给 source 端。

client 端这样子处理就行了，不过 source 那边也要处理，因为在 webrtc 连接成功的时候再去重新 setRemoteSDP 是会报这个错误的:
```sql
Failed to set remote answer sdp: Called in wrong state: stable
```
具体代码就是收到 client 端的 setRemoteSDP 指令的时候，然后重置一下本地的 `localDescription`，然后再去设置 `setRemoteDescription`
```javascript
this.mqttModel.onMsg('webrtc.setRemoteSDP', data => {
  this.mqttModel.pub(`${this.#commandPrefix}toClient`, {
    id: data?.id
  }, {
    useAes: false
  })
  if(this.#connectStatus === SourceWebRtcProcess.CONNECT_STATUS.COMPLETE){
    // 连接结束的交互, 这时候要重新设置本地的 sdp，才不会报这个错 : Failed to set remote answer sdp: Called in wrong state: stable
    this.#pc.setLocalDescription(this.#pc.localDescription).then(result => {
      const offer = new RTCSessionDescription({ "sdp": data.params[0], "type": "answer" })
      this.#pc.setRemoteDescription(offer).then(() => {
        // 这时候就返回 ice
      })
    })
  }else{
    // 连接过程中
    // 接下来处理写入 answer        
    const offer = new RTCSessionDescription({ "sdp": data.params[0], "type": "answer" })
    this.#pc.setRemoteDescription(offer).then(() => {
      // 这时候就返回 ice
    })
  }
})
```
这样子就可以实现，刚开始 webrtc 连接的时候，不启用 audio track， 后面临时要用的时候，再临时添加。

缺点就是 source 也要相应的调整代码

### 3. 刚开始添加一段无意义的音轨，后面直接替换
还有一种更好的交互方式，就是不需要再去重新交换 sdp，而是刚开始的时候，client 就把音轨加进去了， 只不过是一段无意义的空白音轨， 后面真的启用麦克风权限的时候，再将这一段音轨替换成麦克风的音轨就行了。

整个过程都不需要再去交换 sdp， 具体代码:
```javascript
// 初始化 peerconnection
this.#initPc(turnInfo)

// 这边可以考虑初始化一段空白的音轨，然后添加到 pc，这样子，后面只需要 replacetrack 的操作就可以了
// 这边将一段很短的无声 mp3 转换为 base64 编码，然后直接播放，这样子就不需要再去加载音频资源
const url = URL.createObjectURL(dataURLtoBlob(`data:audio/mpeg;base64,SUQzBAAAAAACBFRYWFgAAAASAAADbWFqb3JfYnJhbmQATTRBIABUWFhYAAAAEwAAA21pbm9yX3ZlcnNpb24ANTEyAFRYWFgAAAAcAAADY29tcGF0aWJsZV9icmFuZHMAaXNvbWlzbzIAVElUMgAAACAAAAPhhJDhhaHhhrwh4pmhX+Wui+aXu+a1qV8oTUlOTykAVFBFMQAAAAsAAAPosK3lt6fmnpcAVENPTQAAAAsAAAPosK3lt6fmnpcAVEFMQgAAABQAAAPosK3lt6fmnpfnmoTkuJPovpEAVERSQwAAAAYAAAMyMDIyAFRTU0UAAAAPAAADTGF2ZjU4Ljc2LjEwMAAAAAAAAAAAAAAA//tQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAASW5mbwAAAA8AAAAJAAAEfABFRUVFRUVFRUVFRVxcXFxcXFxcXFxcdHR0dHR0dHR0dHSLi4uLi4uLi4uLi6KioqKioqKioqKiurq6urq6urq6urrR0dHR0dHR0dHR0ejo6Ojo6Ojo6Ojo//////////////8AAAAATGF2YzU4LjEzAAAAAAAAAAAAAAAAJAYbAAAAAAAABHw6c4BSAAAAAAAAAAAAAAAAAAAAAP/7EGQAD/AAAGkAAAAIAAANIAAAAQAAAaQAAAAgAAA0gAAABExBTUUzLjEwMFVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVMQU1FMy4xMDBVVVVVVVVVVVVVVVVVVVVVVVVVVVVV//sSZCIP8AAAaQAAAAgAAA0gAAABAAABpAAAACAAADSAAAAEVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVUxBTUUzLjEwMFVVVVVVVVVVVVVVVVVVVVVVVVVVVVVV//sQZESP8AAAaQAAAAgAAA0gAAABAAABpAAAACAAADSAAAAEVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVX/+xJkZo/wAABpAAAACAAADSAAAAEAAAGkAAAAIAAANIAAAARVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVX/+xBkiQ/wAABpAAAACAAADSAAAAEAAAGkAAAAIAAANIAAAARVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVf/7EmSrD/AAAGkAAAAIAAANIAAAAQAAAaQAAAAgAAA0gAAABFVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVf/7EGTNj/AAAGkAAAAIAAANIAAAAQAAAaQAAAAgAAA0gAAABFVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVV//sSZN0P8AAAaQAAAAgAAA0gAAABAAABpAAAACAAADSAAAAEVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVV//sQZN2P8AAAaQAAAAgAAA0gAAABAAABpAAAACAAADSAAAAEVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVU=`))
let audio = new Audio(url)
let audioStream = audio.mozCaptureStream ? audio.mozCaptureStream() : audio.captureStream()
await audio.play()
// 这边一定要等待开始播放，或者播放结束后，才会有 audio track 
let audioTrack = audioStream.getAudioTracks()[0]
this.#pc.addTrack(audioTrack, audioStream)
```
简单的来说，就是初始化 peerconnection 之后，将一段本地播放完的 audio stream 的 audio track 添加到 peerconnection 对象，所以接下来的 sdp 交换，就会有这个 audio track 的信息。

> 这边的 mp3 音频文件，最好是那种很小的，时间非常短的白噪音，让用户没办法感知到

然后只需要在开启麦克风的时候，replace 这个 audio track 就行了
```javascript
// doc: https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getUserMedia
navigator.mediaDevices.getUserMedia({ video: false, audio: true }).then( stream => {
  let audioTrack = stream.getAudioTracks()[0]
  this.#localAudioTrackSender = this.#pc.getSenders().find(s => s.track.kind === audioTrack.kind)
  this.#localAudioTrackSender.replaceTrack(audioTrack)
})
```

不过这种方式会有一个浏览器兼容问题，就是 safari 浏览器并没有 `captureStream` 这个方法， 所以在 safari 浏览器会失败。

所以如果 safari 浏览器要支持的话，只能用 第一种或者第二种 方式来处理。

## 浏览器作为投屏端(source)启用麦克风将语音传给对方的方式
参照上一种的第三方方式，当浏览器作为投屏端的时候，也是在建立 webrtc 的时候，初始化 peerconnection， 就将一个本地的 audio track 添加进去
```javascript
// 获取分享桌面
async getDisplayMedia() {
  // 获取桌面
  /**
   * 相关文档
   * https://developer.mozilla.org/en-US/docs/Web/API/Screen_Capture_API/Using_Screen_Capture
   * https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getDisplayMedia
   */
  let mediaStream = await navigator.mediaDevices.getDisplayMedia({
    audio: false,
    video: {
        frameRate: 20
    }
  })
  let stopEventHandle = () => {
    // 同时触发 media stop 事件
    this.emit("mediaStop", null)
  }
  // 这边要监听 stop 事件， 这个用户直接点击外面的停止共享屏幕按钮的监听
  mediaStream.getVideoTracks()[0].removeEventListener('ended', stopEventHandle)
  mediaStream.getVideoTracks()[0].addEventListener('ended', stopEventHandle)
  return mediaStream
}
```
```javascript
// 判断是否支持相关api。
if (!navigator.mediaDevices || !navigator.mediaDevices.getDisplayMedia) {
  alert('getDisplayMedia is not supported!')
  return
}

// 获取分享桌面
this.#mediaStream = await this.getDisplayMedia()

// 刚开始的时候，这边可以创建一个基本上等于空白的 audio track 来占位， 后面直接就可以 replacetrack 
const url = URL.createObjectURL(dataURLtoBlob(`data:audio/mpeg;base64,SUQzBAAAAAACBFRYWFgAAAASAAADbWFqb3JfYnJhbmQATTRBIABUWFhYAAAAEwAAA21pbm9yX3ZlcnNpb24ANTEyAFRYWFgAAAAcAAADY29tcGF0aWJsZV9icmFuZHMAaXNvbWlzbzIAVElUMgAAACAAAAPhhJDhhaHhhrwh4pmhX+Wui+aXu+a1qV8oTUlOTykAVFBFMQAAAAsAAAPosK3lt6fmnpcAVENPTQAAAAsAAAPosK3lt6fmnpcAVEFMQgAAABQAAAPosK3lt6fmnpfnmoTkuJPovpEAVERSQwAAAAYAAAMyMDIyAFRTU0UAAAAPAAADTGF2ZjU4Ljc2LjEwMAAAAAAAAAAAAAAA//tQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAASW5mbwAAAA8AAAAJAAAEfABFRUVFRUVFRUVFRVxcXFxcXFxcXFxcdHR0dHR0dHR0dHSLi4uLi4uLi4uLi6KioqKioqKioqKiurq6urq6urq6urrR0dHR0dHR0dHR0ejo6Ojo6Ojo6Ojo//////////////8AAAAATGF2YzU4LjEzAAAAAAAAAAAAAAAAJAYbAAAAAAAABHw6c4BSAAAAAAAAAAAAAAAAAAAAAP/7EGQAD/AAAGkAAAAIAAANIAAAAQAAAaQAAAAgAAA0gAAABExBTUUzLjEwMFVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVMQU1FMy4xMDBVVVVVVVVVVVVVVVVVVVVVVVVVVVVV//sSZCIP8AAAaQAAAAgAAA0gAAABAAABpAAAACAAADSAAAAEVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVUxBTUUzLjEwMFVVVVVVVVVVVVVVVVVVVVVVVVVVVVVV//sQZESP8AAAaQAAAAgAAA0gAAABAAABpAAAACAAADSAAAAEVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVX/+xJkZo/wAABpAAAACAAADSAAAAEAAAGkAAAAIAAANIAAAARVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVX/+xBkiQ/wAABpAAAACAAADSAAAAEAAAGkAAAAIAAANIAAAARVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVf/7EmSrD/AAAGkAAAAIAAANIAAAAQAAAaQAAAAgAAA0gAAABFVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVf/7EGTNj/AAAGkAAAAIAAANIAAAAQAAAaQAAAAgAAA0gAAABFVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVV//sSZN0P8AAAaQAAAAgAAA0gAAABAAABpAAAACAAADSAAAAEVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVV//sQZN2P8AAAaQAAAAgAAA0gAAABAAABpAAAACAAADSAAAAEVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVU=`))
let audio = new Audio(url)
let audioStream = audio.mozCaptureStream ? audio.mozCaptureStream() : audio.captureStream()
await audio.play()
// 这边一定要等待开始播放，或者播放结束后，才会有 audio track 
let audioTrack = audioStream.getAudioTracks()[0]

this.#mediaStream.addTrack(audioTrack)

this.#initPc(turnInfo)

// 添加本地视频轨到 peerConnection 之中
this.#mediaStream.getTracks().forEach(track => {
  this.#pc().addTrack(track, this.#mediaStream)
})
```

然后真的要开启的时候，再去 replace track 就行了
```javascript
  // 打开本地的 audio， 也就是麦克风
  openLocalAudio(){
    navigator.mediaDevices.getUserMedia({ video: false, audio: true }).then( stream => {
        let audioTrack = stream.getAudioTracks()[0]
        let sender = this.rtcModel.getPc().getSenders().find(s => s.track.kind === audioTrack.kind)
        sender.replaceTrack(audioTrack)
    })
  }


  /**
 * 关闭本地音频
 */
 stopLocalAudio() {
  let sender = this.#pc.getSenders().find(s => s.track.kind === "audio")
  sender?.track?.stop()
}
```

## 公共方法
上面会用到两个公共方法:
1. 一个是将 音频文件，比如 mp3 文件，转换为 base64 格式字符串, 事先转好
```javascript
<input type="file" id="fileInput">
 <script>
    　　var fileInput = document.querySelector('#fileInput');
        fileInput.onchange = function () {
            var file = this.files[0];
            var reader = new FileReader();
            reader.readAsDataURL(file);
            reader.onload = function () {
                var data = reader.result;
                console.log('data', data);
            };
        };
</script>
```

2. 将 base64 格式的音频转换为 blob 格式，然后再转换为 url，这样子就可以直接应用
```javascript
function dataURLtoBlob(dataurl) {
  let arr = dataurl.split(','),
      mime = arr[0].match(/:(.*?);/)[1],
      bstr = atob(arr[1]),
      n = bstr.length,
      u8arr = new Uint8Array(n)
  while (n--) {
      u8arr[n] = bstr.charCodeAt(n)
  }
  return new Blob([u8arr], { type: mime })
}
```

这样子就可以不用再引入一个 音频资源 了，也就少了一次 http 加载请求

---
参考资料:
- [webrtc demo](https://webrtc.github.io/samples/)
- [Upgrade a call and turn video on](https://webrtc.github.io/samples/src/content/peerconnection/upgrade/)










