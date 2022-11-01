---
title: bug 追踪系统 Sentry (4) -- 关联 sourceMap
date: 2022-11-01 16:49:22
tags: Sentry
categories: 实用工具集
---
## 前言
通过 {% post_link sentry-3 %} 我们已经知道怎么在项目中引入 sentry 的 sdk，并且将 error 抛送到线上。 但是线上的代码都是通过构建打包压缩过的，基本上没有可读性。

如果抛上去查看的话，错误堆栈长这样子，很难分析:

![](1.png)

如果能像开发环境那样子，可以在控制台查看 sourceMap 的话，就可以直接查看对应的源代码就好了。 Sentry 有提供这个功能。

## 原理
我们都知道构建生成的 sourceMap 不能上传到 release 的线上环境，因为会有安全问题(源代码泄露)。 所以一般也就在开发环境，才会用 sourceMap 去 debug。

那么 sentry 可以使用 sourceMap 原理很简单，就是我们将源代码(包含 sourceMap) 上传到 sentry 后台，他会通过 release 版本来关联， 其实就是结合每次的 release 版本号来管理不同版本号的 sourceMap

因为我们是自建的，所以不用担心代码泄露问题，可以放心的用。

这一块 sentry 官方是有文档的: [Sentry - JavaScript - Source Maps](https://docs.sentry.io/platforms/javascript/sourcemaps/), 为此还提供了一个 cli 工具来上传这些 sourceMap: [Command Line Interface](https://docs.sentry.io/product/cli/)
<!--more-->
> ps: 他虽然有提供了一个 webpack 的插件(原理也是对 cli 进行封装)，但是其他的构建比如 rollup， 原生的都没有， 所以我们直接用 sentry-cli 这个工具来实现

## 版本的概念
在实操之前， 有一个概念要说一下， 就是版本 release 的概念， 顾名思义，就是发布的线上程序的版本号，也是 init 函数的一个参数，具体看官方文档: [Releases & Health](https://docs.sentry.io/platforms/javascript/configuration/releases/)

因为不同的线上版本的代码肯定不一样的，所以 sourceMap 也会不一样，所以 sourceMap 一定是跟版本走的， 所以如果这个版本上线了之后， 如果要查看这个版本的 issue 的源代码， 就要跟着上传这个版本的 sourceMap 才行， 这个是一一对应的

关于版本的指定也很简单，以 {% post_link sentry-3 %} 这节的 vue demo 为例，只需要加上 release 参数就可以设置 issue 的版本了
```javascript
Sentry.init({
  app,
  dsn: "http://ff67a890253e4a1ba5874656e8afbd77@43.xx.xx.96:9000/3",
  release: "test_vue@0.0.6", // 设置版本号
  integrations: [
    new BrowserTracing({
      routingInstrumentation: Sentry.vueRouterInstrumentation(router),
      tracingOrigins: ["localhost", "my-site-url.com", /^\//],
    }),
  ],
  // Set tracesSampleRate to 1.0 to capture 100%
  // of transactions for performance monitoring.
  // We recommend adjusting this value in production
  tracesSampleRate: 1.0,
});
```
然后 sentry 后台就有单独的 release 的 tab 页面可以直接查看

![](2.png)

从截图来看，我这个项目已经被我过渡到了 0.0.8 这个版本。 release 的格式是 `package@version`, `@` 前面的 package 可以随便取，也可以不要，直接后面的 `version` 即可。

## 具体操作
接下来进入具体的实操

### 1. 安装 cli 工具
首先肯定是要安装这个 cli 工具，官方有提供了多种安装方式: [cli Installation](https://docs.sentry.io/product/cli/installation/), 我这边采用最简单的方式，就是用 npm 直接全局安装就行了

```javascript
npm install -g @sentry/cli --unsafe-perm
```

这样子就安装好了

### 2. setup 配置
接下来进行配置，也有官方文档: [Configuration and Authentication](https://docs.sentry.io/product/cli/configuration/)

因为我们是自建的，所以要指定 url 为我们自己的服务器， 用这个指令

```javascript
sentry-cli --url http://43.xx.xx.96:9000 login
```

```javascript
F:\example_code\github\sentry_demo\test-vue>sentry-cli --url http://43.xx.xx.96:9000 login
This helps you signing in your sentry-cli with an authentication token.
If you do not yet have a token ready we can bring up a browser for you
to create a token now.

Sentry server: 43.xx.xx.96
Open browser now? [y/n] y
Enter your token: 78ca87c647a94f84b38d98b01cae9adfddbcb59262914825855020a14b094175
Valid token for user kebingzao@gmail.com

Stored token in C:\Users\admin\.sentryclirc
```

会引导你打开浏览器然后去生成一个 api token，然后再让你填入这个你刚才在后台生成的 token，最后生成一个 `.sentryclirc` 文件，这个就是上传的授权文件

![](3.png)

这个 token 的话，至少要勾选这几个权限(默认就有勾选的):

```javascript
project:read
project:releases
org:read
```

> ps: 看网上其他资料有说还要再勾上 `project:write` 权限，但是我试了一下不需要勾选也能用，而且官方文档确实只指定以上那 3 个权限就足够了

然后查看了一下 `.sentryclirc` 文件，发现就只有一个 auth token:
```javascript
[auth]
token=78ca87c647a94f84b38d98b01cae9adfddbcb59262914825855020a14b094175
```

### 3. 程序指定版本打包并生成 sourceMap
接下来项目就指定一个版本 (就上面 vue 代码指定的 test_vue@0.0.6 )，然后打包生成 sourceMap

![](4.png)

### 4. 使用 cli 上传 sourceMap
这一块官方也是有文档: [Upload Source Maps](https://docs.sentry.io/product/cli/releases/#sentry-cli-sourcemaps), 指令如下:
```javascript
sentry-cli releases files "$VERSION" upload-sourcemaps /path/to/sourcemaps
```

具体实操如下:
```text
F:\example_code\github\sentry_demo\test-vue>sentry-cli --url http://43.xx.xx.96:9000/ releases  -o sentry -p test_vue files test_vue@0.0.6 upload-sourcemaps ./dist/js --url-prefix ~/js/
> Found 4 release files
> Analyzing 4 sources
> ~/js/app.92921d72.js.map
> Analyzing completed in 0.055s                                                                                                                                                                                                     4
> Rewriting sources
> ~/js/chunk-vendors.aac8c5cb.js
> Rewriting completed in 0.019s                                                                                                                                                                                                     4
> Adding source map references
> Bundling files for upload...
> Bundling files for upload... ~/js/chunk-vendors.aac8c5cb.js.map                                                                                                                                                                   4
████████████████████████████████████████████████████████████████████████████████████████████████████████████████░░░░░
> Bundling completed in 0.081s
> Optimizing completed in 0.002s
> Uploading release files...
> Uploading release files...                                                                                                                                                                                                        ░
> Uploading release files...                                                                                                                                                                                                        ░
> Uploading release files...                                                                                                                                                                                                        ░
> Uploading completed in 1.619s
> Uploaded release files to Sentry
> Processing completed in 0.326s
> File upload complete (processing pending on server)
> Organization: sentry
> Project: test_vue
> Release: test_vue@0.0.6
> Dist: None

Source Map Upload Report
  Minified Scripts
    ~/js/app.92921d72.js (sourcemap at app.92921d72.js.map)
    ~/js/chunk-vendors.aac8c5cb.js (sourcemap at chunk-vendors.aac8c5cb.js.map)
  Source Maps
    ~/js/app.92921d72.js.map
    ~/js/chunk-vendors.aac8c5cb.js.map
```
发现了没有，这个上传的指令其实挺长的, 接下来讲一下各参数含义:
```text
sentry-cli --url http://43.xx.xx.96:9000/ releases  -o sentry -p test_vue files test_vue@0.0.6 upload-sourcemaps ./dist/js --url-prefix ~/js/
```

- `--url` -> 要上传的 sentry 服务端点
- `-o` -> 组织名称，在 Organization settings 可以看到，没有特别设置的话，就是默认的 `sentry` 
- `-p` -> 项目名称
- `--url-prefix` -> 线上 js 的相对路径， `~` 表示根目录， 这个地方很坑， 只能用 `~/js/` 这种写法才能匹配到，用这种绝对路径 `/js/` 或者 这种相对路径 `./js/`, 都会匹配不到

![](5.png)

上传上去之后，我们是可以通过这个指令来查看某一个版本的 sourceMap 文件的:
```text
F:\example_code\github\sentry_demo\test-vue>sentry-cli --url http://43.xx.xx.96:9000/ releases  -o sentry -p test_vue  files test_vue@0.0.6 list
+------------------------------------+--------------+-------------------------------+----------+
| Name                               | Distribution | Source Map                    | Size     |
+------------------------------------+--------------+-------------------------------+----------+
| ~/js/app.92921d72.js               |              | app.92921d72.js.map           | 13.94KB  |
| ~/js/app.92921d72.js.map           |              |                               | 15.77KB  |
| ~/js/chunk-vendors.aac8c5cb.js     |              | chunk-vendors.aac8c5cb.js.map | 202.96KB |
| ~/js/chunk-vendors.aac8c5cb.js.map |              |                               | 1.29MB   |
+------------------------------------+--------------+-------------------------------+----------+
```

可以看到 sourceMap 已经传上去了，而且对应匹配顺序也有了。 然后接下来就可以在 后台的这个 release 版本查看到这些上传的 sourceMap 了

![](6.png)

可以看到有 4 个 sourceMap 文件，点进去就可以看到详情文件

![](7.png)

### 5. 抛送 error 并查看
既然 test_vue@0.0.6  版本的 sourceMap 已经上传了，接下来就在 test_vue@0.0.6 的 release 环境下进行错误抛送，看看是否可以看到源代码
> 这边再次强调，这个 cli 的 版本号，一定要跟项目 init 初始化的 release 参数一致， 才能匹配上

![](8.png)

可以看到在 release 版本就可以看到这个 issue 了 (在 issue tab 页面也可以找到)，然后点进去看详情

![](9.png)

这时候堆栈显示的就会换成 sourceMap 解析之后的源代码了。

### 6. 配置优化
我们之前用来上传 sourceMap 的指令太长了
```text
sentry-cli --url http://43.xx.xx.96:9000/ releases  -o sentry -p test_vue files test_vue@0.0.6 upload-sourcemaps ./dist/js --url-prefix ~/js/
```
但是其实 url，organization， project 都可以事先在 `.sentryclirc` 文件设置默认选项，对于本例来说，改成:
```text
[defaults]
url=http://43.xx.xx.96:9000/
org=sentry
project=test_vue

[auth]
token=78ca87c647a94f84b38d98b01cae9adfddbcb59262914825855020a14b094175
```
然后上述两个指令就可以简化成:
```text
sentry-cli releases files test_vue@0.0.6 upload-sourcemaps ./dist/js --url-prefix ~/js/
sentry-cli releases files test_vue@0.0.6 list
```

## cli 其他指令
sentry cli 除了用来上传 sourceMap 之外，还有很多指令，具体可以看官方文档，我这边只是列出一些后续比较常用的指令

### 1. 列出某一个版本的 sourceMap 列表
这个就是上述的这个指令, 不在赘述
```text
sentry-cli releases files "$VERSION" list
```

### 2. 删除某一个版本的 sourceMap 文件 (包含清空)
```text
sentry-cli releases files "$VERSION" delete NAME_OF_FILE
sentry-cli releases files "$VERSION" delete --all
```
具体操作:
```text
F:\example_code\github\sentry_demo\test-vue>sentry-cli releases files test_vue@0.0.2 delete --all
All files deleted.
```

### 3. 删除某一个 release 版本
```text
sentry-cli releases delete "$VERSION"
```
不过这个操作是有限制，如果当前有未解决的 issue 的话，是会失败的
```text
F:\example_code\github\sentry_demo\test-vue>sentry-cli releases delete test_vue@0.0.7
error: API request failed
  caused by: sentry reported an error: This release is referenced by active issues and cannot be removed. (http status: 400)

Add --log-level=[info|debug] or export SENTRY_LOG_LEVEL=[info|debug] to see more output.
Please attach the full debug log to all bug reports.
```
所以要先把这个版本的 issue 先 close 掉，才能删除
```text
F:\example_code\github\sentry_demo\test-vue>sentry-cli releases delete test_vue@0.0.7
Deleted release test_vue@0.0.7!
```

## 总结
通过 sentry 的 cli 工具我们可以很容易的针对线上的某一个 release 版本的代码进行对应的 sourceMap 的文件的上传，然后在后台查看 bug 的时候，可以直接查看源代码。

如果需要注意一点的是，对于 vue 项目来说，有些第三方公共组件的代码会被打成 vendor.js, 这个 js 是很大的，他对应的 sourceMap 也会更大，这样子不仅上传慢，后台解析的时候，其实也会慢， 所以针对这种情况，就不应该上传 vendor 的 sourceMap 文件了。

后续可以将 sentry-cli 工具结合到我们的 release 构建环境，这样子在构建成功的时候，可以同步将对应的 sourceMap 也上传上去。

---
相关 Sentry 系列文章:
- {% post_link sentry-1 %}
- {% post_link sentry-2 %}
- {% post_link sentry-3 %}
- {% post_link sentry-4 %}

---
参考资料:
- [Sentry Command Line Interface](https://docs.sentry.io/product/cli/)
- [Sentry Source Maps](https://docs.sentry.io/platforms/javascript/sourcemaps/)
- [Generating Source Maps](https://docs.sentry.io/platforms/javascript/sourcemaps/generating/)
- [线上bug追踪之Sentry release+sourceMap（二）](https://zhuanlan.zhihu.com/p/75534743)
- [Sentry前端部署拓展篇（sourcemap关联、issue关联、release控制）](http://t.zoukankan.com/10manongit-p-12805199.html)

