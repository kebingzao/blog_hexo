---
title: 记一次 php-fpm 的配置调优
date: 2023-11-21 16:02:20
tags: php
categories: php相关
---
## 前言
前段时间有一个很老的 php 服务发生了一个并发情况，产生大量的 499 状态码，刚开始以为是高并发导致的服务器负载不行。结果去 grafana 后台一看，服务器负载是正常的，然后查了一下同台的 golang 服务，他的返回状态码是非常正常的。

所以怀疑应该是 php 服务有问题，因为我们用的是 nginx + php-fpm 的部署方式， 所以应该是 php-fpm 的配置有问题。

然后抓了一下这一台的 php-fpm 的配置，果然有问题:
<!--more-->
```text
[kbz@ip-10-1-2-80 202311]$ cat /usr/local/php-5.6.40/etc/php-fpm.conf
[global]
pid = /usr/local/php/var/run/php-fpm.pid
error_log = /var/log/php/php-fpm.error.log 
 
[www]
listen = 127.0.0.1:9000
user = www
group = www
pm = static
pm.status_path = /status
pm.max_children = 30
pm.max_requests = 5000
pm.start_servers = 30
pm.min_spare_servers = 10
pm.max_spare_servers = 60
request_terminate_timeout = 60s
request_slowlog_timeout = 9s
slowlog = /var/log/php/php-fpm.slow.log
```

启用的进程管理模式是静态模式 (`static`), 但是开启的进程 `pm.max_children=30` 才 30 个进程，这个明显很有问题，太少了，并发高一点，就堵塞了。

而且看了一下这一台的配置有 4核 8G，完全可以配置高一点，猜测应该是早期的服务，随着业务量的增多，运维只升级了机器的硬件配置，并没有连同 php-fpm 的配置一起调上来。

