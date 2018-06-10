---
title: 关于 XMLHttpRequest 对象需要注意的地方
date: 2018-06-11 00:46:09
tags: js
categories: 前端相关
---
这段时间重新理了一下ajax技术，并重新理了一下XMLHttpRequest 对象，发现还是有几个地方比较容易忘记或者是需要注意的几个地方。
### Ajax和XMLHttpRequest不是同一个东西
Ajax不等同于XMLHttpRequest，细究起来它们两个是属于不同维度的2个概念。
ajax是一种技术方案。它依赖的是现有的CSS/HTML/Javascript，而其中最核心的依赖是浏览器提供的XMLHttpRequest对象，是这个对象使得浏览器可以发出HTTP请求与接收HTTP响应。
{% blockquote What is AJAX? http://www.tutorialspoint.com/ajax/what_is_ajax.htm %}
AJAX stands for Asynchronous JavaScript and XML. AJAX is a new technique for creating better, faster, and more interactive web applications with the help of XML, HTML, CSS, and Java Script.
{% endblockquote %}
用一句话来总结两者的关系：我们使用XMLHttpRequest对象来发送一个Ajax请求。
<!--more-->
### XMLHttpRequest标准分为Level 1和Level 2
XMLHttpRequest Level 1主要存在以下缺点：
1. 受同源策略的限制，不能发送跨域请求；
2. 不能发送二进制文件（如图片、视频、音频等），只能发送纯文本数据；
3. 在发送和获取数据的过程中，无法实时获取进度信息，只能判断是否完成；

那么Level 2对Level 1 进行了改进，XMLHttpRequest Level 2中新增了以下功能：
1. 可以发送跨域请求，在服务端允许的情况下；
2. 支持发送和接收二进制数据；
3. 新增formData对象，支持发送表单数据；
4. 发送和获取数据时，可以获取进度信息；
5. 可以设置请求的超时时间；

level 2的功能可以看这篇文章[XMLHttpRequest Level 2 使用指南](http://www.ruanyifeng.com/blog/2012/09/xmlhttprequest_level_2.html)，文章中对新增的功能都有具体代码示例。

### XMLHttpRequest兼容性
1. IE8/IE9、Opera Mini 完全不支持xhr对象
2. IE10/IE11部分支持，不支持 xhr.responseType为json
3. 部分浏览器不支持设置请求超时，即无法使用xhr.timeout
4. 部分浏览器不支持xhr.responseType为blob
5. IE10 支持xhr了，但是不支持 withCredentials 属性，也就是跨域的时候，没法带cookie过去(即服务端没法读取到cookie)，因此如果在IE10，有用到xhr跨域请求的话，并且服务端需要校验cookie信息的，那么只能将cookie读出来，然后作为参数带过去。

### 设置request header
发送ajax的时候，xhr提供了setRequestHeader来允许我们修改请求 header。
{% codeblock lang:js %}
void setRequestHeader(DOMString header, DOMString value);
{% endcodeblock %}
但是这边要注意几个问题：
1. header 的 key 大小写不敏感
2. setRequestHeader必须在open()方法之后，send()方法之前调用，否则会抛错；
3. setRequestHeader可以调用多次，最终的值不会采用覆盖override的方式，而是采用追加append的方式。

{% codeblock lang:js %}
var client = new XMLHttpRequest();
client.open('GET', 'demo.cgi');
client.setRequestHeader('X-Test', 'one');
client.setRequestHeader('X-Test', 'two');
// 最终request header中"X-Test"为: one, two
client.send();
{% endcodeblock %}
这边还需要注意一个问题： 就是如果要设置自定义头部的话，比如客户端用来做JWT校验的 X-TOKEN，如果是跨域的话，那么服务端是要处理的，也就是服务端要允许接受这个头部才行(如果没有的话，会报跨域错误)，以php为例，那么就要设置CORS的其中一个跨域头为：
{% codeblock lang:js %}
$response->header('Access-Control-Allow-Headers', 'X-TOKEN');
{% endcodeblock %}
当然这个要改到服务端的代码，如果服务端的跨域默认允许客户端自己发送他们自己的自定义头部的话，可以这样设置：
{% codeblock lang:js %}
$response->header('Access-Control-Allow-Headers', $request->header('Access-Control-Request-Headers'));
$response->header('Access-Control-Allow-Headers', 'origin, content-type');
{% endcodeblock %}
其实就是把client的请求头部全部设置为跨域所能接受的头部。

### 获取response header
xhr提供了2个用来获取响应头部的方法：getAllResponseHeaders和getResponseHeader。前者是获取 response 中的所有header 字段，后者只是获取某个指定 header 字段的值。另外，getResponseHeader(header)的header参数不区分大小写。
xhr在获取响应头部的时候，不是所有的头部都可以获取的：
1. <font color=red>Set-Cookie</font>、<font color=red>Set-Cookie2</font>这2个字段，无论是同域还是跨域请求, 都没法获取，这个是因为 [W3C的 xhr 标准中做了限制](https://www.w3.org/TR/XMLHttpRequest/)
2. 对于跨域请求，客户端允许获取的response header字段只限于“<font color=red>simple response header</font>”和“<font color=red>Access-Control-Expose-Headers</font>” ，这个是因为[W3C 的 cors 标准对于跨域请求也做了限制](https://www.w3.org/TR/cors/#access-control-allow-credentials-response-header)

simple response header 只有这6个基本字段：Cache-Control、Content-Language、Content-Type、Expires、Last-Modified、Pragma。
如果想拿到其他字段，就必须在Access-Control-Expose-Headers里面指定，这个头部是服务端在返回跨域响应的时候，所返回的一个可选的头部。
比如以php来说，如果在跨域的时候，设置这个响应头部的时候：
{% codeblock lang:js %}
$response->header('Access-Control-Expose-Headers', 'x-redirect');
{% endcodeblock %}
那么我们就可以通过 xhr.getResponseHeader("x-redirect") 来获取这个服务端返回的自定义头部

### 设置请求的超时时间
XMLHttpRequest提供了timeout属性来允许设置请求的超时时间。
{% codeblock lang:js %}
xhr.timeout
{% endcodeblock %}
单位：milliseconds 毫秒
默认值：0，即不设置超时。
这边有涉及到到一个记时的开始时间和结束时间：
开始时间：调用xhr.send()方法的时候
结束时间：xhr.loadend事件触发的时候
不过有两个点需要注意：
1. 可以在 send()之后再设置此xhr.timeout，但计时起始点仍为调用xhr.send()方法的时刻。
2. 当xhr为一个sync同步请求时，xhr.timeout必须置为0，否则会抛错。

如果超时，那么对应就是 ontimeout 事件会被触发。

### 如何发一个同步请求
xhr默认发的是异步请求，但也支持发同步请求（当然实际开发中应该尽量避免使用）。到底是异步还是同步请求，由xhr.open（）传入的async参数决定。
也就是当async为false的时候，其实就是发送同步请求，当xhr为同步请求时，有如下限制：
1. xhr.timeout必须为0
2. xhr.withCredentials必须为 false
3. xhr.responseType必须为""（注意置为"text"也不允许）

若上面任何一个限制不满足，都会抛错，而对于异步请求，则没有这些参数设置上的限制。
而且应该尽量避免使用sync同步请求，为什么呢？
因为我们无法设置请求超时时间（xhr.timeout为0，即不限时）。在不限制超时的情况下，有可能同步请求一直处于pending状态，服务端迟迟不返回响应，这样整个页面就会一直阻塞，无法响应用户的其他交互。
另外，标准中并没有提及同步请求时事件触发的限制，如在 chrome中，当xhr为同步请求时，在xhr.readyState由2变成3时，并不会触发 onreadystatechange事件，xhr.upload.onprogress和 xhr.onprogress事件也不会触发。

### 如何获取上传、下载的进度
可以通过onprogress事件来实时显示进度，默认情况下这个事件每50ms触发一次。需要注意的是，上传过程和下载过程触发的是不同对象的onprogress事件：
1. 上传触发的是xhr.upload对象的 onprogress事件
2. 下载触发的是xhr对象的onprogress事件

{% codeblock lang:js %}
xhr.onprogress = updateProgress;
xhr.upload.onprogress = updateProgress;
function updateProgress(event) {
    if (event.lengthComputable) {
      var completedPercent = event.loaded / event.total;
    }
 }
{% endcodeblock %}

### xhr.withCredentials与 CORS 的关系
我们都知道，在发同域请求时，浏览器会将cookie自动加在request header中。但大家是否遇到过这样的场景：在发送跨域请求时，cookie并没有自动加在request header中。
造成这个问题的原因是：在CORS标准中做了规定，默认情况下，浏览器在发送跨域请求时，不能发送任何认证信息（credentials）如"cookies"和"HTTP authentication schemes"。除非xhr.withCredentials为true（xhr对象有一个属性叫withCredentials，默认值为false）。
所以根本原因是cookies也是一种认证信息，在跨域请求中，client端必须手动设置xhr.withCredentials=true，且server端也必须允许request能携带认证信息（即response header中包含Access-Control-Allow-Credentials:true），这样浏览器才会自动将cookie加在request header中。
<font color=red>另外，要特别注意一点，一旦跨域request能够携带认证信息，server端一定不能将Access-Control-Allow-Origin设置为*，而必须设置为请求页面的域名。</font>

以php为例，如果要设置cors的话，中间件的标准代码如下：
{% codeblock lang:js %}
public function handle(Request $request, Closure $next)
{
    $response = $next($request);
    $origin = $request->header('Origin');
    $origin = rtrim($origin, '/');
    $origin = isset($_SERVER['HTTP_ORIGIN']) ? $_SERVER['HTTP_ORIGIN'] : $origin;
    // 判断是否是合法域名，这个方法自己实现，反正这个origin一定要在允许跨域的白名单之内
    if(!Http::checkIsVerifyDomain($origin)){
        $origin = 'http://yourdomain.com';
    }
    $response->header('Access-Control-Allow-Methods', 'HEAD, GET, POST, PUT, PATCH, DELETE');
    // 如果有客户端有自定义头部的话，那么这个也要加上去
    // $response->header('Access-Control-Allow-Headers', $request->header('Access-Control-Request-Headers'));
    $response->header('Access-Control-Allow-Headers', 'origin, content-type');
    $response->header('Access-Control-Allow-Origin', $origin);
    $response->header('Access-Control-Allow-Credentials', 'true');
    return $response;
}
{% endcodeblock %}

### xhr相关事件
1. XMLHttpRequestEventTarget接口定义了7个事件：
    1. onloadstart
    2. onprogress
    3. onabort
    4. ontimeout
    5. onerror
    6. onload
    7. onloadend
2. 每一个XMLHttpRequest里面都有一个upload属性，而upload是一个XMLHttpRequestUpload对象
3. XMLHttpRequest和XMLHttpRequestUpload都继承了同一个XMLHttpRequestEventTarget接口，所以xhr和xhr.upload都有第一条列举的7个事件,而onreadystatechange是XMLHttpRequest独有的事件

所以这么一看就很清晰了：
xhr一共有8个相关事件：7个XMLHttpRequestEventTarget事件+1个独有的onreadystatechange事件；而xhr.upload只有7个XMLHttpRequestEventTarget事件。

当请求一切正常时，相关的事件触发顺序如下：
1. 触发xhr.onreadystatechange(之后每次readyState变化时，都会触发一次)
2. 触发xhr.onloadstart
    // 传阶段开始：
3. 触发xhr.upload.onloadstart
4. 触发xhr.upload.onprogress
5. 触发xhr.upload.onload
6. 触发xhr.upload.onloadend
    //上传结束，下载阶段开始：
7. 触发xhr.onprogress
8. 触发xhr.onload
9. 触发xhr.onloadend

---

参照的资料：
如果想快速又比较全面的了解xhr对象，那么就看这个[你真的会使用XMLHttpRequest吗？](https://segmentfault.com/a/1190000004322487#articleHeader13)
想真正搞懂XMLHttpRequest，最靠谱的方法还是看 [W3C的xhr 标准](https://www.w3.org/TR/XMLHttpRequest/);
想了解跨域请求，则可以参考[W3C的 cors 标准](https://www.w3.org/TR/cors/);
