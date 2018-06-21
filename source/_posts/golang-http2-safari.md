---
title: golang 踩坑之 - https下开启http2会在Safari下报错(http2 stream closed)
date: 2018-06-22 01:30:26
tags: 
    - golang
    - http2
categories: golang相关
---
之前项目的一个转发服务有发生过一个问题，就是在新版的Safari上（版本11），https 的链接会一直连不上。然后我看了一下log，发现一直在刷log，明显是Safari一直在重试请求，而且一直报这个错误：
{% codeblock lang:js %}
[id(12a0de8483dc83df60d2d0ac49118e57): io copy fail : http2: stream closed
{% endcodeblock %}
这个错误就是http2的错误，但是为啥就新版的Safari会，旧版的Safari和chrome 和 Firefox都正常？？
<!--more-->
后面发现可能是新版的Safari的安全机制的问题。因为golang 的http库 https 是默认启用http2的， 所以解决方法其实就是将 https 默认启用的 http2 传输去掉：[参考资料](https://groups.google.com/forum/#!topic/golang-dev/S_K0Rplup3M)
所以就改了一个配置(就是下面的 TLSNextProto 选项):
{% codeblock lang:js %}
s := &http.Server{
       Addr:      config.SslWebPort,
       TLSConfig: tlsConfig,
       //todo  https 下禁用http2 （不然新版的Safari会一直请求失败，然后会一直重试）
       TLSNextProto: map[string]func(*http.Server, *tls.Conn, http.Handler){
              "spdy/3": func(s *http.Server, conn *tls.Conn, h http.Handler) {
                     buf := make([]byte, 1)
                     if n, err := conn.Read(buf); err != nil {
                            log.Errorf("%v|%v\n", n, err)
                     }
              },
       },
}

if err := s.ListenAndServeTLS(config.SslCrt, config.SslKey); err != nil {
       log.Error("ListenAndServeTLS err,", err)
       log.Flush()
       os.Exit(1)
}
{% endcodeblock %}

这样程序就不会开启http2传输了。后面更新上去，发现 Safari 正常了。