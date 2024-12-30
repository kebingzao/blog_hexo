---
title: mqtt 使用 emqx 替代 vernemq，抛弃 redis acl 校验
date: 2023-08-26 14:18:56
tags: 
- mqtt
- emqx
- vernemq
categories: webrtc相关
---
## 前言
我们现在 vernemq 的 redis acl 所造成的主从同步，其实非常消耗资源，不管是当前服务器的资源，还是 redis 从库的资源，尤其是这个 redis 从库比较大的时候，可能就要占用系统的 2G 内存了，这个成本是非常高的。

更不用说在国际网络抖动的情况下，很容易出现主从同步延迟，从而导致 acl 授权失败。

因此综上考虑，我们决定用 [emqx](https://docs.emqx.com/zh/) 来代替 vernemq。

## emqx 几个优点
1. 支持 jwt acl，取缔 vernemq 的 redis acl，从而干掉 redis 的主从同步
2. 版本升级问题，vernemq 我们现在用的版本 `1.9.2` 是最后一个支持二进制安装的开源版本，新的版本如果要开源版的话，那么就要自己编译，否则就要用他们的商业版。 而 emqx 的版本一直都有 docker 的开源版本迭代，而且迭代很快
3. emqx 有一个 dashboard 的面板，可以从 web ui 上看到很多的服务信息，还支持很多的热配置。 而 vernemq 是没有的。
4. Prometheus 统计情况， vernemq 的 Prometheus 统计虽然也有，但是统计的指标没有 emqx 统计的指标多

```text
- 常规指标，包括连接数、主题数和订阅数；
- 消息指标，包括发布和接收的消息数量，以及每秒发布和接收的消息数量；
- 系统指标，包括进程数、CPU 和 Erlang VM 内存等（需要使用 Node Exporter）；
- 数据报文指标，包括连接、发布、接收的报文数量等。
```
<!--more-->

再加上 emqx 在测试服已经跑了一年了，稳定性还不错，相关踩的坑也有处理。所以线上替换的成本其实不高，可以直接平替。

## 具体 emqx 的安装和配置过程
> emqx 的版本，统一以线上的 `v5.3.0` 版本

### 1. docker 安装
> 具体教程 https://www.emqx.io/docs/zh/latest/deploy/install-docker.html

这边要注意几个细节，这几个持久化的数据要挂载到宿主机的目录:
```text
/opt/emqx/data
/opt/emqx/etc
/opt/emqx/log
```

同时为了保证跟之前分配的 ws 的 1889 的端口匹配上，容器中 ws 的 8083 端口在容器启动的时候，要映射成宿主机的 1889 端口。

然后原先的 ssl 和 wss，因为 wss 会由 nginx 进行代理，因此本质上 emqx 不需要起这两个服务，启动的时候，可以起，但是 8084 和 8883 这两个端口，不能对外开放。

对外开放的，依然只有 `1883 (tcp)`, `1889 (ws)`, `443 (wss by nginx)`

因为有 dashboard 的 webui 的端口是 18083， 为了防止被嗅探，这个官方默认端口也要改成一个不容易被知道的，因为只有我们自己用，所以可以改成 18123 端口

所以启动应该是:
```text
docker run -d --name emqx \
  -p 1883:1883 -p 1889:8083 \
  -p 8084:8084 -p 8883:8883 \
  -p 18123:18083 \
  -v $PWD/etc:/opt/emqx/etc \
  -v $PWD/data:/opt/emqx/data \
  -v $PWD/log:/opt/emqx/log \
  emqx/emqx:5.3.0
```

### 2. dashboard 配置

#### 2.1 对 18123 端口做权限，不对外开放
这个 web ui 的 18123 端口，只能允许`内网访问`，并开放对 `Prometheus 采集服务器` 的访问权限就够了，其他都不允许, 哪怕是其他业务服务器也不行。

而且虽然 web 管理后台允许添加多个用户，但是因为没有对用户的管理 ACL 权限，每个人都是管理员，因此除了登录密码，要设置复杂之外。 这个后台的账号密码，运维人员要保管好，不要暴露

#### 2.2 在 dashboard 进行项目初始化配置

##### 2.2.1 开启文件日志 log
默认情况下是 开启控制台日志， 并没有开启文件日志。 但是在生产环境中，我们应该反过来，应该关闭控制台日志，然后开启文件日志

具体操作: 在 `[管理]-[日志]` 中处理， 关闭控制台日志，开启文件日志, 日志的其他配置默认

![](1.png)

相关文档，可以看: [emqx 日志](https://www.emqx.io/docs/zh/latest/observability/log.html)

##### 2.2.2 开启保留消息并设置有效期
emqx 的保留消息默认是不过期的，如果 vernemq 也有用到保留消息的机制的话，那么 emqx 的策略要跟 vernemq 的一致，包含过期时间配置，清理间隔时间等。 因此不管有没有用到保留消息，我们也应该设置保留消息的有效期
> 查了一下，貌似 vernemq 的保留消息也是不过期的，是存在内存的，每次重启都会清掉

具体操作: 在 `[管理]-[MQTT 配置]-保留消息` 中处理，开启并设置为 12 小时后过期

![](2.png)

相关文档，可以看: [emqx 保留消息](https://www.emqx.io/docs/zh/latest/dashboard/retained.html)

##### 2.2.3 设置 jwt 的客户端认证
接下来要添加 jwt 的客户端认证

具体操作: 在 `[访问控制]-[客户端认证]-创建` 添加，`注意 secret 的值要复杂一点`

![](3.png)

相关文档，可以看: [emqx jwt 认证](https://www.emqx.io/docs/zh/latest/access-control/authn/jwt.html)

##### 2.2.4 设置 mqtt 空闲超时时间
接下来要添加设置 mqtt 空闲超时时间
```text
设置连接被断开或进入休眠状态前的等待时间，空闲超时后，
  - 如暂未收到客户端的 CONNECT 报文，连接将断开；
  - 如已收到客户端的 CONNECT 报文，连接将进入休眠模式以节省系统资源。
```
注意：请合理设置该参数值，如等待时间设置过长，可能造成系统资源的浪费

这边不知道原先的 vernemq 默认值的多少，因此还是保持默认的 15s，后面如果有需要再调整，在 `[集群配置]-[MQTT 配置]-通用`

![](4.png)

##### 2.2.5 取消 file acl 的默认允许
emqx 默认在 客户端授权 那边有启用了一个 File 的 acl 策略，主要内容是这样子:
```text
%% 允许用户名开头的 dashboard 的客户端订阅 "$SYS/#" 这个主题
{allow, {username, {re, "^dashboard$"}}, subscribe, ["$SYS/#"]}.

%% 允许来自127.0.0.1 的用户发布和订阅 "$SYS/#" 以及 "#"
{allow, {ipaddr, "127.0.0.1"}, all, ["$SYS/#", "#"]}.

%% 拒绝其他所有用户订阅 "$SYS/#" 和 "#" 主题
{deny, all, subscribe, ["$SYS/#", {eq, "#"}]}.

%% 如果前面的规则都没有匹配到，则允许所有操作
{allow, all}.
```
前三条貌似没有问题，但是第四条的 allow all 肯定是不行的，线上环境要去掉，要改成

![](5.png)

相关文档，可以看: [emqx acl 文件](https://www.emqx.io/docs/zh/latest/access-control/authz/file.html)

##### 2.2.6 禁用缓存并修改默认的授权验证配置
因为我们走的是 jwt 的预设 acl 授权，因此是不需要有授权缓存的，因此可以 disable 掉，并且因为有严格的 acl 权限，因此如果未匹配的话，应该是 deny 操作，并且 deny 操作的时候，~~不选择抛弃本次消息(ignore)，而是直接断开连接(disconnect)~~
> **后面发现在测试过程中，如果 acl 失败的话，直接断开连接的话，很容易会引起客户端的无限重连，因为对于底层代码来说，连接成功之后，就会订阅，如果这时候被服务端断开了，就会再走重连机制，最后导致无限重连尝试，因此这边不能设置为 disconnected，而是要走 ignore，也就是忽略本次的 pub 和 sub， 但是连接不断开**

具体操作: 在 `[访问控制]-[客户端授权]-右上角授权设置` 

所以要改成

![](6.png)

相关文档，可以看: [emqx 授权](https://www.emqx.io/docs/zh/latest/access-control/authz/authz.html#%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6)

##### 2.2.7 监听关闭 tls 和 wss
事实上我们在使用中， emqx 只用到了 tcp 和 ws，ssl 和 wss 不需要，也没有为之配证书，所以直接关掉即可

具体操作: 在 `[集群配置]-[监听器]` 关闭启用 ssl 和 wss

![](7.png)

相关文档，可以看: [emqx 开启 SSL/TLS 连接](https://www.emqx.io/docs/zh/latest/network/emqx-mqtt-tls.html)

##### 2.2.8 关闭遥测功能
说白了，就是 emqx 默认保留一些数据抛送的设置，这个不需要，直接关掉:
```text
EMQ 通过遥测收集有关 EMQX 开源版的使用情况的信息，这种功能旨在为我们提供有关用户、社区的综合信息，以及对 EMQX 开源版使用方式的了解。
与我们共享这些指标可以帮助我们更好地了解您如何使用我们的产品，并可以持续地帮助我们改进产品。
```

具体操作: 在主面板的右上角的设置中，选择关闭即可

![](8.png)

相关文档，可以看: [emqx 遥测](https://www.emqx.io/docs/zh/latest/telemetry/telemetry.html)

##### 2.2.9 不使用 restful API
emqx 提供了非常丰富的 restful API，本质上 dashboard 的操作都是调用 api 的，但是为了安全，我们不单独使用 restful API 来操作，因此我们不在后台去生成用于 restful API 认证的 API 密钥

相关文档，可以看: [emqx REST API](https://www.emqx.io/docs/zh/latest/admin/api.html)


#### 3. 直接设置 hocon 配置文件
如果每一次部署新服务，都要上后台点点点，其实很费时间，还有一种方式，就是当第一台配置完之后，就可以在 `/opt/emqx/data/configs/cluster.hocon` 查看所有的配置文件。

所有在 dashboard 操作所修改的配置，都会存在 `cluster.hocon` 这个配置文件，因此只需要我们安装完之后，直接修改这个配置文件的话，也可以起到在 dashboard 手动修改的效果。并且同步到 dashboard 界面中。

具体文档: [emqx 配置重写](https://www.emqx.io/docs/zh/latest/configuration/configuration.html#%E9%85%8D%E7%BD%AE%E9%87%8D%E5%86%99)

也就是安装完之后，直接修改 `/etc/emqx/data/configs/cluster.hocon` 文件也是可以的，

##### 操作步骤如下：
- 直接拷贝已安装 emqx 服务器的 `cluster.hocon` 文件，因为配置都是一样的
- 拷贝后，如果直接重启，会报 acl 找不到，所以,还需要在 创建  `mkdir -p /etc/emqx/data/authz` 目录，同时拷贝 `acl.conf` 文件，并修改属主属组 `chown -R nginx:nginx acl.conf`
- 最后，重启启动emqx服务

> jwt 的 secret 一定要跟线上一样

> 上面注意一个细节，以上那种方式处理，`authz/acl.conf` 这个文件是不能写的，权限不对， root 才能写，docker 用户写不了，web 后台操作会报 400，刚好省的被误修改。因此这个注意一下即可。

最后附上相关标准配置文件: `cluster.hocon` , `acl.conf`
```text
[kbz@VM-16-5-centos configs]$ cat cluster.hocon
authentication = [
  {
    algorithm = hmac-based
    enable = true
    from = password
    mechanism = jwt
    secret = xxxxx
    secret_base64_encoded = false
    use_jwks = false
    verify_claims {}
  }
]
authorization {
  cache {
    enable = false
    max_size = 32
    ttl = 1m
  }
  deny_action = ignore
  no_match = deny
  sources = [
    {
      enable = true
      path = "data/authz/acl.conf"
      type = file
    }
  ]
}
listeners {
  ssl {
    default {
      acceptors = 16
      access_rules = ["allow all"]
      authentication = []
      bind = "0.0.0.0:8883"
      enable = false
      enable_authn = true
      max_connections = infinity
      mountpoint = ""
      proxy_protocol = false
      proxy_protocol_timeout = 3s
      ssl_options {
        cacertfile = "${EMQX_ETC_DIR}/certs/cacert.pem"
        certfile = "${EMQX_ETC_DIR}/certs/cert.pem"
        ciphers = []
        client_renegotiation = true
        depth = 10
        enable_crl_check = false
        fail_if_no_peer_cert = false
        gc_after_handshake = false
        handshake_timeout = 15s
        hibernate_after = 5s
        honor_cipher_order = true
        keyfile = "${EMQX_ETC_DIR}/certs/key.pem"
        log_level = notice
        ocsp {
          enable_ocsp_stapling = false
          refresh_http_timeout = 15s
          refresh_interval = 5m
        }
        reuse_sessions = true
        secure_renegotiate = true
        verify = verify_none
        versions = [tlsv1.3, tlsv1.2]
      }
      tcp_options {
        active_n = 100
        backlog = 1024
        buffer = 4KB
        high_watermark = 1MB
        keepalive = none
        nodelay = true
        reuseaddr = true
        send_timeout = 15s
        send_timeout_close = true
      }
    }
  }
  tcp {
    default {
      acceptors = 16
      access_rules = ["allow all"]
      authentication = []
      bind = "0.0.0.0:1883"
      enable = true
      enable_authn = true
      max_connections = infinity
      mountpoint = ""
      proxy_protocol = false
      proxy_protocol_timeout = 3s
      tcp_options {
        active_n = 100
        backlog = 1024
        buffer = 4KB
        high_watermark = 1MB
        keepalive = none
        nodelay = true
        reuseaddr = true
        send_timeout = 15s
        send_timeout_close = true
      }
    }
  }
  ws {
    default {
      acceptors = 16
      access_rules = ["allow all"]
      authentication = []
      bind = "0.0.0.0:8083"
      enable = true
      enable_authn = true
      max_connections = infinity
      mountpoint = ""
      proxy_protocol = false
      proxy_protocol_timeout = 3s
      tcp_options {
        active_n = 100
        backlog = 1024
        buffer = 4KB
        high_watermark = 1MB
        keepalive = none
        nodelay = true
        reuseaddr = true
        send_timeout = 15s
        send_timeout_close = true
      }
      websocket {
        allow_origin_absence = true
        check_origin_enable = false
        check_origins = "http://localhost:18083, http://127.0.0.1:18083"
        compress = false
        deflate_opts {
          client_context_takeover = takeover
          client_max_window_bits = 15
          mem_level = 8
          server_context_takeover = takeover
          server_max_window_bits = 15
          strategy = default
        }
        fail_if_no_subprotocol = true
        idle_timeout = 7200s
        max_frame_size = infinity
        mqtt_path = "/mqtt"
        mqtt_piggyback = multiple
        proxy_address_header = x-forwarded-for
        proxy_port_header = x-forwarded-port
        supported_subprotocols = "mqtt, mqtt-v3, mqtt-v3.1.1, mqtt-v5"
      }
    }
  }
  wss {
    default {
      acceptors = 16
      access_rules = ["allow all"]
      authentication = []
      bind = "0.0.0.0:8084"
      enable = false
      enable_authn = true
      max_connections = infinity
      mountpoint = ""
      proxy_protocol = false
      proxy_protocol_timeout = 3s
      ssl_options {
        cacertfile = "${EMQX_ETC_DIR}/certs/cacert.pem"
        certfile = "${EMQX_ETC_DIR}/certs/cert.pem"
        ciphers = []
        client_renegotiation = true
        depth = 10
        fail_if_no_peer_cert = false
        handshake_timeout = 15s
        hibernate_after = 5s
        honor_cipher_order = true
        keyfile = "${EMQX_ETC_DIR}/certs/key.pem"
        log_level = notice
        reuse_sessions = true
        secure_renegotiate = true
        verify = verify_none
        versions = [tlsv1.3, tlsv1.2]
      }
      tcp_options {
        active_n = 100
        backlog = 1024
        buffer = 4KB
        high_watermark = 1MB
        keepalive = none
        nodelay = true
        reuseaddr = true
        send_timeout = 15s
        send_timeout_close = true
      }
      websocket {
        allow_origin_absence = true
        check_origin_enable = false
        check_origins = "http://localhost:18083, http://127.0.0.1:18083"
        compress = false
        deflate_opts {
          client_context_takeover = takeover
          client_max_window_bits = 15
          mem_level = 8
          server_context_takeover = takeover
          server_max_window_bits = 15
          strategy = default
        }
        fail_if_no_subprotocol = true
        idle_timeout = 7200s
        max_frame_size = infinity
        mqtt_path = "/mqtt"
        mqtt_piggyback = multiple
        proxy_address_header = x-forwarded-for
        proxy_port_header = x-forwarded-port
        supported_subprotocols = "mqtt, mqtt-v3, mqtt-v3.1.1, mqtt-v5"
      }
    }
  }
}
log {
  console {
    enable = false
    formatter = text
    level = warning
    time_offset = system
  }
  file {
    default {
      enable = true
      formatter = text
      level = info
      path = "/opt/emqx/log/emqx.log"
      rotation_count = 10
      rotation_size = 50MB
      time_offset = system
    }
  }
}
retainer {
  backend {
    index_specs = [
      [1, 2, 3],
      [1, 3],
      [2, 3],
      [3]
    ]
    max_retained_messages = 0
    storage_type = ram
    type = built_in_database
  }
  enable = true
  max_payload_size = 1MB
  msg_clear_interval = 0s
  msg_expiry_interval = 12h
  stop_publish_clear_msg = false
}
telemetry {enable = false}


[kbz@VM-16-5-centos configs]$ cat /etc/emqx/data/authz/acl.conf 
%%--------------------------------------------------------------------
%% -type(ipaddr() :: {ipaddr, string()}).
%%
%% -type(ipaddrs() :: {ipaddrs, [string()]}).
%%
%% -type(username() :: {user | username, string()} | {user | username, {re, regex()}}).
%%
%% -type(clientid() :: {client | clientid, string()} | {client | clientid, {re, regex()}}).
%%
%% -type(who() :: ipaddr() | ipaddrs() | username() | clientid() |
%%                {'and', [ipaddr() | ipaddrs() | username() | clientid()]} |
%%                {'or',  [ipaddr() | ipaddrs() | username() | clientid()]} |
%%                all).
%%
%% -type(action() :: subscribe | publish | all).
%%
%% -type(topic_filters() :: string()).
%%
%% -type(topics() :: [topic_filters() | {eq, topic_filters()}]).
%%
%% -type(permission() :: allow | deny).
%%
%% -type(rule() :: {permission(), who(), action(), topics()} | {permission(), all}).
%%--------------------------------------------------------------------

{allow, {username, {re, "^dashboard$"}}, subscribe, ["$SYS/#"]}.

{allow, {ipaddr, "127.0.0.1"}, all, ["$SYS/#", "#"]}.

{deny, all, subscribe, ["$SYS/#", {eq, "#"}]}.
```

#### 4. 使用 nginx 代理 wss 并转发 ws
同时监听 wss 和 1890 端口，具体配置如下:
```text
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}
upstream mqtt_cn{
        server 127.0.0.1:1889 fail_timeout=0;
}

server {
    server_name  test.example.com;
    listen       443 ssl http2;
    listen       1890 ssl http2;
    include      ssl/ssl_go_example.cn.conf;
    access_log /var/log/nginx/test.example.com.access.log main;
    access_log /var/log/nginx/test.example.com.4xx5xx.log combined if=$loggable;
    error_log /var/log/nginx/test.example.com.error.log warn;

    location / {
          proxy_pass http://mqtt_cn;
          proxy_read_timeout 1800s;
          proxy_send_timeout 1800s;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_http_version 1.1;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection "upgrade";
    }
}

```

## 启用 Prometheus 统计
具体可以查看 [EMQX+Prometheus+Grafana：MQTT 数据可视化监控实践](https://www.emqx.com/zh/blog/emqx-prometheus-grafana)

## 集群
emqx 和 vernemq 一样，都是基于 Erlang/OTP 的集群机制，因为会涉及到频繁的 rpc 调用，因此如果要集群的话，只能同区集群
> 具体看 [分布式集群介绍](https://www.emqx.io/docs/zh/latest/deploy/cluster/introduction.html)

他对同一个集群之内的网络延迟要求很高，只能 10ms 之内，超过 100ms 将不可用
```text
核心节点之间的网络延迟建议 10ms 以下，实测高于 100ms 将不可用，请将核心节点部署在同一个私有网络下；
复制节点和核心节点之间同样建议部署在同一个私有网络下，但网络质量要求可以比核心节点间略低。
```

这意味着要做集群只能同一个区域的同一个数据中心，才能可能。一旦不同数据中心(走外网)，或者跨区(走内网都不行)，都没办法达到要求。

在他们的[未来版本](https://www.emqx.io/docs/zh/latest/getting-started/roadmap.html)中，其实也有要解决这个问题，但是肯定是跟我们用的开源版没有关系，应该在云版本做线路优化才有可能:
```text
未来版本
- 支持跨数据中心集群链接
- 多云集群
- 全球多区域地理分布式集群
- 默认使用 Elixir 发布版本
- 在后台通信中使用 QUIC 协议
- 在规则引擎中支持其他语言和外部运行时（例如 JavaScript、Python）
```

> 这一块好像他们官网 2024 年有实现了这个功能，可以跨界点会话了

因此如果我们后面要做集群的话，也只能在同一个区域内做集群，就跟之前 vernemq 的 hk 那两台的集群一样，只能同区域内做，没办法跨区做。

## 关于 emqx 有几个当前我们用不到，但是后续可能会用到的功能
emqx 有非常多有用的功能，有些可能我们用不着，有些可能后续我们用得到

### 1. 监控设置和告警
在 dashboard 后台，可以在 `[管理]-[监控]` 设置系统监控，并且在触发的时候，可以在 `[监控]-[告警]` 查看预警

具体文档: [emqx 告警](https://www.emqx.io/docs/zh/latest/observability/alarms.html)

### 2. 黑名单
EMQX 为用户提供了黑名单功能来禁止某些客户端的访问，除了可以封禁客户端 ID 以外，还支持直接封禁用户名甚至 IP 地址。可以在 dashboard 的 `[访问控制]-[黑名单]` 中设置

具体文档: [emqx 黑名单](https://www.emqx.io/docs/zh/latest/access-control/blacklist.html)

### 3. 连接抖动检测
在黑名单功能的基础上，EMQX 支持自动封禁那些被检测到短时间内频繁登录的客户端，并且在一段时间内拒绝这些客户端的登录，以避免此类客户端过多占用服务器资源而影响其他客户端的正常使用。

需要注意的是，连接抖动检测功能只会封禁客户端 ID，并不封禁用户名和 IP 地址，即该机器只要更换客户端标 ID 就能够继续登录。

抖动检测功能默认关闭，您可以通过 EMQX Dashboard 或配置文件开启该功能。在 `[访问控制]-[连续抖动]`

具体文档: [emqx 连接抖动检测](https://www.emqx.io/docs/zh/latest/access-control/flapping-detect.html)

### 4. 日志追踪 (Trace)
通过开启 DEBUG 级别日志能够有效地排查各类问题，但这会引起大量日志落地进而影响 EMQX 整体性能，尤其是在有大量连接与消息收发的生产环境中，该手段几乎是不可实施的。

针对这种问题，EMQX 5.0 新增了在线日志追踪(Trace)功能，允许用户指定客户端 ID、主题或 IP 实时过滤输出 DEBUG 级别日志。

具体文档: [emqx 日志追踪 (Trace)](https://www.emqx.io/docs/zh/latest/observability/tracer.html)

### 5. 慢订阅统计
正常情况下 EMQX 内部消息传输耗时都很低（毫秒级以下），大部分时间消耗都集中在网络传输上，针对客户端偶尔出现订阅消息时延高，EMQX 提供了慢订阅统计功能。

可以在 `[问题分析]-[慢订阅]` 设置和查看

具体文档: [emqx 慢订阅统计](https://www.emqx.io/docs/zh/latest/observability/slow-subscribers-statistics.html)

### 6. 主题监控
EMQX 提供了主题监控功能，可以统计指定主题下的消息收发数量、速率等指标。您可以通过 Dashboard 的 问题分析 -> 主题监控 页面查看和使用这一功能，也可以通过 HTTP API 完成相应操作。

可以在 `[问题分析]-[主题监控]` 设置和查看

出于整体性能考虑，目前主题监控功能仅支持主题名，即不支持带有 + 或 # 通配符的主题过滤器，例如 a/+ 等。如果我们解决了性能问题，将来有一天会提供支持。

具体文档: [emqx 主题监控](https://www.emqx.io/docs/zh/latest/observability/topic-metrics.html)

### 7. MQTT 高级特性 主题重写/代理订阅/延迟发布
这些高级功能也可以在 dashboard 后台配置，虽然现在用不着，但是可以了解一下，有可能后面用得到

- [主题重写](https://www.emqx.io/docs/zh/latest/messaging/mqtt-topic-rewrite.html)
- [代理订阅](https://www.emqx.io/docs/zh/latest/messaging/mqtt-auto-subscription.html)
- [延迟发布](https://www.emqx.io/docs/zh/latest/messaging/mqtt-delayed-publish.html)
- [遗嘱消息](https://www.emqx.io/docs/zh/latest/messaging/mqtt-will-message.html)

## 相关调研和参考文献
- [EMQX+Prometheus+Grafana：MQTT 数据可视化监控实践](https://www.emqx.com/zh/blog/emqx-prometheus-grafana)
- [emqx 官方文档](https://www.emqx.io/docs/zh/latest/)
- [emqx mqtt 教程](https://www.emqx.com/zh/mqtt-guide)
- [分布式集群介绍](https://www.emqx.io/docs/zh/latest/deploy/cluster/introduction.html)









