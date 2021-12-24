---
title: nginx 针对某个路由设置 301 跳转，并且排除某一个符合条件的路由
date: 2021-12-24 10:38:06
tags: nginx
categories: nginx相关
---
## 前言
之前官网重构的时候，为了保证原页面的 seo 权重，就有针对一些旧页面路由进行 301 重定向到新路由。 具体看 {% post_link nginx-301-4 %}， 但是后面又提了一个新需求， 就是针对一些路由进行目录结构上的调整。

比如原先是 `a.html` 后面会移动到 `product/a.html`， 而且 html 的名称还不能变。 所以这时候从用户访问来看就是:
```text
访问 www.foo.com/a.html --> 301 重定向 --> www.foo.com/product/a.html
考虑到大部分是有小语种路径的，比如 zh-cn，那么就是
访问 www.foo.com/zh-cn/a.html --> 301 重定向 --> www.foo.com/zh-cn/product/a.html
```

## 原先的方法不适用
如果是原先这种方式的话:
```text
if ($request_uri ~* "a.html") {
    rewrite ^/(.*)a.html$ /$1product/a.html permanent;
}
```
就会发现其实重定向后的 url 也是符合这个规则的，就会导致 nginx 无限重定向。 所以这样子是不行的
<!--more-->
## 尝试一: 使用正则表达式排除特定字符串
其实 正则表达式是可以 零宽度断言(?!exp)，来进行排除某个字符串的。

这个是因为正则有一个向前查找的语法(也叫顺序环视) `(?=exp)`, `(?=exp)` 会查找 exp 之前的【位置】, 如果将等号换成感叹号，就变成了否定语义，也就是说查找的位置的后面不能是 exp。

一般情况下 `?!` 要与特定的锚点相结合，例如`^`行开头或者`$`行结尾。

而本例的这个 `product`, 它并不会在路由的开头 (可能还有多语言路径)， 也不会在结尾 (后面还有具体的 html 路由)， 原则上可以用:
```text
^(?!.*product)/a.html$
```
来实现， 不过我试了一下，还是不行，会漏掉有匹配 `a.html` 的情况。但是如果只是单独的排除掉 `product/a.html` 的路由。那么其实用这个语法是可以的:
```text
^(?!.*product/a\.html).*$
```
这样子只要路由不包含 `product/a.html` 字串的，就会匹配到。  但是还不满足我的需求， 我的需求是要在当前符合 `a.html` 的匹配下，还要排除掉 `product/a.html` 的情况， 但是很显然， 按照我的测试来说，这个得分为两个匹配来处理。

但是如果要分为两个匹配的话，我就不需要用这种复杂的正则来处理了，直接在 nginx 那边启用变量就行了。

## 启用变量来区分
如果要启用变量来处理，那么就很简单:
1. 初始化一个变量 0
2. 匹配到 `a.html`， 变量设置为 1
3. 匹配到 `product/a.html`， 变量设置为 0， 表示不跳转
4. 最后判断变量如果为 1 的话，那么就重定向

具体逻辑如下:
```text
set $needrw "0";
if ($request_uri ~* "/a.html") {
   set $needrw "1";
}

if ($request_uri ~* "/product/a.html") {
   set $needrw "0";
}

if ($needrw = "1") {
   rewrite ^/(.*)a.html /$1product/a.html permanent;
}
```
这样子就简单易懂了。 当前缺点就是 多写了好多代码。

那么如果有多个路由都要这么干呢，那不是直接吐血了。比如:
```text
a -> kk/a.html
b -> age/b.html
c --> jj/c.html
```
那么就要用 更有效的方式，将 `$needrw` 当做要跳转的变量， 然后直接判断不匹配的方式，就行了:
```text
set $needrw "0";
# 先匹配固定的，然后设置对应的跳转路由
if ($request_uri ~* "/b.html") {
   set $needrw "age/b.html";
}

if ($request_uri ~* "/a.html") {
   set $needrw "kk/a.html";
}

if ($request_uri ~* "/c.html") {
   set $needrw "jj/c.html";
}
# 排除掉特定的不跳转的路由
if ($request_uri ~* "/age/b.html") {
   set $needrw "0";
}

if ($request_uri ~* "/kk/a.html") {
   set $needrw "0";
}

if ($request_uri ~* "/jj/c.html") {
   set $needrw "0";
}
# 最后如果是要跳转的，直接取变量来跳转
if ($needrw != "0") {
   rewrite ^(/?)(.*)/(.*).html $1$2/$needrw permanent;
}    
```
这样子虽然代码也不少， 但是比一个一个指定会好很多。而且变量只需要一个就行了。

---

参考资料:
- [利用正则表达式排除特定字符串](https://www.cnblogs.com/wangqiguo/archive/2012/05/08/2486548.html)




