---
title: 使用 Swagger + Yapi 构建界面优雅的服务端接口 & 测试文档
date: 2020-12-16 10:33:08
tags: swagger
categories: 实用工具集
---
## 前言
相信做过开发的，都或多或少地被接口文档折磨过。前端经常抱怨后端给的接口文档与实际情况不一致。后端又觉得编写及维护接口文档会耗费不少精力，经常来不及更新。其实无论是前端调用后端，还是后端调用后端，都期望有一个好的接口文档。但是这个接口文档对于程序员来说，就跟注释一样，经常会抱怨别人写的代码没有写注释，然而自己写起代码起来，最讨厌的，也是写注释。所以仅仅只通过强制来规范大家是不够的，随着时间推移，版本迭代，接口文档往往很容易就跟不上代码了。

## 在 gitlab 私有库上写 wiki 文档
早期我们团队的后端接口文档都是写在我们内部的 gitlab 私有库项目中的 wiki 上， 类似于:
<!--more-->

![1](1.png)

但是这样子就会有上面遇到的问题，就是随着接口的迭代，后面的文档的维护也逐渐跟不上。后面就很形式了。

## 使用 swagger 来生成接口文档并测试
后面使用 [swagger](https://github.com/swagger-api/swagger-ui) 来生成项目的接口文档，并且还支持接口请求测试。

Swagger 是一个规范和完整的框架，用于生成、描述、调用和可视化 RESTful 风格的 Web 服务。

总体目标是使客户端和文件系统作为服务器以同样的速度来更新。文件的方法、参数和模型紧密集成到服务器端的代码，允许 API 来始终保持同步。Swagger 让部署管理和使用功能强大的 API 从未如此简单。

### 1. 搭建 swagger ui
因为 swagger ui 是一个静态站点，所以只要本地有 web server 就可以了。首先先到 [swagger github](https://github.com/swagger-api/swagger-ui) git clone 一下最新的版本

![1](2.png)

虽然这个项目文件很多，但是我们其实只需要 dist 目录下的静态资源文件而已

![1](3.png)

所以我们将 dist 下面的文件放到 web server 下的 swagger 目录下，这样子就可以直接访问了

![1](4.png)

这边默认有一个预填的 json
```text
http://petstore.swagger.io/v2/swagger.json
```
后面可以在 `index.html` 修改这个配置, 改成我们自己的项目所生成的 json 文件

![1](5.png)

而这个 json 文件其实就是 swagger ui 文档配置文件。 他就是通过这个 json 文档来生成接口和测试文档的。 这个 json 格式是这样子的:

![1](6.png)

这时候替换完成之后，我们就可以看到我们这个项目的文档了:

![1](7.png)

这时候还不能进行接口测试， 因为这个是本地搭建的，所以直接请求线上的项目，会有跨域问题。 不过如果部署到线上同一个域，或者接口有设置 cors 的话，就可以了。

### 2. 查看接口并且测试
一旦你的 json 文件有对接口进行文档配置的话，那么就可以在 swagger 上直接查看文档并且还可以进行接口测试。

![1](8.png)

点击 `try it out` 按钮就可以进行接口测试

![1](9.png)

非常的方便。

### 3. json 文件的生成
我们可以看到 swagger ui 的所有资源都是静态的， 全部是通过加载这个我们所设置的 json 文件来生成接口文档的。 那么这个 json 文件是怎么生成的 ？？  以我们的这个 php 项目为例，找到这个 doc 接口， 可以看到代码很简单
```php
/**
 * 生成api doc
 * @return Swagger\Annotations\Swagger
 */
public function doc()
{
    $swaggerStr = Swagger\scan(base_path('app'));
    // header('Content-Type: application/json');
    $data = json_decode($swaggerStr, true);
    $data['host'] = config('app.doc_host');
    // return $swaggerStr;
    return $this->echoJson($data);
}
```
他其实就是调用了一个 swagger 的第三方包
 
![1](10.png)
 
然后根据每一个接口所设置的符合规则的注释 (要符合 swagger 语法)，最后生成 json 文档， 所以正常生成的 json 文档是这样子的

![1](11.png)

当然如果语法有问题的话，就会报错

![1](12.png)

### 4. swagger 语法
要为接口生成接口文档，那么就需要在对应的接口写上符合 swagger 语法的注释，以上面截图的 `des` 加密接口为例，那么他的 swagger 的语法注释就是:
```php
    /**
     * @SWG\Definition(
     *         definition="DesText",
     *         required={"text"},
     *         @SWG\Property(
     *             property="text",
     *             type="string",
     *             format="string",
     *             description="需要加密解密的string"
     *         ),
     *     )
     */


    /**
     * *   @SWG\Post(
     *     path="/utils/des/encrypt",
     *     tags={"Des"},
     *     summary="Des Encrypt/ json/text 加密为q串(旧 Q 串)",
     *     description="返回200即成功",
     *     @SWG\Parameter(
     *         name="加密消息串",
     *         in="body",
     *         description="raw body为需要加密的串",
     *         required=true,
     *         @SWG\Schema(ref="#/definitions/DesText"),
     *     ),
     *     consumes={"application/json"},
     * produces={
     *         "application/json"
     *     },
     * @SWG\Response(
     *         response=200,
     *         description="成功",
     *     ),
     * @SWG\Response(
     *         response="500",
     *         description="加密失败失败",
     *         @SWG\Schema(ref="#/definitions/Error")
     *     )
     * )
     * @param Request $request
     * @return mixed
     */
    public function encrypt(Request $request)
    {
        $key = $request->get('key');
        if ($key && $key != '' && $key != '{key}') {
            $this->rebuildDes($key);
        }
        $input = file_get_contents('php://input');
        //$input = $request->input('text');
        return $this->des->encrypt($input);
    }
```

所以只需要在写接口或者迭代接口的时候，补上对应的 swagger 注释，这样子就可以生成对应的接口文档了。 可以说解决了上面的痛点。

关于 swagger 的注释语法， 可以看: [Quick Annotation Overview](https://github.com/swagger-api/swagger-core/wiki/Swagger-2.X---Annotations#quick-annotation-overview)

## 使用 YApi 来搭建更优雅的文档界面
### 1. swagger ui 的界面问题
虽然 swagger ui 已经可以很好的处理接口文档和测试，但是当这个项目的接口很多的时候，要查找某一个接口，简直是噩梦，因为他也没有搜索功能， 也没有收藏功能，每次找接口， 全靠 chrome 浏览器的 `ctrl + f` 的全局搜索。 每次要用几个工具接口的时候，都要找到吐血

![1](1.gif)

而网上针对 swagger ui 的这个界面问题，也是吐槽巨多，而且功能也有点缺陷 (比如测试的时候，不能自定义 header， 没办法写自动化测试脚本)， 所以也出现一些很优秀的，并且可以兼容 swagger 的服务， 而我们要用的 YApi 就是其一, 他不仅可以增强测试功能，而且界面更优雅

### 2. YApi 简介
YApi是高效、易用、功能强大的API管理平台，旨在为开发、产品、测试人员提供更优雅的接口管理服务。YApi在Github上已累计获得了18K+Star，具有优秀的交互体验，YApi不仅提供了常用的接口管理功能，还提供了权限管理、Mock数据、Swagger数据导入等功能，总之功能很强大！

### 3. 安装
#### 3.1 环境准备
要安装 YApi 需要先安装 `nodejs` 和 `mongoDB`， 所以要先装好。 而 nodejs 和 mongoDB 我的本地环境之前就安装了。 这边就不多做文章

顺便说一下，我的本地环境是 `windows 7`

#### 3.2 安装 yapi-cli
> yapi-cli是YApi官方提供的安装工具，可以通过可视化界面来部署YApi服务，非常方便！

```text
F:\server-readme>npm install -g yapi-cli --registry https://registry.npm.taobao.org
npm WARN deprecated bson@1.0.9: Fixed a critical issue with BSON serialization documented in CVE-2019-2391, see https://bit.ly/2KcpXdo for more details
C:\Program Files\nodejs\yapi-cli -> C:\Program Files\nodejs\node_modules\yapi-cli\bin\yapi-cli
C:\Program Files\nodejs\yapi -> C:\Program Files\nodejs\node_modules\yapi-cli\bin\yapi-cli
+ yapi-cli@1.5.0
added 257 packages in 48.635s
```

安装成功后使用`yapi server`命令来启动YApi的可视化部署界面。
```text
F:\server-readme>yapi server
在浏览器打开 http://0.0.0.0:9090 访问。非本地服务器，请将 0.0.0.0 替换成指定的域名或ip
```
所以接下来访问 `http://127.0.0.1:9090/` 来安装 yapi

![1](13.png)

修改一下路径和其他参数

![1](14.png)

点击开始部署， 这时候会出现部署日志

![1](15.png)

后面初始化的时候，报错了， 本地的 mongo的 版本太低了。 至少要 2.6 的版本才行。

```text
Error:  error: MongoNetworkError: Server at 127.0.0.1:27017 reports maximum wire version 0, but this version of the Node.js Driver requires at least 2 (MongoDB 2.6), mongodb Authentication failed
```

后面我查了一下本地之前安装的 mongo， 发现是 2.4.4， 版本太低了

![1](16.png)

所以我们得重新装一个更高版本的 mongoDB, 只要是 2.6 以上，因为本地已经有一个旧版本的 mongoDB 了，所以默认端口的 27017 已经被用了， 所以新装的高版本的 mongoDB 就用 27018 的端口来运行

#### 3.3 安装高版本的 mongoDB，并用 27018 端口
通过 https://www.mongodb.com/try/download/community , 我们下载了 `4.0.21` 的版本。 安装完之后，在安装目录的 bin 目录下的 `mongod.cfg` 的启动端口改成 27018

![1](17.png)

同时改完之后，服务还要启动一下 (上面的那个 mongoDB 的服务是旧版本的)

![1](18.png)

而且这时候的版本就是 4.0.21 了， 终于符合了。

![1](19.png)

所以接下来继续部署，不过这时候要将部署界面的 数据库端口改成 27018， 这时候就成功了

![1](20.png)

#### 3.4 启动服务
既然部署成功了，那么接下来就是启动 YApi 服务了
```text
F:\my-yapi>node vendors/server/app.js
log: -------------------------------------swaggerSyncUtils constructor-----------------------------------------------
log: 服务已启动，请打开下面链接访问:
http://127.0.0.1:3008/
log: mongodb load success...
```

这样子服务就启动了，而且 YApi 跟 swagger ui 不一样，他是有用户权限管理的。所以要登录一下。

![1](21.png)

使用上面的管理员账号 admin@admin.com:ymfe.org 登录一下

![1](22.png)

### 4. 使用
#### 4.1 创建项目
接下来我们就要从 swagger 导入数据了， 首先要先建一个分组

![1](23.png)

然后在该分组下，点击添加项目

![1](24.png)

创建成功之后，可以看到里面的接口都是空的

![1](25.png)

#### 4.2 导入项目的 swagger json 文件

接下来我们就要将这个 swagger 项目的接口文档的 json 文件导进去， 点击 `数据管理`

![1](26.png)

点击上传， 这时候会弹一个提示

![1](27.png)

点击确认。 当导出成功之后，至此 我们的某一个项目的 Swagger 中的API接口已成功导入到 YApi，点击接口标签查看所有导入接口。 就可以看到接口列表有我们的接口了

![1](28.png)

而且整个格局非常的清晰。 点进去接口详情，可以看到具体的信息，包括备注

![1](29.png)

#### 4.3 设置接口请求 url
接下来试试接口运行功能，我们会发现默认的接口请求地址并不符合我们的要求，需要在环境配置中设置；

![1](30.png)

我们发现请求地址不是我们想要的， 所以这个得改一下， 所以点击 `环境配置`

![1](31.png)

点击环境配置

![1](32.png)

这样子就好了。

#### 4.4 安装 chrome 跨域插件
因为是本地环境搭建的，然后请求是业务服务接口，所以会有跨域请求 (后面让运维人员将这个 YApi 也放到线上去，就不会有跨域了)，Chrome 浏览器需要安装跨域请求插件，[下载地址](https://github.com/YMFE/cross-request/archive/master.zip)， 这边有 [安装文档](https://juejin.cn/post/6844904057707085832)

所以我们得装一下这个插件 ，不然发送按钮点击不了。

点击下载 zip 包， 解压， 然后到 chrome 的扩展程序那边点击 加载已解压的扩展程序

![1](33.png)

然后重新刷新一下 YApi 页面，这时候就发现发送按钮可以点击了

![1](34.png)

然后设置一下参数，最后点击发送，这样子就成功了

#### 4.5 设置 header
而且比之 swagger ui 更好的一点就是，他允许你可以对 header 进行配置。 因为我们有些接口就有安全校验的，而安全校验的token事实上是放到 header 进行请求的。 而不是放到 body 参数体的。

所以我们可以在`设置` 那边的 `环境配置`，对自定义的 header 进行设置。（是不是想到 postman）

![1](35.png)

点击保存，然后在请求的时候，就可以看到 请求的 header 有带上这个自定义的 header 头部了

![1](36.png)

这时候就可以看到自定义的 header 就会带过去了。 再测试带有 token header 的接口，特别好用。

### 5. 从 swagger 自动同步
当我们的接口修改了，API文档如何同步呢，我们可以通过`设置` -> `Swagger自动同步` 来开启自动同步功能，有三种数据同步模式可以选择, 我们选择的是`完全覆盖`, 完全将接口定义交给后端

![1](37.png)

采用默认值就行了，两分钟执行同步一次， 可以看到有 log 输出
```text
log: 定时器触发, syncJsonUrl:https://test-admin.xxx.com/v2/doc,合并模式:merge
log: 定时器触发, syncJsonUrl:https://test-admin.xxx.com/v2/doc,合并模式:merge
log: 定时器触发, syncJsonUrl:https://test-admin.xxx.com/v2/doc,合并模式:merge
log: 定时器触发, syncJsonUrl:https://test-admin.xxx.com/v2/doc,合并模式:merge
```

### 6. mock 功能
因为我们的接口定义都全部由后端来处理，所以一般不会用到 mock 功能。但是有时候会遇到接口后端还没有来得及写，但是前端就要联调了，那么就可以用这个 mock 功能，先给前端一个 mock 的接口和返回值。

举个例子，我从`公共分类`那边 新建了一个接口，叫 `获取自定义用户信息`

![1](40.png)

然后进行编辑， 设置 mock 返回值

![1](41.png)

最后只要请求 mock 地址 (每一个接口的显示详情下都有一个对应的 mock 地址)，就可以了

![1](42.png)

![1](43.png)

这样子就非常方便。

### 7.权限管理
YApi 是有权限管理的，如果有新的成员加入进来，需要查看API文档怎么办？ 简单分为三个步骤即可
1. 首先可以通过注册界面注册一个成员账号
2. 之后使用管理员账号登录，然后通过成员列表->添加成员，将用户添加到相应分组
3. 最后使用成员账号登录即可访问相应API文档了。

关于更多的关于 YApi 的权限管理，可以查看 [权限权利](https://hellosean1025.github.io/yapi/documents/manage.html)

## 总结
通过 Swagger + YApi 的组合，我们可以很好的实现项目的接口文档的编写和测试。 尤其是后者 YApi 的加入，解决了之前只用 Swagger ui 的几个痛点:
1. swagger 的界面太难搜索接口， 结合 YApi 可以很快的通过 tag 和 搜索来定位接口， 或者可以创建一个分类来当收藏夹用
2. swagger 测试接口的时候，没办法自定义 header 参数， YApi 可以
3. YApi 可以设置 mock 返回值，能让前端无需后台实现也可以调试接口
4. YApi 可以存放多个项目的接口文档， 而 swagger 要查看其它项目的文档，都得切换 json 链接
5. YApi 提供了权限管理功能，可以保证 API 文档的安全


---
参考文档:
- [当Swagger遇上YApi，瞬间高大上了！](https://juejin.cn/post/6906279483448393735)
- [Swagger界面丑、功能弱怎么破？用Postman增强下就给力了！](https://mp.weixin.qq.com/s/rbKUJAhv6WorFWgDNUDWTg)
- [YApi documents](https://hellosean1025.github.io/yapi/documents/index.html)
- [Swagger Quick Annotation Overview](https://github.com/swagger-api/swagger-core/wiki/Swagger-2.X---Annotations#quick-annotation-overview)

