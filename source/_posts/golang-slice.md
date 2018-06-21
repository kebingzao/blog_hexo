---
title: golang 踩坑之 - slice bounds out of range
date: 2018-06-22 01:14:31
tags: golang
categories: golang相关
---
之前有发生过一个现象，我们的一个go服务定时会出现重启的行为，时间间隔有可能是几个小时，也有可能是几天。 后面查看了一下supervisor的error log，发现：
{% codeblock lang:js %}
panic: runtime error: slice bounds out of range

goroutine 7217 [running]:
panic(0x6f6a40, 0xc42000c0a0)
    /usr/local/go1.7.6/src/runtime/panic.go:500 +0x1a1
main.phoneHandler(0x8b3ba0, 0xc4200ee088)
    /data/code/go/src/stream-forward/forward_handlers.go:27 +0x902
created by main.phoneServe
    /data/code/go/src/stream-forward/main.go:72 +0xeb
{% endcodeblock %}

看情况好像是一个 切片数组越界的错误？？ 我们定位到具体的代码：
{% codeblock lang:js %}
buf := make([]byte, 512)

n := bytes.Index(buf, []byte{'\n'})
deviceId := string(buf[:n])
{% endcodeblock %}
果然发现了一个 可能会导致slice切片越界的行为。如果 index 检索不到对应的字符，那么n就会为 -1，当 n为-1 的时候，就会报这个错误。 所以这边要改成：
{% codeblock lang:js %}
n := bytes.Index(buf, []byte{'\n'})
// 这边如果返回-1的话，那么下面那个slice就会报一个数组越界的错误，导致程序退出
if n <= 0 {
    return
}
deviceId := string(buf[:n])
{% endcodeblock %}

这边只要提前判断就行了。