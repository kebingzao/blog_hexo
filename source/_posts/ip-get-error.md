---
title: 记一次获取客户端IP不正确的情况
date: 2019-05-30 17:20:36
tags: nginx
categories: nginx相关
---
## 前言
之前有测试同学反馈他请求的转发服会分配非中国区。后面我们查了一下，发现我们程序获取的客户端 ip 不是真正的客户端ip，而是负载均衡服务器的ip 或者是 cdn 服务端的ip (反正就是反向代理服务器的ip)。
因此我们查看了一下获取 ip 的方法 (golang) :
```html
//获取客户端ip, nginx代理传入的是 x-real-ip
func ClientIp(r *http.Request) string {
   ip := r.Header.Get("X-Real-Ip")
   if ip == "" {
      s := strings.Split(r.RemoteAddr, ":")
      ip = s[0]
   }
   return ip
}
```
发现这边只优先获取 **x-real-ip**， 如果找不到，那么就获取 **remote addr**。
<!--more-->
## 分析
但是这样会有一个问题，因为 **x-real-ip** 其实就是实际上跟我们的业务服务器请求的那一台服务器，如果我们的服务有用了负载均衡或者动态加速服务的话。那么往往最后跟业务服务器进行请求的服务器就不是用户的客户端，而是负载均衡服务器或者是动态CDN服务器。因此就会导致我们获取的客户端ip就不对，就不是用户的真正的客户端ip了。
首先我们来看一下 nginx 的转发配置， 这样是为了记录完整的代理过程:
```html
    location /p20/ {
        access_by_lua_file "$lua_dir/access/jwt.lua";
        proxy_pass http://p20/;
        proxy_set_header  X-Real-IP       $remote_addr;
        proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
    }
```
可以看到会将 **$remote_addr** 赋给 **x-real-ip** 这个头部。 而 **x-real-ip** 是不能被篡改的。而对于 nginx 转发的服务，也不能直接取请求的 **remote addr**，因为这时候就会取到本机的 nginx 的服务的 ip 和端口。
我们来看个例子： 这边有个接口：
```html
func (t *Test) Ip() revel.Result {
    res := Res{}
    data := Data{}
    data.REMOTEADDR = t.Request.RemoteAddr
    data.XREALIP = t.Request.Header.Get("X-Real-IP")
    data.XFORWARDFOR = t.Request.Header.Get("X-Forwarded-For")
    res.Code = 1
    res.Data = data
    return utils.EchoResult(t.Controller, res)
}
```
然后我们请求一下，看看输出结果：
```html
{
    code: 1,
    msg: "",
    data:  {
        x_real_ip: "118.xx.19.127",
        x_forward_for: "125.xx.202.250, 118.xx.19.127, 118.xx.19.127",
        remote_addr: "127.0.0.1:35971"
    },
    extra: null
}
```
因为我们这个服务有用到了腾讯云的 clb 负载均衡服务。 所以可以看到 **x-real-ip** 其实就是最后一台负载均衡服务器的 ip。并不是真实的客户端ip。而 **remote addr** 也不是客户端ip，而是 nginx 本机服务的 ip 和 端口。只有 **x-forward-for** 这个头部的第一个ip地址才是真实的客户端ip。
## X-Forwarded-For
**X-Forwarded-For** 是一个可叠加的过程，后面的代理会把前面代理的 IP 加入**X-Forwarded-For**，类似于python的列表append的作用.
**Remote Address**代表的是当前HTTP请求的远程地址，即HTTP请求的源地址。HTTP协议在三次握手时使用的就是这个**Remote Address**地址，在发送响应报文时也是使用这个**Remote Address**地址。因此，如果请求者伪造**Remote Address**地址，他将无法收到HTTP的响应报文，此时伪造没有任何意义。这也就使得**Remote Address**默认具有防篡改的功能。
在一些大型网站中，来自用户的HTTP请求会经过反向代理服务器的转发，此时，服务器收到的**Remote Address**地址就是反向代理服务器的地址。在这样的情况下，用户的真实IP地址将被丢失，因此有了HTTP扩展头部**X-Forward-For**。当反向代理服务器转发用户的HTTP请求时，需要将用户的真实IP地址写入到**X-Forward-For**中，以便后端服务能够使用。由于**X-Forward-For**是可修改的，所以**X-Forward-For**中的地址在某种程度上不可信。所以，在进行与安全有关的操作时，只能通过**Remote Address**获取用户的IP地址，不能相信任何请求头。
当然，在使用 nginx 等反向代理服务器的时候，是必须使用**X-Forward-For**来获取用户IP地址的（此时**Remote Address**是nginx的地址），因为此时**X-Forward-For**中的地址是由 nginx 写入的，而 nginx 是可信任的。不过此时要注意，要禁止web对外提供服务。
## 最后修改
所以最后这个方法要改成要优先取 **X-Forward-For** 的值。没有再取 **x-real-ip**， 再没有，再取 **remote address**。
```html
// 获取客户端 ip
// X-Forwarded-For : client ip, proxy ip, proxy ip ..., 理论上应该取逗号分隔后的第一个 ip
// X-Real-Ip : 最接近 nginx 的 ip, 如果请求经过多个代理转发，那么获取到的最后一个代理服务器的 ip
// remote addr : go net 包取的 ip, 如果经过 nginx upstream 反向代理到本机的 go 程序，则会取到 127.0.0.1
func ClientIp(r *http.Request) string {
    ip := ""

    // X-Forwarded-For
    xForwardedFor := r.Header.Get("X-Forwarded-For")
    if xForwardedFor != "" {
        proxyIps := strings.Split(xForwardedFor, ",")
        if len(proxyIps) > 0 {
            ip = proxyIps[0]
            log.Info(ip, " hit x-forwarded-for")
            return ip
        }
    }

    // X-Real-IP, X-Real-Ip
    ip = r.Header.Get("X-Real-IP")
    if ip == "" {
        ip = r.Header.Get("X-Real-Ip")
    }
    if ip != "" {
        log.Info(ip, " hit x-real-ip")
        return  ip
    }

    // Remote Addr
    s := strings.Split(r.RemoteAddr, ":")
    if len(s) > 0 {
        ip = s[0]
        log.Info(ip, " hit remote addr")
    }
    return ip
}
```
最后再附上 PHP 程序的对应方法：
```html
/**
* 获取浏览器的ip地址
*/
public static function getClientIP()
{
    if (!empty($_SERVER['HTTP_X_FORWARDED_FOR'])) {
        $ip = $_SERVER['HTTP_X_FORWARDED_FOR'];
    } elseif (!empty($_SERVER['HTTP_CLIENT_IP'])) {
        $ip = $_SERVER['HTTP_CLIENT_IP'];
    } elseif (!empty($_SERVER['REMOTE_ADDR'])) {
        $ip = $_SERVER['REMOTE_ADDR'];
    } elseif (getenv('HTTP_X_FORWARDED_FOR')) {
        $ip = getenv('HTTP_X_FORWARDED_FOR');
    } elseif (getenv('HTTP_CLIENT_IP')) {
        $ip = getenv('HTTP_CLIENT_IP');
    } elseif (getenv('REMOTE_ADDR')) {
        $ip = getenv('REMOTE_ADDR');
    } else {
        $ip = '';
    }

    if (strpos($ip, ',') !== false) {
        $ips = explode(',', $ip);
        $ip = trim($ips[0]);
    }

    return $ip;
}
```
## 参考
[HTTP 请求头中的 X-Forwarded-For，X-Real-IP](https://www.cnblogs.com/diaosir/p/6890825.html)
[X-Forwarded-For的一些理解](https://www.cnblogs.com/huaxingtianxia/p/6369089.html)


