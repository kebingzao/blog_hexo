---
title: Nginx 的 location 匹配规则总结
date: 2020-09-16 16:04:08
tags: nginx
categories: nginx相关
---
## 前言
之前在看 Nginx 的 location 匹配规则的时候，参考了一些网上的文章，但是这些文章，要么不全要么就是有问题的，后面打算结合我自己的实践，自己写一篇算了。

本次实践的环境:
- 系统: CentOS 7
- Nginx 版本: 1.18.0

## location 匹配的变量
Nginx 的 location 规则匹配的变量是 `$request_uri`, 所以不用管后面的参数 `$query_string`

## location 匹配的种类
格式主要是这个:
```text
location [空格 | = | ~ | ~* | ^~ | @ ] /uri/ {
  ...
}
```
其实上面分为三部分:
1. 最前面的字符 (location modifier) 匹配规则
2. 后面 uri 的匹配规则
3. 大括号内的路由转发

## location modifier
格式:
```text
[空格 | = | ~ | ~* | ^~ | @ ]
```
接下来解释一下，这些都表示啥意思:

|字符|解释|
|---|---|
| 空格 | 如果直接是一个空格，那就是不写 location modifier ，Nginx 仍然能去匹配 pattern 。这种情况下，匹配那些以指定的 patern 开头的 URI，注意这里的 URI 只能是普通字符串，不能使用正则表达式。|
| = | 表示精确匹配，如果找到，立即停止搜索并立即处理此请求。|
| ~ | 表示执行一个正则匹配，区分大小写匹配 |
| ~* | 表示执行一个正则匹配，不区分大小写匹配， 注意，如果是运行 Nginx server 的系统本身对大小写不敏感，比如 Windows ，那么 ~* 和 ~ 这两个表现是一样的|
| ^~ | 即表示只匹配普通字符（跟空格类似，但是它优先级比空格高）。使用前缀匹配，^表示“非”，即不查询正则表达式。如果匹配成功，并且所匹配的字符串是最长的， 则不再匹配其他location。|
| @ | 用于定义一个 Location块，且该块不能被外部Client 所访问，只能被Nginx内部配置指令所访问，比如try_files 或 error_page |

### location modifier 的匹配顺序
首先我们直接看官方的 wiki:
{% blockquote nginx http://nginx.org/en/docs/http/ngx_http_core_module.html#location %}
To find location matching a given request, nginx first checks locations defined using the prefix strings (prefix locations). Among them, the location with the longest matching prefix is selected and remembered. Then regular expressions are checked, in the order of their appearance in the configuration file. The search of regular expressions terminates on the first match, and the corresponding configuration is used. If no match with a regular expression is found then the configuration of the prefix location remembered earlier is used.
{% endblockquote %}
之前还看过一个更具体的文档:
{% blockquote nginx https://www.nginx.com/resources/wiki/#location %}
The order in which location directives are checked as follows:

1. Directives with the "=" prefix that match the query exactly (literal string). If found, searching stops.
2. All remaining directives with conventional strings. If this match used the `^~" prefix, searching stops.
3. Regular expressions, in the order they are defined in the configuration file.
4. If #3 yielded a match, that result is used. Otherwise, the match from #2 is used.

To determine which location directive matches a particular query, the literal strings are checked first. Literal strings match the beginning portion of the query - the most specific match will be used. Afterwards, regular expressions are checked in the order defined in the configuration file. The first regular expression to match the query will stop the search. If no regular expression matches are found, the result from the literal search is used.
{% endblockquote %}

翻译过来就是:


### 辟谣之 - ~ 比 ~* 优先级高
之前也有在网上看到这种说法，就是匹配顺序的时候，如果都匹配了，那么 ~ 比 ~* 优先级高，其实这个说法是错误的，这哥俩并没有谁比谁高贵，根据上面的匹配顺序，如果都是正则匹配，那么就是谁排在前面，就采用谁。 做个实践, 我的执行顺序是这样子的:
```text
	location ~* \.jpG$ {
   return 200  '1';
  }

	location ~ \.jpg$ {
   return 200  '2';
  }
```
我的测试结果如下:
```text
[root@VM_156_200_centos ~]# curl 127.0.0.1/1.jpg
1
[root@VM_156_200_centos ~]# curl 127.0.0.1/1.JPG
1
```
只要有匹配，肯定是最上面的那个 1。

### 辟谣之 - modifier 包含 !~ 和 !~*
我看到有些网上的文章说 nginx 的 location modifier 还包含 `!~` 和 `!~*` 这两个，其实是不对的， 如果你这样子:
```text
location !~ \.(gif|jpg)$ {
  return 200  '1';
}
```
那么在 reload 的时候， nginx 会报这个错误:
```text
nginx: [emerg] invalid location modifier "!~" in /usr/local/nginx/conf/nginx.conf:25
```
不过因为 Nginx 的正则是使用PCRE（Perl Compatible Regular Expressions）, 所以我们可以这样子写来达到我们想要的目的:
```text
location ~ \.*(?<!(gif|jpg))$ {
 return 200  '1';
}
```
做个实践，假设我的路由是这样子: 如果后缀含有 gif 或者是 jpg， 那么就会返回 200， 否则就会返回 404
```text
location / {
  return 200 '404';
}

location ~ \.(gif|jpg)$ {
 return 200  '200';
}
```
测试结果如下:
```text
[root@VM_156_200_centos ~]# curl 127.0.0.1/1.gif
200
[root@VM_156_200_centos ~]# curl 127.0.0.1/1.jpg
200
[root@VM_156_200_centos ~]# curl 127.0.0.1/1.js
404
[root@VM_156_200_centos ~]# curl 127.0.0.1/1.css
404
```
这个结果是对的。 那么就换成，如果后缀不是 gif 或者是 jpg，那么才返回 200， 否则就返回 404 (相当于上述的 !~):
```text
location / {
  return 200 '404';
}
location ~ \.*(?<!(gif|jpg))$ {
  return 200  '200';
}
```
可以看到，同样的结果，结果相反了:
```text
[root@VM_156_200_centos ~]# curl 127.0.0.1/1.gif
404
[root@VM_156_200_centos ~]# curl 127.0.0.1/1.jpg
404
[root@VM_156_200_centos ~]# curl 127.0.0.1/1.js
200
[root@VM_156_200_centos ~]# curl 127.0.0.1/1.css
200
```
所以是可以实现这种效果的。







