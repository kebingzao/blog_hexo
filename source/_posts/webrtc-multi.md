---
title: 多方 webrtc 的选择
date: 2020-06-19 16:20:27
tags: webrtc
categories: webrtc相关
---
## 前言
我们通常用的 webrtc 技术其实是一对一的 call，也就是 peer to peer。

![](1.png)

ClientA 和 ClientB 如果能够顺利建立 P2P 的连接，则可直接通过 P2P 互相交换数据。如果由于某些网络环境原因，无法成功打通 P2P 连接的话，则可以通过一台 TURN Server 来中转数据给对方。

但是有时候我们会有一对多的情况，比如视频直播之类的(视频直播还有另一种技术: 推流 `rtmp - cdn`)， 甚至是多对多的情况，比如多方会议。 这时候是没法满足的。所以我们需要对 webrtc 技术延伸出 一对多，甚至 多对多的 架构形式。结合网上的资料 (以下的资料和总结大量参考了文章底部列出的参考资料)。 要实现多方的 webrtc，有以下三种选择：
- Mesh
- MCU
- SFU

<!--more-->
接下来一一介绍一下。

## [Mesh](https://webrtc.org.cn/20180805-codecs-webrtc-p2p/)
有了WebRTC,对于为了建立多方视频通话而向连接中添加不止一个用户这件事你有很多种选择。Mesh可能是其中最明显的解决方法。就像你已经知道的，为了使连接成为可能，每一个使用 RTCPeerConnectionAPI 的 peer 必须创建一个连接对象。这个连接对象加入了所有相关信息，例如视频和音频流。

API接着使用中间发信过程传递的所有数据建立连接。

我们接着对于所有被加入通话的 peers 重复这个过程。换句话说，我们对于每一个建立了另外的 RTCPeerConnection 对象。

因此，如果我们使用这种策略向上面的图片中加入另一个peer的话，总体效果将是这样。

![](2.png)

现在，让我们看看如果我们添加更多的 peers 会发生什么。当用户数量保持增长时，过程和带宽开始过渡消耗，在移动设备上这种情况更明显可见，资源更加有限，在宽带连接方面，通常与一个每月限制使用量的合约有关。

伴随这些，我们可以说 Mesh 是实现多方视频通话的最简单方式，因为它不需要任何基础结构的改变。所有的工作都是在浏览器完成的。唯一的问题就是它只对小数量的用户有效。

![](3.png)

如果我们想要支持更多用户，我们需要寻找另一种策略，一种允许我们混合所有这些由 Mesh 建立的连接的策略，为了避免客户端的大量CPU和带宽消耗。

## [MCU](https://webrtc.org.cn/20180805-codecs-webrtc-mcu/)
对于多方WebRTC一个不错的选择是 MCU (Multipoint Conferencing Unit) 。MCU 表示多点控制单元，又被称为混合，实现多方WebRTC交流的另一种策略。伴随着MCU，想法由使用peer建立连接变为只需要连接到中心服务器，中心服务器反过来发送信息到其它 peers，并且对其它 peers 也是这样。

中心服务器，接收媒体服务器的名字，并掌控处理被发送到 peers 的媒体流和数据。这个过程对于不同的实现方案有所不同，但是可以简化为五步：

![](4.png)

MCU 设备从 peers 接收媒体流，对其进行解码并创建一个布局，之后它对其进行编码最终发送到 peers。现在每一个 peer 只需要在流中发送和接收。这个过程如下图所示：

![](5.png)

通过使用MCU，我们避免了Mesh中的所有问题。即使用户数量增加，这也不会对用户处理能力和带宽产生影响，因为每一个用户只连接到一个peer，媒体服务器。

这意味着每个人都很开心，或者他们真的开心么？我知道一些人对此不开心，这些就是对媒体服务器付费的人。

使用了这种方案，你需要将一台服务器放在中间，一个非常昂贵的服务器，因为它将要掌控处理媒体信息。这个过程消耗大量CPU因为它必须对媒体编解码。

MCU是一个可以解决Mesh中出现的问题的可替代方案，但是花费很高。如果你需要一个在服务器端或客户端花费不高的方案，或许应该尝试另一条路线。

## [SFU](https://webrtc.org.cn/20180805-codecs-sfu-media/)
多方WebRTC选择3 的方案是 SFU (Selective Forwarding Unit)，它表示选择转发单元。SFU背后的想法与MCU相同。它在中间有一台媒体服务器，所有peers向它发送流，唯一不同的是，它不会做繁重的处理，服务器将其引到其它peers，这样它们可以进行任何所需的处理。

这样服务器不需要能支持那么繁重的处理运算，你可以利用MCU提供的好处，同时避免 Mesh hassles 和 MCU 的高代价。 SFU的结构如下图所示：

![](6.png)

在SFU结构中，每个peer向媒体服务器发送自己的流，媒体服务器反过来将流引到其它peers。当然，这增加了客户端的花费，客户端现在必须对媒体流编解码。另外，当用户数量增加时，下载带宽也会增加。

