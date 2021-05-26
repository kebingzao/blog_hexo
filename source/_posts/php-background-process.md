---
title: php 接口提前响应返回，然后继续执行后台逻辑
date: 2021-05-25 19:10:49
tags: php
categories: php相关
---
## 前言
前段时间有一个 php 的业务接口，响应时间很长， 超过了 10s， 然后 php-fpm 就显示超时了，然后 nginx 就返回 502 了。

虽然后面通过配置 `php-fpm.conf` 的这个设置，将其改为 20s (之前是设置为 10s)
```php
[kbz@ip-10-1-2-146 ~]$  cat /usr/local/php/etc/php-fpm.conf | grep te_timeout
request_terminate_timeout = 20s
```

然后让前端的超时时间也改成 30s。 这样子接口是可以跑了， 但是还是慢。

后面发现是这个接口会去执行一些很复杂的运算逻辑，而且涉及到一些库操作。 但是在这之前需要返回给前端的数据都已经有了。

所以这一块执行时间比较长的运算逻辑其实是可以做成异步。 而我们目前项目的异步都是通过抛消息队列去处理的。 但是因为这一块逻辑涉及的相关联的函数比较多， 所以在消息队列那边再写一套也不现实。 如果是抛送 curl 单独执行这一部分逻辑的话，一旦响应时间超过了 20s， 也会 502。
<!--more-->
所以就打算在同一个接口中， 先将要给前端的数据先返回， 然后这一部分逻辑再慢慢的在后台运行。

## 具体操作
php 程序(我目前用的还是 5.6 版本) 是可以在接口中，进行一些异步操作的， 不过写法有点特殊。 以本例来说， 进行异步操作之前， 要先将需要的结果集返回给前端， 然后再慢慢进行这些异步操作。 这个流程就会涉及到以下几个函数:

### 1. ob_end_clean()
这个是清除之前的缓冲内容，这是必需的，如果之前的缓存不为空的话，里面可能有 http 头或者其它内容，导致后面的内容不能及时的输出

### 2. header("Connection: close")
告诉浏览器，连接关闭了，这样浏览器就不用等待服务器的响应。

### 3. header("HTTP/1.1 200 OK")
发送 200 状态码，要不然可能浏览器会重试，特别是有代理的情况下

### 4. ob_start()
开启当前代码缓冲

### 5. ob_end_flush()
输出当前缓冲

### 6. flush()
输出PHP缓冲

### 7. ignore_user_abort(true)
在关闭连接后，继续运行php脚本

### 8. set_time_limit(0)
no time limit，不设置超时时间（根据实际情况使用）

### 9. fastcgi_finish_request()
这个在有用 fpm 的时候，会用到， 也是将提前返回响应，然后接下来的逻辑后台执行。

### 封装成函数
我将其抽成一个异步返回的函数， 最后的代码就是:
```php
// 异步的成功返回函数
public function returnSuccessJsonDataAsync($otherReturnData = [], $timeout = 60, $allowCors = true){
    $msgData = ['code' => 1, 'msg' => 'Success'];
    if(empty($otherReturnData)){
        $data = $msgData;
    }else{
        $data = array_merge($msgData, $otherReturnData);
    }
    // 接下来提前告诉 浏览器返回， 其他的后台允许
    ob_end_clean();
    //告诉浏览器，连接关闭了，这样浏览器就不用等待服务器的响应
    header("Connection: close");
    header("HTTP/1.1 200 OK");
    ob_start();
    $str = json_encode($data);
    header('Content-type: application/json');
    header('Content-Length: ' . strlen($str));
    
    // 如果允许跨域，那么就设置跨域头
    if($allowCors){
        $origin = Yii::$app->request->getHeaders()->get('Origin');
        header("Access-Control-Allow-Origin: {$allowCors}");
        header('Access-Control-Allow-Credentials: true');
        header('Access-Control-Allow-Methods: POST, GET, OPTIONS');
        header('Access-Control-Allow-Headers: origin, content-type');
        header('Access-Control-Max-Age: 86400');
    }
    
    echo $str;

    ob_end_flush();
    if(ob_get_length()){
        ob_flush();
    }
    flush();
    // yii或yaf默认不会立即输出，加上此句即可（前提是用的fpm）
    if (function_exists("fastcgi_finish_request")) {
        fastcgi_finish_request(); // 响应完成, 立即返回到前端,关闭连接
    }

    /******** background process starts here ********/
    //在关闭连接后，继续运行php脚本
    ignore_user_abort(true);
    //no time limit，不设置超时时间（根据实际情况使用）
    set_time_limit($timeout);
}
```

这个方法注意几个细节:
1. `set_time_limit` 并没有设置为 0 (不限超时), 而是设置一个比较合理的值，比如 60s， 因为我认为 60s 之内还没办法执行完成的话， 要么逻辑有问题，要么直接放到消息队列去消费，反正不适合当前场景。 而且一旦设置为 0 的话，一旦代码有问题，进程就会迟迟无法释放，而 fpm 能分配的进程是有限的，就会导致 php-cgi 的进程数用满了， 新的请求过来，因为没有新的进程来处理了，所以 nginx 就会返回 502 Bad Gateway。
2. 这边输出给前端，用的是 `echo` 来输出，从而保证接下来的逻辑会在请求返回之后可以继续执行。 接下来不能用的 `die;`  或者 `Yii::$app->end();` 这种语法，不然整个请求进程都会被结束，也就无法继续后台执行了。


调用的方式也非常简单， 当之前的操作得到数据之后， 就先将这些数据返回给前端:
```php
// 提前返回给前端， 剩下来的逻辑，就后台继续运行
$this->returnSuccessJsonDataAsync(['data' => $data]);

//=========== 接下来的逻辑后台继续运行=================
$eventData = new EventData();
$eventData->info = [
    'device_id' => $deviceId,
]
// 执行一些很耗时的逻辑
```

不过在我的实践中，因为该项目用的是 php YII2 框架， log 的输出也要等 后台的程序一起执行完之后， 才会一起输出到文件中。

也就是假设请求 1s 的时候，请求先返回了， 然后继续执行后台程序，耗时 9s， 这时候到 10s 的时候，才会把对应的 log 打出来。


---

参考资料:
- [另类方式实现PHP后台运行](https://zhuanlan.zhihu.com/p/56455344)
- [PHP返回前端后继续执行后台任务](https://ranjuan.cn/php-run-background/)





