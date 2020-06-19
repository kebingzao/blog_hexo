---
title: 初试 webrtc SFU 开源框架 - Kurento
date: 2020-06-19 16:51:43
tags: webrtc
categories: webrtc相关
---
## 前言
最近有在看多方 webrtc 的解决方案: {% post_link webrtc-multi %}, 后面觉得 SFU 应该是不错的解决方案，而目前开源的 SFU 框架有好几个:
- [Janus](https://janus.conf.meetecho.com/)
- [Jitsi](https://meet.jit.si/)
- [Kurento](http://www.kurento.org/)
- [mediasoup](https://mediasoup.org/)
- [Medooze](http://www.medooze.com/)

后面打算一个一个用下，看看效果， 所以本次先用 Kurento 这个框架看看效果。
<!--more-->
## 安装 Kurento
Kurento 安装其实非常简单, 具体文档: [Installation Guide](https://doc-kurento.readthedocs.io/en/6.13.2/user/installation.html), 我们采用了 docker 容器的安装方式, 镜像地址: [Kurento Media Server](https://hub.docker.com/r/kurento/kurento-media-server), 教程很简单，直接按照上面来操作就行了

### 1. 拉取镜像
```text
docker pull kurento/kurento-media-server:latest
```
```text
[root@VM_156_200_centos ~]# docker pull kurento/kurento-media-server:latest
latest: Pulling from kurento/kurento-media-server
e92ed755c008: Pull complete
...
a8e4c7815847: Pull complete
Digest: sha256:4b27d8f6cf853c629c14682b249f4ea7f9e186f29619bdbabde2c9564a87dc45
Status: Downloaded newer image for kurento/kurento-media-server:latest
```

### 2. 启动容器
```text
docker run --name kms -d -p 8888:8888 \
    kurento/kurento-media-server:latest
```
```text
[root@VM_156_200_centos ~]# docker run --name kms -d -p 8888:8888 \
>     kurento/kurento-media-server:latest
f596838cdaaf8a1ae7cea81c76d9d3bd9b85038738e4243bf3f4cac095980288
```
这时候可以用 `docker ps` 查看是否存在:
```text
[root@VM_156_200_centos ~]# docker ps
CONTAINER ID        IMAGE                                 COMMAND                  CREATED             STATUS                             PORTS                    NAMES
f596838cdaaf        kurento/kurento-media-server:latest   "/entrypoint.sh"         28 seconds ago      Up 26 seconds (health: starting)   0.0.0.0:8888->8888/tcp   kms
```
### 3. 测试服务是否正常
因为这个是一个 ws 的服务，所以用 curl 测试的时候，要加上 upgrade 头部:
```text
[kbz@centos156 test-nms.airdroid.com]$  curl     --include     --header "Connection: Upgrade"     --header "Upgrade: websocket"     --header "Host: 127.0.0.1:8888"     --header "Origin: 127.0.0.1"     http://127.0.0.1:8888/kurento
HTTP/1.1 500 Internal Server Error
Server: WebSocket++/0.7.0
```
虽然报 500 错误，但是按照文档说的，这个是正常的:
{% blockquote kurento  "https://hub.docker.com/r/kurento/kurento-media-server" %}
To check whether KMS is up and listening for connections, use the following command:

$ curl \
    --include \
    --header "Connection: Upgrade" \
    --header "Upgrade: websocket" \
    --header "Host: 127.0.0.1:8888" \
    --header "Origin: 127.0.0.1" \
    http://127.0.0.1:8888/kurento
You should get a response similar to this one:

HTTP/1.1 500 Internal Server Error
Server: WebSocket++/0.7.0
Ignore the "Server Error" message: this is expected, and it actually proves that KMS is up and listening for connections.

The health checker script inside this Docker image does something very similar in order to check if the container is healthy.
{% endblockquote %}

当然我们也可以用第三方的 ws 客户端连一下，是可以连接成功的:

![](1.png)

也可以通过查看进程来判断 kms 的进程是否在:
```text
[root@VM_156_200_centos ~]# ps -fC kurento-media-server
UID        PID  PPID  C STIME TTY          TIME CMD
root     26964 26949  0 13:39 ?        00:00:00 /usr/bin/kurento-media-server
```
配置文件是在容器中的: `/etc/kurento/kurento.conf.json`, 先不用改配置文件。保持默认即可。

## 安装 stun/turn
要完整的跑起来，除了 KMS 安装成功之后， turn server 也要安装，这个是 Kurento 官网推荐的安装文档: [STUN/TURN server install](https://doc-kurento.readthedocs.io/en/6.13.2/user/installation.html#installation-stun-turn)

具体步骤这边不细说了，之前文章就说过 coturn 的安装了: {% post_link turn-install %}, 安装之后可以到 [WebRTC samples Trickle ICE](https://webrtc.github.io/samples/src/content/peerconnection/trickle-ice/) 这个站点测试, 这个在之前 coturn 权限校验的时候 也说过这个 ice 的测试站点: {% post_link turn-verify %}

## hello-world demo
既然安装完之后，那么要测试是否安装成功，写个 hello world demo 呗，官方也有完整的[教程文档](https://doc-kurento.readthedocs.io/en/6.13.2/user/tutorials.html), 我们搞个 hello world 级别的先试跑一下，先去 github 将 [教程文档 demo 实例](https://github.com/Kurento/kurento-tutorial-node) 拉下来。

这个教程有很多的实例，我们先用这个 hello world 的实例跑一下，看是否正常(服务端是 node 程序，保证 node 版本在 8.x 以上，并且有全局安装 bower):

![](2.png)

整个 hello world 的教程文档如下: [Node.js - Hello world](https://doc-kurento.readthedocs.io/en/6.13.2/tutorials/node/tutorial-helloworld.html), 照着操作就行了:
```text
git clone https://github.com/Kurento/kurento-tutorial-node.git
cd kurento-tutorial-node/kurento-hello-world
git checkout master
npm install
cd static
bower install --allow-root
cd ..
npm start
```
项目跑起来了:
```text
F:\airdroid_code\github\kurento-tutorial-node\kurento-hello-world>npm start

> kurento-hello-world@6.14.0 start F:\airdroid_code\github\kurento-tutorial-node\kurento-hello-world
> node server.js

(node:64572) Warning: N-API is an experimental feature and could change at any time.
Kurento Tutorial started
Open https://localhost:8443/ with a WebRTC capable browser
```
这边要注意一个细节，因为我安装 kerento 的服务器(腾讯云) 和 启动 node服务的机器(本机) 不是同一台。所以 server.js 中的 ws_uri 要改成对应的 kurento 服务器的地址:
```text
ws_uri: 'ws://119.29.xx.28:8888/kurento'
```
当然教程也提供一种不改代码，直接启动的时候设置配置项，这样子也可以:
```text
npm start -- --ws_uri=ws://kms_host:kms_port/kurento
```
这两种都可以。

但是跑起来之后，只发现本地有图像，远程的没有 (因为没有真正的摄像头，用虚拟摄像头代替):

![](3.png)

而人家的示例，远程的图像也是有的:

![](4.png)

后面想了一下，发现是因为 Kurento 服务 和 我的浏览器 这两个 peer 端根本不在同一个局域网， 所以要借助 turn server 来转发。 但是我之前安装完 turn server 之后，并没有在 kerento 上进行配置。而对应的 demo 的 server.js 也没有连接 turn server 的代码。

虽然 demo 没有配置 turnserver ，但是教程文档确实有写， 看了一下文档: [How to configure STUN/TURN?](https://doc-kurento.readthedocs.io/en/6.13.2/user/faq.html#faq-stun-configure), 发现有两种配置方式:
- 一种是直接在 kms 服务的配置文件 `/etc/kurento/modules/kurento/WebRtcEndpoint.conf.ini` 中直接配置默认的 turn server 转发，然后重启 kms 服务
- 另一种是在浏览器端连接 kms 的时候，动态指定 turn server 地址，这种方式也是可以的，具体 API 文档: [setTurnUrl](https://doc-kurento.readthedocs.io/en/latest/_static/client-jsdoc/module-elements.WebRtcEndpoint.html#setTurnUrl)

我用的是第二种，也就是连接的时候，动态指定 turn server 的地址，代码修改如下: server.js 加上一行代码就行了, 原先的代码是这样子:
```text
createMediaElements(pipeline, ws, function(error, webRtcEndpoint) {
  if (error) {
      pipeline.release();
      return callback(error);
  }

  if (candidatesQueue[sessionId]) {
      while(candidatesQueue[sessionId].length) {
          var candidate = candidatesQueue[sessionId].shift();
          webRtcEndpoint.addIceCandidate(candidate);
      }
  }

  connectMediaElements(webRtcEndpoint, function(error) {
      if (error) {
          pipeline.release();
          return callback(error);
      }

      webRtcEndpoint.on('OnIceCandidate', function(event) {
          var candidate = kurento.getComplexType('IceCandidate')(event.candidate);
          ws.send(JSON.stringify({
              id : 'iceCandidate',
              candidate : candidate
          }));
      });

      webRtcEndpoint.processOffer(sdpOffer, function(error, sdpAnswer) {
          if (error) {
              pipeline.release();
              return callback(error);
          }

          sessions[sessionId] = {
              'pipeline' : pipeline,
              'webRtcEndpoint' : webRtcEndpoint
          }
          return callback(null, sdpAnswer);
      });

      webRtcEndpoint.gatherCandidates(function(error) {
          if (error) {
              return callback(error);
          }
      });
  });
});
```
只要在创建 webRtcEndpoint 并且准备开始连接 kms 的时候，设置 turn url 就行了:
```text
createMediaElements(pipeline, ws, function(error, webRtcEndpoint) {
    if (error) {
        pipeline.release();
        return callback(error);
    }

    // 设置 turnserver
    webRtcEndpoint.setTurnUrl("username:pwd@193.xxx.xxx.33:3478", function(data){
        if (candidatesQueue[sessionId]) {
            while(candidatesQueue[sessionId].length) {
                var candidate = candidatesQueue[sessionId].shift();
                webRtcEndpoint.addIceCandidate(candidate);
            }
        }

        connectMediaElements(webRtcEndpoint, function(error) {
            if (error) {
                pipeline.release();
                return callback(error);
            }

            webRtcEndpoint.on('OnIceCandidate', function(event) {
                var candidate = kurento.getComplexType('IceCandidate')(event.candidate);
                ws.send(JSON.stringify({
                    id : 'iceCandidate',
                    candidate : candidate
                }));
            });

            webRtcEndpoint.processOffer(sdpOffer, function(error, sdpAnswer) {
                if (error) {
                    pipeline.release();
                    return callback(error);
                }

                sessions[sessionId] = {
                    'pipeline' : pipeline,
                    'webRtcEndpoint' : webRtcEndpoint
                }
                return callback(null, sdpAnswer);
            });

            webRtcEndpoint.gatherCandidates(function(error) {
                if (error) {
                    return callback(error);
                }
            });
        });

    });
});
```
这样子就可以确保在连接 kms 的时候，已经设置好 turn server 了。重新试了一下，

![](5.png)

这时候远程图像也有了。说明 turn 转发成功了。

## one2many-call demo
hello world demo 演示成功了，至少证明 Kurento 可以作为另一个 peer 来进行 webrtc P2P 连接。 接下来我们就测试一下他的主要功能，就是一对多的转发。 刚好教程里面也有一个 一对多的直播的 [demo](https://github.com/Kurento/kurento-tutorial-node/tree/master/kurento-one2many-call), 具体代码不细讲了，直接看代码就行了，主要讲一下这个 demo 要实现的几个功能:
1. 这个是一个直播间，点击 [Presenter] 就是主持人，然后这时候主持人先进去，然后右边就是图像(这个图像其实就是本地图像，也就是 local stream， 所以他不会卡，并且清晰)
2. 这时候直播间开启了，其他人进来，点击 [Viewer], 右边就会出现图像 (这个图像就是 remote stream，通过 turn server 转发的，所以有时候会卡，也会模糊)
3. 观看的人可以有很多个，所以这个一对多，其实就是一个 主播 对应 多个观众。 观众点击 [Stop] 按钮就是离开直播间，不影响主播，也不影响其他观众。
4. 同时在已经有主播存在的情况下，观众进来只能点 [Viewer] 按钮进行观看，或者点击 [Stop] 离开直播间。 如果点击 [Presenter] 按钮是没有反应的，会显示当前已经有一个主播了。
5. 主播如果点击 [Stop] 按钮 就是结束直播，所有的在线的观众都会退出直播间 (体现在右边的图像没有了)
6. 因为是 webrtc 技术，所以主播在说话的时候，所有的观众都是可以听到的。(当然观众说话，主播听不到)

这个是一个很简单的直播间示例，代码也不复杂，在原来的 server.js 做一下调整，在生成 主播节点和观众节点的时候，记得把设置 turn url 的代码放上去就行了，其他都不用修改:
```text
presenter.pipeline.create('WebRtcEndpoint', function(error, webRtcEndpoint) {
		if (error) {
			stop(sessionId);
			return callback(error);
		}

    // 设置 turnserver
    webRtcEndpoint.setTurnUrl("username:pwd=@193.xx.xx.33:3478", function(data) {
        // todo
```
示例效果就是，当主播开启之后，接下来就是一堆的观众进来，进行观看:

![](6.png)

需要说明的一点就是，在测试的时候，有发现几个问题:
1. 当观看的人数越来越多的时候，观众的界面会变得比较卡，也比较模糊，但是当人数比较少的时候，就会比较流畅和清晰，这个应该跟 kms 所在的服务器的性能有关系，因为转发不过来
2. 当观看的人数达到一定个数，比如 40+， 我的 node 程序就会崩掉，我估计也跟我的本机的性能有关系
3. 当观看到达一定时间的时候，比如 10 分钟，就会有概率出现除了主播还在线，其他观众全部掉线的情况(没有图片了), 这个原因还在找, 还没有细究


## Kurento 的日志
Kurento 是用 docker 安装的，但是可以通过 docker logs 将日志输出到宿主机:
```text
[root@VM_156_200_centos ~]# docker logs --follow kms >"kms-$(date '+%Y%m%dT%H%M%S').log" 2>&1
```
这时候在执行目录下，就会有一个 log 文件了:
```text
 kms-20200618T134019.log
```
然后就可以查看 log 了:
```text
[root@VM_156_200_centos ~]# tail -f kms-20200618T134019.log
1:02:19.297836682     1 0x7f273c049b80 INFO    KurentoWebRtcEndpointImpl WebRtcEndpointImpl.cpp:571:WebRtcEndpointImpl: TURN server not found in config; remember that NAT traversal requires STUN or TURN
1:02:19.426399932     1 0x7f274000ee30 INFO         basertpendpoint kmsbasertpendpoint.c:1118:kms_base_rtp_endpoint_start_transport_send:<kmswebrtcendpoint5> Media 'video' has REMB
```

## 总结
总的来说， Kerento 这个 SFU 的框架确实可以实现多方 webrtc 的情况，当然更深层次的问题还是有很多(包括性能优化和性能瓶颈)， 不过从本次体验来看，无论是安装，配置还是开发应该都是比较容易的，不会有太多的理解成本。 接下来就再试试其他几个 SFU 的框架。
