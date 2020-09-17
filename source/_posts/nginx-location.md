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
Nginx 的 location 规则匹配的变量是 `$request_uri`, 所以不用管后面的参数 `$query_string` (或者 `$args`)

## location 匹配的种类
格式主要是这个:
```text
location [空格 | = | ~ | ~* | ^~ | @ ] /uri/ {
  ...
}
```
其实上面分为三部分:
1. 最前面的字符 (location modifier) 匹配规则
2. 后面 uri 的匹配规则 (location match)
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

总结一下过来就是， 按照优先级的步骤如下
1. 优先查找精确匹配，精确匹配 (=) 的 location 如果匹配请求 URI 的话，此 location 被马上使用，匹配过程结束。
2. 接下来进行字符串匹配(空格 和 ~^), 找到匹配最长的那个，如果发现匹配最长的那个是 ^~ 前缀, 那么也停止搜索并且马上使用，匹配过程结束。 否则继续往下走。
3. 如果字符串匹配没有，或者匹配的最长字符串不是 ^~ 前缀 (比如是空格匹配)，那么就继续搜索正则表达式匹配， 这时候就根据在配置文件定义的顺序，取最上面的配置(正则匹配跟匹配长度没关系，只跟位置有关系，只取顺序最上面的匹配)
4. 如果第三步找到了，那么就用第三步的匹配，否则就用第二步的匹配 (字符匹配最长的空格匹配)

简单的来说就是顺序如下:
```text
精确匹配 > 字符串匹配( 长 > 短 [ 注: ^~ 匹配则停止匹配 ]) > 正则匹配( 上 > 下 )
```
换成符号的优先级就是:
```text
[=] > [^~] > [~/~*] > [空格]
```

这边注意几个细节:
1. 常规字符串匹配类型。是按前缀匹配(从跟开始)。 而且正则匹配是包含匹配，只要包含就可以匹配
2. ~ 和 ~* 的优先级一样，取决于在配置文件中的位置，最上面的为主，跟匹配的字符串长度没关系，所以在写的时候，应该越精准的要放在越前面才对
3. 空格匹配和 ^~ 都是字符串匹配，所以如果两个后面的匹配字符串一样，是会报错的，因为 nginx 会认为两者的匹配规则一致，所以会有冲突
4. ^~, =, ~, ~* 这些修饰符和后面的 URI 字符串中间可以不使用空格隔开(大部分都是用空格隔开)。但是 @ 修饰符必须和 URI 字符串直接连接。

接下来通过几个实践来帮助理解:

### 例子 1
配置文件配置:
```text
  location / {
		  return 200 '404';
  }

	location ~ /hello {
   	  return 200 '1';
	}

	location ^~ /hello {
    	return 200 '2';
	}

	location = /hello {
      return 200 '3';
	}

	location ~* /hello {
      return 200 '4';
  }
```
这个是测试结果
```text
[root@VM_156_200_centos ~]# curl 127.0.0.1/hello  #精确匹配，直接结束
3
[root@VM_156_200_centos ~]# curl 127.0.0.1/hello11 #字符串匹配，并且最大长度的匹配是 ~^,直接结束
2
[root@VM_156_200_centos ~]# curl 127.0.0.1/hello/22 #字符串匹配，并且最大长度的匹配是 ~^,直接结束
2
[root@VM_156_200_centos ~]# curl 127.0.0.1/11/hello/ #字符串不匹配(前缀匹配)，正则匹配有两个，取最上面的那个
1
[root@VM_156_200_centos ~]# curl 127.0.0.1/11/Hello/ #字符串不匹配(前缀匹配)，正则匹配有一个(大小写不敏感)，取最上面的那个
4
[root@VM_156_200_centos ~]# curl 127.0.0.1/11/Hell #都不匹配，有设置通用匹配，取通用匹配
404
```

### 例子 2
配置文件配置:
```text
   location /images/test.png {
       return 200  '1';     
   }        

   location ^~ /images/ { 
        return 200  '2';      
   }    

   location ~ /images/ {        
        return 200  '3';   
   } 
   
   location ~ /images/test.png {        
        return 200  '4';   
   }
```
这个是测试结果
```text
[root@VM_156_200_centos ~]#  curl http://127.0.0.1/images/test.png 
3
[root@VM_156_200_centos ~]#  curl http://127.0.0.1/images/1 
2
```
第一个返回 3 是因为如果 普通匹配和正则匹配都存在的话， 并且因为 ^~ 匹配不是最长的话，那么就取 正则匹配， 正则匹配满足条件的有两个， 取最上面那个， 所以是 3， (本例的字符串最长匹配是空格匹配)。(对于本例来说，如果去掉后面的两个正则匹配，那么返回的就是 1， 因为空格匹配的字符串是最长的)

第二个返回 2 是因为普通匹配和正则匹配都存在，但是这个^~ 匹配是最长的，所以就是 2。

### 普通字符串的匹配冲突
如果我这样子写:
```text
   location /images/test.png {
       return 200  '1';     
   }        

   location ^~ /images/test.png { 
        return 200  '2';      
   }  
```
这时候我 reload 是会报错的:
```text
root@VM_156_200_centos sbin]# ./nginx -s reload
nginx: [emerg] duplicate location "/images/test.png" in /usr/local/nginx/conf/nginx.conf:20
```
这个是因为这两个匹配的规则是一样的，所以会有冲突，了解更多请看 {% post_link centos7-systemctl-reload-nginx %}

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

## location match [uri]
这里主要填的是需要匹配的 path 路径，根据前面的符号，这里可以填写精确到 path 路径，也可以填正则表达式，Nginx 用的是 PCRE正则表达式语法，主要是以下:










---

参考文档
- [Nginx的location配置规则总结 - 运维笔记](https://www.cnblogs.com/kevingrace/p/6804429.html)
- [Nginx location匹配规则](https://www.cnblogs.com/BeiGuo-FengGuang/p/9844128.html)
- [Nginx 源代码笔记 - URI 匹配](https://ialloc.org/blog/ngx-notes-http-location/)
- [Module ngx_http_core_module](http://nginx.org/en/docs/http/ngx_http_core_module.html#location)