然而，媒体服务器具有中心控制点，它可以控制正在被引导路线的媒体，为了最小化这两个问题的影响。这就是SFU选择性部分起到作用的地方。尽管现在服务器不必进行过多的像编解码之类的处理了，其它任务可以被媒体完成。基本上服务器完成的处理可以被总结为以下三步：

![](7.png)

媒体服务器接收流，选择对它做什么并最终发送它。也有可能决定不发送某一个具体的流，或使用同时联播或SVC来选择性的根据接收端发送具体类型的流。例如，我们决定向具有快速网络连接和高质量计算机的用户发送1080p视频流，但是对使用宽带连接的移动设备用户发送360p视频流。

尽管具有这些缺点，SFU提供了一种结构，它不像 Mesh 那么容易瘫痪，同时比 MCU 的花费低。

## 总结
综上所述， 可以看到 SFU 最适合这种多方 webrtc 的场景，SFU 服务器最核心的特点是把自己 “伪装” 成了一个 WebRTC 的 Peer 客户端，WebRTC 的其他客户端其实并不知道自己通过 P2P 连接过去的是一台真实的客户端还是一台服务器，我们通常把这种连接称之为 P2S，即：Peer to Server。除了 “伪装” 成一个 WebRTC 的 Peer 客户端外，SFU 服务器还有一个最重要的能力就是具备 one-to-many 的能力，即可以将一个 Client 端的数据转发到其他多个 Client 端。
                                
这种网络拓扑结构中，无论多少人同时进行视频通话，每个 WebRTC 的客户端只需要连接一个 SFU 服务器，上行一路数据即可，极大减少了多人视频通话场景下 Mesh 模型给客户端带来的上行带宽压力。
                                
SFU 服务器跟 TURN 服务器最大的不同是，TURN 服务器仅仅是为 WebRTC 客户端提供的一种辅助的数据转发通道，在 P2P 不通的时候进行透明的数据转发。而 SFU 是 “懂业务” 的， 它跟 WebRTC 客户端是平等的关系，甚至 “接管了” WebRTC 客户端的数据转发的申请和控制。 而且SFU 很灵活，必要时候他也可以像 MCU 那样做处理，也可以选择要发送给哪些 peer。

## SFU 开源框架
目前市面上的 SFU 的开源框架主要有以下:

- [Janus](https://janus.conf.meetecho.com/)
- [Jitsi](https://meet.jit.si/)
- [Kurento](http://www.kurento.org/)
- [mediasoup](https://mediasoup.org/)
- [Medooze](http://www.medooze.com/)

## 关于 SFU 负载测试的资料:
关于上面五个 SFU 开源框架的性能测试，下面有最新的测试资料(2020-04), 可以参考一下, 比较稳定的框架，可以支撑同时在线客户端超过 200 多个。

- [转折点——WebRTC SFU负载测试（一）](https://webrtc.org.cn/20200320-sfu1/)
- [转折点——WebRTC SFU负载测试（二）](https://webrtc.org.cn/20200401-sfu2/)
- [转折点——WebRTC SFU负载测试（三）](https://webrtc.org.cn/20200403-sfu3/)

---

## 参考
- [WebRTC　简介](https://mp.weixin.qq.com/s/NsiU8rVYYMbDjVBGZlr3Xg)
- [多方WebRTC选择1：Mesh](https://webrtc.org.cn/20180805-codecs-webrtc-p2p/)
- [多方WebRTC选择2：MCU](https://webrtc.org.cn/20180805-codecs-webrtc-mcu/)
- [多方WebRTC选择3：SFU](https://webrtc.org.cn/20180805-codecs-sfu-media/)
- [一文盘点直播技术中的编解码、直播协议、网络传输与简单实现](https://segmentfault.com/a/1190000016819686)
- [如何使用WebRTC和Kurento媒体服务器,来建立视频会议App(一)](https://webrtc.org.cn/20180817-webrtc-video-js/)
- [如何使用WebRTC和Kurento媒体服务器,来建立视频会议App(二)](https://webrtc.org.cn/20180819-webrtc-js-vedio/)
- [[Tutorial] How to Build a Video Conference Application with WebRTC](https://webrtc.ventures/2018/07/tutorial-build-video-conference-application-webrtc-2/)
- [WebRTC SFU中发送数据包丢失反馈](https://webrtc.org.cn/20190816-webrtc-sfu-rtcp/)
- [架构设计：基于Webrtc、Kurento的一种低延迟架构实现](https://www.jianshu.com/p/ac307371def4)
- [WebRTC 开发实践：为什么你需要 SFU 服务器](https://zhuanlan.zhihu.com/p/56428846)
- [一张图概括淘宝直播背后的前端技术 | 赠送多媒体前端手册](https://juejin.im/post/5edf5a3ae51d45786672c31c)