## php-fpm 的三种进程管理方式
直接看官方文档: [php-fpm.conf 全局配置段](https://www.php.net/manual/zh/install.fpm.configuration.php), 我们知道 php-fpm 有三种进程管理方式:
1. `static` - 子进程的数量是固定的（pm.max_children）
2. `ondemand` - 子进程在有需求时才产生, 当请求时才启动。与 dynamic 相反，后者在服务启动时就启动了 pm.start_servers 个子进程
3. `dynamic` - 子进程的数量在下面配置的基础上动态设置：pm.max_children，pm.start_servers，pm.min_spare_servers，pm.max_spare_servers

### 1. 几个相关的重要配置

|配置|描述|
|---|---|
| `pm.max_children` | pm 设置为 static 时表示创建的子进程的数量，pm 设置为 dynamic 时表示最大可创建的子进程的数量。必须设置 |
| `pm.start_servers`| 设置启动时创建的子进程数目。仅在 pm 设置为 dynamic 时使用。默认值：min_spare_servers + (max_spare_servers - min_spare_servers) / 2 |
| `pm.min_spare_servers`| 设置空闲服务进程的最低数目,如果空闲状态 worker 子进程小于该值，会创建一些子进程（直到达到该值）。仅在 pm 设置为 dynamic 时使用。必须设置 |
| `pm.max_spare_servers`| 设置空闲服务进程的最大数目,如果空闲状态 worker 子进程大于该值，会杀死一些子进程（直到达到该值）。仅在 pm 设置为 dynamic 时使用。必须设置 |
| `pm.max_requests`| 设置每个子进程重生之前服务的请求数。对于可能存在内存泄漏的第三方模块来说是非常有用的。如果设置为 '0' 则一直接受请求 |
| `pm.request_terminate_timeout`| 设置单个请求的超时中止时间。该选项可能会对 php.ini 设置中的 'max_execution_time' 因为某些特殊原因没有中止运行的脚本有用 |
| `request_slowlog_timeout` | 当一个请求该设置的超时时间后，就会将对应的 PHP 调用堆栈信息完整写入到慢日志中。设置为 '0' 表示 'Off' |
| `slowlog` | 慢请求的记录日志。默认值：#INSTALL_PREFIX#/log/php-fpm.log.slow |

### 1. 静态模式 static
静态模式的进程数只吃 `pm.max_children` 这个配置，顾名思义，一启动的时候，就会起 `pm.max_children` 个进程。

**优点:**
1. 方法简单, 只需要配 `pm.max_children` 这个配置项
2. 避免了频繁开启关闭进程的开销

**缺点:**
1. 因为只需考虑`pm.max_children` 这个配置项，因此很吃系统资源， 数量取决于 CPU 的个数和应用的响应时间，尤其是 CPU, php-fpm 有很多地方都会用到 CPU 资源，比如复杂业务计算，外部资源调用，IO 操作，并发等等。

### 2. 动态模式 dynamic
动态模式刚开始启动的时候，会吃 `pm.start_servers` 这个配置，先 fork 出这个配置的子进程出来，可以管理的最大子进程是吃 `pm.max_children` 这个配置。

然后除了这两个配置之外，还吃另外两个配置  `pm.min_spare_servers` 和 `pm.max_spare_servers` 针对空闲子进程的弹性伸缩， 因此动态模式的子进程的最小值应该是 `pm.min_spare_servers` 这个才对。

事实上动态模式正确的子进程配置应该是
```text
pm.min_spare_servers <= pm.start_servers <= pm.max_spare_servers <= pm.max_children
```
不然就会启动失败，比如配置是这样子的:
```text
pm = dynamic
pm.max_children = 100
pm.start_servers = 20
pm.min_spare_servers = 10
pm.max_spare_servers = 50
```
表示 php-fpm 启动时候，就要 fork 20 个子进程，然后在业务请求中，最多可以到 100 个子进程。比如一波高并发，一下子就干到了 100 个(多出来的请求只能堵塞排队了，因为已经满了)

这时候流量慢慢减少，有 80 个进程进程处理完了，就变成空闲，这时候就会将空闲的进程释放掉一部分，本例就是 80-50=30， 也就是当前的进程就只剩下 20 + 50 = 70. 然后剩余的 20 个子进程也处理完了，又空出来了，这时候还要再释放一波 70-50=20，因此当一波流量过后，还剩下 50 个空闲进程，也就是上面设置的最大空闲进程数。

这 50 个空闲进程，我们可以通过 `kill -9` 的指令来删掉，不过当删掉最后只剩下 10 个空闲进程的时候， 再删一次的时候，就会发现进程是删掉了，但是马上又重建了一个出来，保持最少有 10 个空闲子进程，也就是上面设置的最小空闲进程数。

**优点:**
1. 动态扩容，不浪费系统资源

**缺点:**
1. 当所有的 worker 进程都在工作时，新的请求到来需要等待创建 worker 进程，最长等待 1s（内部存在一个 1s 的定时器，去查看，创建进程）
2. 会频繁的启动停止进程，消耗系统资源（当请求数稳定时，不需要频繁销毁）

### 3. 按需模式 ondemand
按需模式，启动的时候只有一个 master 主进程，要接下来有访问才有子进程。 他会吃这个配置:
- `pm.process_idle_timeout` -> 秒数，多久之后结束空闲进程。 仅当设置 pm 为 ondemand。 可用单位：s（秒），m（分），h（小时）或者 d（天）。默认单位：10s

**优点:**
1. 按流量需求创建，不浪费系统资源

**缺点:**
1. 由于 PHP-FPM 是短连接，所以每次请求都会先建立连接，建立连接的过程会耗费系统之源，当流量很大时，频繁的创建销毁进程开销很大，不适合大流量环境部署


## 最后调整
从上面的三种模式来看， 对于生产环境，尤其是有比较大并发的业务线，静态模式肯定是最合适的， 动态模式和按需模式 每次都要频繁的启动停止进程，开销太大了，响应根本来不及。

事实上我有试过在高并发的时候，用动态模式，结果就是直接被打崩了，直接 502 了。

虽然是静态模式，但是配置也要非常合理，上述的那个 4核8G ，然后只给 32 个最大进程，明显就很不合理，正常以一个进程 30M 来算， 8G 的内存至少可以支持 250+ 以上的进程数。

当然不需要这么极限，可以先调到 100 或者 150，假设以 100 为例，那么就是:
```text
pm.max_children = 100
```
其他参数倒是没啥问题，唯一一个比较会影响资源回收的 `pm.max_requests = 5000` 设置的也比较合理，意味着我一个单进程在处理了超过 5000 个请求之后，就会回收了。重新让 php-fpm fork 一个新的子进程，可以有效避免内存泄露。

最后调完上线之后，我们可以看到当前的子进程就变成 101 个:
```text
[kbz@vm-16-119-centos ~]$ ps -ef |grep php-fpm | grep -v grep |wc -l
101
```
之所以多一个是因为有一个是 master 进程, 所以数量是对的。
```text
[kbz@vm-16-119-centos ~]$ ps -ef |grep php-fpm | grep -v grep
root     26190     1  0 07:03 ?        00:00:00 php-fpm: master process (/usr/local/php/etc/php-fpm.conf)
www      26191 26190  0 07:03 ?        00:00:40 php-fpm: pool www
www      26192 26190  0 07:03 ?        00:00:39 php-fpm: pool www
...
```

上线之后，发现负载虽然并没有下降很多，但是响应时间减低了很多，同时异常状态码比如 499 也减少了非常多。

不过如果还要进一步的减少负载的话，还得开启 OPcache: {% post_link php-opcache %}

---
参考资料:
- [php-fpm.conf 全局配置段](https://www.php.net/manual/zh/install.fpm.configuration.php)
- [PHP-FPM 进程管理的三种模式](https://juejin.cn/post/7018583928978014245)
- [对线面试官：php-fpm优化总结](https://learnku.com/articles/68365)

