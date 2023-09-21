---
title: 使用 verdaccio 搭建前端内部私有 npm 仓库
date: 2023-09-19 18:38:25
tags: 
- npm
- verdaccio
categories: node相关
---
## 前言
随着前端项目越来越多，尤其是内部的 js lib 库越来越多，怎么有效的管理 lib 库的发布，就变成一个很重要的问题了。

以之前做的 air-ui 来说，其实在进行版本发布的时候，为了保证不污染 gitlab 上的 master 分支，所以就专门写了脚本在版本发布的时候，进行分支切换来处理，具体步骤多又繁琐:
```text
1. 检查当前分支是否是远程的 master 分支，如果不是，返回错误
2. 执行 yarn dist， 生成 lib 目录
3. 将 lib 目录保存到一个 gitignore 的一个目录 lib_tmp
4. commit and push master 分支 （如果 status 有改变的话）
5. 切换到 master-release 分支，将 lib_tmp 的文件覆盖 lib 目录
6. commit and push master-release 分支
7. master-release 打 tag
8. 把临时文件 lib_tmp 删掉
```
> 具体细节查看 {% post_link air-ui-16 %}

这种流程是有一些缺点和问题的:
1. **发布流程复杂化：** 需要编写专门的脚本来确定哪些构建产物的代码需要发布，同时还需手动删除不必要的代码。
2. **容易产生混淆：** 使用 `git tag` 来发布代码可能会引起混淆，因为它通常用于标记版本，而不是发布代码。
3. **无法利用语义化版本控制的范围（例如：^1.0.0 或 ~1.0.0）来精确指定依赖的版本**

但是如果我们有自己的 npm 库的话，其实对于库的发布就变得非常简单，只需要简单的 3 步:
1. 更新`package.json`中的版本号
2. 代码构建
3. 最后执行`npm publish`命令，即发布成功

<!--more-->
根本不需要对项目的代码做耦合。 而且随着项目的迭代和新增，内部维护自己的 js 库，不管是组件库，还是工具库，或者是业务库，其实都会变得越来越多。因此是需要搭建自己的内部 npm 库的。它不仅会简化js库发布流程，还可以增强前端各项目通用代码流通。

## verdaccio 简介
我们使用 verdaccio 部署了npm 私有源, [Verdaccio](https://verdaccio.org/docs/next/what-is-verdaccio) 是一个轻量级的私有 npm 代理注册服务器，它可以帮助你建立自己的私有仓库，对 npm 包进行管理。以下是 Verdaccio 的一些主要功能：

1.  **私有包管理**：你可以使用 Verdaccio 来发布和存储你的私有包，这对于保护你的源代码和内部使用的包非常有用。
2.  **缓存代理**：Verdaccio 可以作为一个 npm 代理，它会缓存所有从 npmjs.com 下载的公共包，这样即使你失去了与 npmjs.com 的连接，你仍然可以安装这些包。
3.  **权限控制**：Verdaccio 支持用户认证和包权限控制，你可以控制谁可以访问和发布你的私有包。
4.  **易于扩展**：Verdaccio 支持插件，你可以使用插件来扩展 Verdaccio 的功能，例如添加新的认证方法，改变存储方式等。
5.  **兼容性**：Verdaccio 完全兼容 npmjs.com，你可以使用 npm 或者 yarn 等工具来与 Verdaccio 交互。
6.  **易于部署**：Verdaccio 可以在多种环境中部署，包括 Docker，Kubernetes，云服务等。
7.  **Web UI**：Verdaccio 提供了一个用户友好的 web 界面，你可以在这个界面上搜索和管理你的包。

接下来开始进入实操环节

## 安装
Verdaccio 有多种安装，因为是安装在服务端，因此为了不污染服务器全局环境，我这边是使用 docker 来安装: [Running Verdaccio using Docker](https://verdaccio.org/docs/next/docker)
> 我的安装环境是 CentOS 7

### 安装 docker
```text
[root@VM-64-9-centos ~]# sudo yum install -y yum-utils
[root@VM-64-9-centos ~]# sudo yum-config-manager \
> --add-repo \
> https://download.docker.com/linux/centos/docker-ce.repo
[root@VM-64-9-centos ~]# sudo yum install docker-ce docker-ce-cli containerd.io
[root@VM-64-9-centos ~]# sudo systemctl start docker
root@VM-1-3-centos ~]# docker -v
Docker version 24.0.6, build ed223bc
```

### 安装镜像
```text
V_PATH=/etc/verdaccio; docker run -d -it  --name verdaccio   \
  -p 4873:4873   \
  -v $V_PATH/conf:/verdaccio/conf   \
  -v $V_PATH/storage:/verdaccio/storage   \
  -v $V_PATH/plugins:/verdaccio/plugins  \
  verdaccio/verdaccio
```
这边是直接用挂载宿主机 `/etc/verdaccio` 目录的方式来对 verdaccio 的持久化数据进行维护, `-d` 表示后台运行
> 不需要先拉取镜像，启动的时候，如果本地找不到镜像，会自己的 pull 线上的镜像

```text
[root@VM-1-3-centos conf]# V_PATH=/etc/verdaccio; docker run -d -it  --name verdaccio   -p 4873:4873   -v $V_PATH/conf:/verdaccio/conf   -v $V_PATH/storage:/verdaccio/storage   -v $V_PATH/plugins:/verdaccio/plugins   verdaccio/verdaccio
ddaaa3532dd313cab24302011fe439c36771fbbb794f6869af650d9a13404e4e
[root@VM-1-3-centos conf]# docker ps
CONTAINER ID   IMAGE                 COMMAND                  CREATED         STATUS         PORTS                                       NAMES
ddaaa3532dd3   verdaccio/verdaccio   "uid_entrypoint /bin…"   5 seconds ago   Up 5 seconds   0.0.0.0:4873->4873/tcp, :::4873->4873/tcp   verdaccio
```
不过这边要注意一个细节，因为是走挂载的方式，所以启动 docker 时候的配置文件 `config.yaml`，我们要先创建好，直接拷贝官网 github 上的默认文件即可: [config.yaml](https://github.com/verdaccio/verdaccio/blob/5.x/conf/docker.yaml)
> 记得重命名为 `config.yaml`, 完整路径是: `/etc/verdaccio/conf/config.yaml`

如果刚开始没有这个配置文件的话，启动 docker 的时候就会报这个错误:
```text
cannot open config file /verdaccio/conf/config.yaml: false
```

当 docker 启动之后，我们可以通过`docker logs -f verdaccio` 查看当前这个容器的当前 log 输出。

同时也因为启动成功，所以我们也可以查看 webui 界面，端口就是上面的 `4873`

![1](1.png)

## 配置文件分析
现在服务已经启动了，用的配置文件是默认的配置，我们来看下都有哪些配置，下面只列出 `config.yaml` 文件中有启用的配置:
```text
# 存放包的目录
storage: /verdaccio/storage/data
# 存放插件的目录
plugins: /verdaccio/plugins

# https://verdaccio.org/docs/webui
# webui 界面的配置
web:
  title: Verdaccio

# https://verdaccio.org/docs/configuration#authentication
# 权限校验，通过 adduser 创建的用户信息，会放在这个文件
auth:
  htpasswd:
    file: /verdaccio/storage/htpasswd

# https://verdaccio.org/docs/configuration#uplinks
# 其他代理的资源库，在设置 packages 权限的时候，可以通过设置 proxy 来代理下载外部资源包，默认是走 npmjs 官网
# 当然也可以添加多个，比如也可以设置淘宝源，然后可以自己指定不同 scope package 包的时候，如果本私有库找不到，可以通过走不同的 uplink 资源库来进行代理下载
uplinks:
  npmjs:
    url: https://registry.npmjs.org/


# https://verdaccio.org/docs/protect-your-dependencies/
# https://verdaccio.org/docs/configuration#packages
# 包权限管理，可以根据不同的 scope 来设置不同权限，默认就是访问下载是所有人，但是 publish 和 unpublish 是要有用户登录权限才行
# 然后也可以通过配置 proxy 代理源，在当找不到包的时候，可以去外网下载包
packages:
  '@*/*':
    access: $all
    publish: $authenticated
    unpublish: $authenticated
    proxy: npmjs

  '**':
    access: $all
    publish: $authenticated
    unpublish: $authenticated
    proxy: npmjs


# https://verdaccio.org/docs/configuration#server
# 用来修改服务器属性
server:
  keepAliveTimeout: 60

# https://verdaccio.org/docs/configuration/#audit
# 校验中间件
middlewares:
  audit:
    enabled: true

# https://verdaccio.org/docs/logger
# log 输出设置，也可以设置输出到 file 
log: { type: stdout, format: pretty, level: http }

```
默认配置其实不多，并且官方文档其实说的很清楚了，这边不再赘述， 正常情况下，在有挂载 `conf` 和 `storage` 和 `plugin` 目录的情况下，配置项保持默认其实就够了，后面如果需要调整的话，下面会单独说明

## 实际操作
接下来我们来实际操作整个过程
> 本次测试的客户端的 node 版本是 `16.20.0` 版本

### 1. 创建账号
要发布的话，肯定是要有一个账号的。默认情况下，肯定是没有账号的 (使用 `npm whoami` 查看当前用户，要指定源，不然就会指向默认源):
```text
>npm whoami --registry http://43.139.27.159:4873/
npm ERR! code ENEEDAUTH
npm ERR! need auth This command requires you to be logged in.
npm ERR! need auth You need to authorize this machine using `npm adduser`
```
通过 `adduser` 创建一个
```text
>npm adduser --registry http://43.139.27.159:4873/
npm WARN adduser `adduser` will be split into `login` and `register` in a future version. `adduser` will become an alias of `register`. `login` (currently an alias) will become its own command.
npm notice Log in on http://43.139.27.159:4873/
Username: zachke
Password:
Email: (this IS public) kebingzao@gmail.com
Logged in as zachke on http://43.139.27.159:4873/.
```

不过这边要注意一个细节，就是如果是用 docker 安装的话，因为用户新增的时候，会写入 htpasswd 文件，由于该文件是在宿主机上，也就是 `/etc/verdaccio/storage/htpasswd` 这个文件，而 verdaccio 容器中拥有自己的用户名，名字就叫 verdaccio，所以无法写入 root 用户拥有的文件。

所以这时候在创建的时候，就会报:
```text
npm ERR! 500 Internal Server Error - PUT http://43.139.27.159:4873/-/user/org.couchdb.user:zachke - internal server error
```

解决的方式也很简单，只要将宿主机的这个 storage 目录的 owner 设置为 verdaccio 用户即可。而且 docker 容器中的 uid 和 gid 和宿主机是共享的，只不过没有具体的名称。

因此我们通过 `docker inspect verdaccio` 查看 uid, 发现是 `10001`
```text
[root@VM-1-3-centos ~]# docker inspect verdaccio | grep UID
                "VERDACCIO_USER_UID=10001",
```
至于 gid，就需要进入到这个容器中，并且打开用户组文件进行查找, 可以看到是 `65533`
```text
~ $ cat /etc/group | grep verdaccio
nogroup:x:65533:verdaccio
```
所以在宿主机改一下 storage 的权限即可，改成 verdaccio 的 user id，和 group id，也就是 10001 和 65533, `-R` 表示子目录也同样需要
```text
[root@VM-1-3-centos verdaccio]# sudo chown -R 10001:65533 /etc/verdaccio/storage
```
这样子就可以创建成功了:
```text
>npm whoami --registry http://43.139.27.159:4873/
zachke
```
也可以在 `storage/htpasswd` 这个查看这个文件， 就可以发现已经有一个用户了
```text
[root@VM-1-3-centos verdaccio]# cat /etc/verdaccio/storage/htpasswd
zachke:wKTp2Zn.123456:autocreated 2023-09-18T02:25:30.987Z
```
同时通过这个账号也可以登录 web ui 后台

![1](2.png)

既然有添加用户，那肯定有登录和登出:
```text
npm login --registry http://43.139.27.159:4873/
npm logout --registry http://43.139.27.159:4873/
```
> 添加用户就默认登录了,如果要切换同一个源的不同用户，就要先登出，然后再登录

### 2. 创建项目
接下来就可以创建一个新的项目, 使用 `npm init` 指令可以快速生成一个只包含 `package.json` 文件的项目:
```text
$ mkdir testlog
$ cd testlog/
$ npm init
```
基本上一路默认下去，就可以创建一个项目了。
> 如果不想一个一个项的配置，直接使用 `npm init -y` 指令创建，就会生成一份全部走默认值的`package.json` 文件

这时候这个 testlog 项目就会有一个 `package.json` 的文件了
```json
{
  "name": "testlog",
  "version": "1.0.0",
  "description": "is test log",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC"
}
```

单独只有这个文件肯定是不行，因为上面的 `main` 字段已经默认入口文件是 `index.js`, 所以这时候需要再创建一个 `index.js` 文件来当作入口文件， 内容如下:
```text
function mylog() {
    console.log('mylog: ', ...arguments);
}

module.exports = {mylog};
```
功能很简单，就是针对 `console.log` 做了一层封装，加了一个 log 输出前缀

### 3. 发布项目
接下来我们就发布这个项目，版本就是上面的 `version: 1.0.0`:
```text
\testlog> npm publish --registry http://43.139.27.159:4873/
npm notice
npm notice 📦  testlog@1.0.0
npm notice === Tarball Contents ===
npm notice 93B  index.js
npm notice 214B package.json
npm notice === Tarball Details ===
npm notice name:          testlog
npm notice version:       1.0.0
npm notice filename:      testlog-1.0.0.tgz
npm notice package size:  332 B
npm notice unpacked size: 307 B
npm notice shasum:        8e13c7124703cf5a6fe8d73dce54ef541f8edcb4
npm notice integrity:     sha512-vUYCEp6/Lw7cj[...]vYziTPKdNFRBg==
npm notice total files:   2
npm notice
npm notice Publishing to http://43.139.27.159:4873/
+ testlog@1.0.0
```
这时候就可以再 web 后台看到了:

![1](3.png)

可以看到首页列表中，已经有一个包了，并且他的标题就是包名(package.json 文件的 name 字段), 描述就是 package.json 文件的 description 字段， 版本就是 `1.0.0`, 因为 author 字段是空，所以作者变成 `Unknown`.
> 如果有 README.md 文件的话，描述会优先获取 README.md 文件中主标题下的第一行，如果没有 README.md 文件的话，描述才会获取 package.json 文件的 description 字段

点进去会发现没有详情，也就是没有 README.md， 右边的详情也没有 author

![1](4.png)

所以接下来我们再迭代一个版本，因为是 patch 版本，所以新的版本号是 `1.0.1`， 然后填写了 author 这一栏，增加了 README 文件。
```json
{
  "name": "testlog",
  "version": "1.0.1",
  "description": "is test log",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "zach",
  "license": "ISC"
}
```
再 publish 一次，可以看到版本号变成 `1.0.1` 了， author 信息有了， readme 内容也有了

![1](5.png)

每次发布的时候， version 字段都要往上加，不能往下，可以走最后一位修订号(patch)，也可以是第二位的次版本号(minor)，或者第一位的主版本号(major)，反正就是不能往下，不然会报错。

#### 注意事项
同时要注意，正常情况下，在 publish 的时候，npm 默认也会排除[一些文件](https://docs.npmjs.com/cli/v9/using-npm/developers#keeping-files-out-of-your-package)，不会将他们发布到包中。 比如过滤掉这些:
```text
.*.swp
._*
.DS_Store
.git
.gitignore
.hg
.npmignore
.npmrc
.lock-wscript
.svn
.wafpickle-*
config.gypi
CVS
npm-debug.log
```
其他的剩余文件都会发布上去，如果需要排除其他文件，可以在`.npmignore`中配置。

同时如果你的包是需要构建的，只需要发布构建后的文件，`src` 目录不需要，那么就要在 `package.json` 指定 `files` 字段，将构建后的目录或者文件包含在 `files` 数组里面， 这样子 npm 在 publish 的时候，就会默认只发布 `files` 数组里面的文件。
> `package.json`, `README.md`, `CHANGELOG.md`, `LICENCE` 这几个文件不需要特意指定到 files 数组，也会自动包含, 除非手动强制在 `.npmignore` 中配置不要


### 4. 撤销已发布的包
撤销已发布的包在 Verdaccio 中是通过使用 npm unpublish 命令来完成的，格式为:
```text
npm unpublish [@scope/]pkg[@version] --registry http://43.139.27.159:4873
```
- `@scope/` 是可选的，仅在你撤销的包在特定作用域下时需要。
- `pkg` 是你要撤销的包的名称。
- `@version` 是你要撤销的包的版本。如果你想撤销所有版本，可以省略这部分。

对于本例来说，由于在初始化的时候，并没有针对这个包设定作用域，因此不需要填。想撤销刚才发布的 `1.0.1` 版本, 就可以:
```text
\testlog> npm unpublish testlog@1.0.1 --registry http://43.139.27.159:4873/
- testlog@1.0.1
```
这时候 web 后台就只剩下 1.0.0 的版本了。 

如果想整个包都撤掉，也就是删掉，不带版本号就行了，不过要加 `-f` 指令，表示强制删除，如果没有的话，会报错
```text
\testlog> npm unpublish testlog -f --registry http://43.139.27.159:4873/
npm WARN using --force Recommended protections disabled.
- testlog
```
这时候整个包都被删掉了，web 后台看不到这个包了

![1](6.png)

### 5. 关于 scope 作用域
首先为啥会有 scope 的需求, 很简单，所有 npm 包都有一个名称。如果你要给自己的包，起一个有「寓意」的名字，会发现绝大多数单词/短语已被占用，即使没有真正的内容，这真是无可奈何的事儿。如何解决这一困境呢？ 你可以发布带作用域/范围（scope）的包，通过 `scope + pkg` 的全名称来让你的包不会重复，并且便于组织识别。
> 事实上 npm 上面的 scope 其实代表的就是 org 组织

所以正常我们在使用私有库 lib 的时候，如果是自己自研的，那么会带上 scope, 这个是因为我们的私有库的 lib，一般包含 3 种类型的 lib:
1. 团队内部自研的，这时候 scope 就会是我们自己定的组织名称, 比如组织名称叫 air， 那么内部自研的库，都应该是以 air 这个 scope 命名。当然如果你内部自研的库很多，还可以再细分，比如如果是 js 功能的库，那么 scope 就可以叫 `air-js`, 如果是组件库的，那么 scope 就可以叫 `air-component`
2. 基于线上开源项目再进行二开自己维护的，这时候 scope 就是开源项目的 scope
3. 之前线上有开源，有地址，但是后面不再维护，或者是很旧的找不到线上包的库，这时候我们就会 fork 一份到私有库，但是不进行二开需求，这时候 scope 就是 fork

因为刚才的测试项目被 `unpublish` 删掉了, 所以我们重新再创建一个具有 scope 的项目，通过 `npm init --scope xxx` 来指定项目初始化的 scope
```text
$ mkdir testlog2
$ cd testlog2/
$ npm init --scope zachke -y
Wrote to \testlog2\package.json:
{
  "name": "@zachke/testlog2",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC"
}
```
这时候我就建了一个 scope 为 `zachke` 的包了，包名为 `testlog2`

### 6. 安装私有包
那么怎么在项目中安装呢? 有几种方式，首先我们先创建一个项目，就一个简单的 node 项目，也不引用什么 vue 框架了:
```text
$ mkdir my-project
$ cd my-project/
$ npm init -y
```

#### 1. 命令行直接指定 registry 安装
接下来直接安装刚才的那个包:
```text
\my-project> npm install @zachke/testlog2 --registry http://43.139.27.159:4873/  
```
这时候就可以再 `package.json` 看到这个包的依赖了

![1](7.png)

接下来直接引用这个包就可以了。 补上项目的 `index.js` 入口文件:
```text
const { mylog } = require('@zachke/testlog2');

mylog("hello!!")
```
并且补上 script 运行指令 `node index.js`:
```text
{
  "name": "my-project",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "@zachke/testlog2": "^1.0.0"
  }
}
```
接下来就可以使用 npm 运行了:
```text
\my-project> npm run start

> my-project@1.0.0 start
> node index.js

mylog:  hello!!
```
可以看到这个项目已经运行起来了，并且我们的私有包有被成功引用。 不过这种情况的安装有个问题，他是依赖 lock 文件，如果是 npm 安装，那么就是依赖 `package-lock.json` 这个文件，也就是本地中的

![1](8.png)

如果是 yarn 安装的，那么就是依赖于 yarn.lock 文件。

这种方式的安装缺点就是一旦这个 lock 文件被删掉，那么就不会从私有源安装。因为外部公共 npmjs 没有这个私有包，就会报错。

#### 2. 私有库设置为全局的源
第二种安装方式，就是将你的私有库的源设置为全局的源
```markdown
npm config set registry http://43.139.27.159:4873

yarn config set registry http://43.139.27.159:4873
```

这样就可以使你的项目中的所有包都从私有源安装，Verdaccio 会自动代理 npmjs.org上的包 (可以指定 uplink 的其他代理)，并且会缓存这些包以加快安装速度，但是可靠性未验证，暂时不推荐。

而且为了安全，其实我是推荐在私有库的配置中，将上游的 npm 的代理配置关闭的，也就是非私有库的包，不应该通过私有库地址去安装, 也就是将 packages 中的各个 scoped 的配置下面的 `proxy` 配置删掉或者注释掉
```text
packages:
  '@*/*':
    # scoped packages
    access: $all
    publish: $authenticated
    unpublish: $authenticated
    # 既然是私有库了，找不到就需要去公网找，这样子更安全
    # proxy: npmjs
```
一旦关闭了 proxy 功能，这时候安装包就更不能走将私有库设置为全局源的方式了，不然就会出现该项目安装其他非私有库的时候，就会找不到报错。

#### 3. 项目配置中指定特定的registry为特定的scope提供包 (推荐)
本例就是在 my-project 项目下，创建一个 npm 包配置文件 `.npmrc` ，比如本项目就是 `.npmrc`, 然后在这个文件中指定特定包的特定源，比如本例就是
> 如果是 yarn 安装的，那么就是 `.yarnrc` 文件

![1](9.png)

这边配 scope 就可以了，它会将所有这个 scope 中的包，全部换成后面指定的地址来下载，也就是我们的私有库地址。
```text
@zachke:registry=http://43.139.27.159:4873/
```
scope 如果不写的话，比如这样子
```text
registry=http://43.139.27.159:4873/
```
那其实就是全局配置了，是只针对这个项目的全局配置，本质上跟第二种全局改 registry 的效果一样。也不推荐，所以还是要有 scope

测试结果如下，先把原先的包删掉，然后再安装
```text
\my-project> npm uninstall @zachke/testlog2 --registry http://43.139.27.159:4873/
\my-project> npm install @zachke/testlog2
```
而这个也是我们项目中最常用的安装私有库包的方式。

## 发布操作加钉钉通知
接下来我们给发布行为的操作，加上钉钉通知，Verdaccio 是支持 hook 触发的，具体看:
- [verdaccio send Notifications](https://verdaccio.org/docs/notifications/)
- [自定义机器人接入](https://open.dingtalk.com/document/robots/custom-robot-access)

然后 config.yaml 文件中的最后配上，要记得重启 docker 容器生效 (`docker restart verdaccio`)
```text
# notify
notify:
  'dingding':
    method: POST
    headers: [{ 'Content-Type': 'application/json' }]
    endpoint: https://oapi.dingtalk.com/robot/send?access_token=0427285b76cebabe510d14c294dba6738dcb5991515368e7293f16f9f0ba47a6
    content: '{"msgtype":"markdown","markdown":{"title":"【{{ name }}】 版本更新","text":"> 新的包版本更新!!!<br> **包名**: {{ name }} <br> **版本**: {{publishedPackage}} <br> **发布者**: {{publisher.name}} <br> **查看详情**: [查看链接](http://43.139.27.159:4873/-/web/detail/{{ name }})","at":{"isAtAll":false}}}'
```
然后重新推送一个新的版本，这时候可以看到有 log 了 (可以通过 `docker logs -f verdaccio` 查看 log):
```text
info --- auth/allow_action: access granted to: zachke
info --- zachke is allowed publish for @zachke/testlog2
http <-- 200, user: zachke(125.77.202.250), req: 'PUT /@zachke%2ftestlog2', bytes: 1496/0
info --- A notification has been shipped: {"msgtype":"markdown","markdown":{"title":"【@zachke/testlog2】 版本更新","text":"> 新的包版本更新!!!<br> **包名**: @zachke/testlog2 <br> **版本**: @zachke/testlog2@1.0.5 <br> **发布者**: zachke <br> **查看详情**: [查看链接](http://43.139.27.159:4873/-/web/detail/@zachke/testlog2)","at":{"isAtAll":false}}}
```
效果就是钉钉可以收到机器人通知了:

![1](10.png)

## 最佳实践
### 1. 修改 web ui
正常情况，我们不太需要修改 web 界面 ui，但是为了更好的定制化，一般我们只需要改几个值，比如 `title`， `logo`， `primary_color` 这三个值就够了

比如我在配置文件 config.yaml 改成
```text
web:
  title: zachNpm
  logo: https://kebingzao.com/images/avatar.png
  primary_color: '#67b7bf'
```
重启一下 docker 容器，就可以看到生效了

![1](11.png)

> 如果还想再彻底一点的，`favicon` 还是可以调整的，甚至加载的 js 都可以换，但是一般没啥必要

### 2. 不启用 package 的代理功能
上文其实有说过，其实 Verdaccio 是支持非常灵活的代理下载功能的，比如我可以在 unlinks 指定多个代理源:
```text
uplinks:
  npmjs:
    url: https://registry.npmjs.org/
  server2:
    url: http://mirror.local.net/
    timeout: 100ms
  server3:
    url: http://mirror2.local.net:9000/
  baduplink:
    url: http://localhost:55666/
```
然后在 package 配置那边针对不同的 scope 来指定不同的 proxy, 甚至同一个 scope 可以指定多个下载源
```text
packages:
    'my-company-*':
        access: $all
        publish: $authenticated
        unpublish: $authenticated
        proxy: server3 npmjs
    '@my-local-scope/*':
        access: $all
        publish: $authenticated
        unpublish: $authenticated
        proxy: server2
    '**':
        access: $all
        publish: $authenticated
        unpublish: $authenticated
        proxy: npmjs
```
但是为了安全，我是不建议在私有库的资源里面再去代理下载公网的包的，所以应该将 proxy 的配置去掉，私有库就只负责下载私有库里面的私有包就行了。

### 3. nginx 代理 https
Verdaccio 是支持 tls 的 https 配置，但是我们一般还是用 nginx 来代理 https 请求，一方面是便于证书的管理，每年换证书的时候，统一换 nginx 的证书就够了，其他服务的证书不需要额外去换。

另一方面 nginx 对证书链和支持和校验也是非常成熟的，不用担心会因为证书链的验证问题，导致某些情况会出现 ssl 的链接错误。

所以应该这样子配置
```text
[root@test-server conf.d]# cat npm.foo.com.conf
server {
    listen 80;
    listen 443 ssl http2;
    server_name npm.foo.com;
    ssl_certificate             ssl/foo.com.crt;
    ssl_certificate_key         ssl/foo.com.key;
    ssl_session_timeout         5m;
    ssl_protocols               TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    
    #兼容低版本的TLSv1.0 1.1 版本加密
    ssl_ciphers EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH:ECDHE-RSA-AES128-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA128:DHE-RSA-AES128-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES128-GCM-SHA128:ECDHE-RSA-AES128-SHA384:ECDHE-RSA-AES128-SHA128:ECDHE-RSA-AES128-SHA:ECDHE-RSA-AES128-SHA:DHE-RSA-AES128-SHA128:DHE-RSA-AES128-SHA128:DHE-RSA-AES128-SHA:DHE-RSA-AES128-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA384:AES128-GCM-SHA128:AES128-SHA128:AES128-SHA128:AES128-SHA:AES128-SHA:DES-CBC3-SHA:HIGH:!aNULL:!eNULL:!EXPORT:!DES:!MD5:!PSK:!RC4;
    ssl_prefer_server_ciphers   on;

    access_log /var/log/nginx/npm.foo.com.access.log;
    error_log /var/log/nginx/npm.foo.com.error.log;
    
    location /{
         add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
         add_header Access-Control-Allow-Headers 'DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization';
         if ($request_method = 'OPTIONS') {
             return 204;
         }
         proxy_set_header    X-Real-IP $remote_addr;
         proxy_set_header    Host $http_host;
         proxy_set_header    X-Forwarded-Proto $scheme;
         proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
         proxy_pass          http://127.0.0.1:4873/;
    }
}
```
同时这边还要注意一个细节，就是要将 nginx post 的限制调大一点，在 nginx.conf 那边调整，
```text
client_max_body_size 10M;
```
> 调整跟 verdaccio 的默认配置值一样即可: `max_body_size: 10mb`

不然就会出现发布的时候包过大，出现报错 413 的情况。

### 4. 遵循版本号语义化
在发布版本的时候，版本号一定要遵循语义化的规则，具体看: [语义化版本 2.0.0](https://semver.org/lang/zh-CN/)

版本格式分为三个格式: 版本格式：`主版本号.次版本号.修订号`(`MAJOR.MINOR.PATCH`)，版本号递增规则如下：
- 主版本号：当你做了不兼容的 API 修改，
- 次版本号：当你做了向下兼容的功能性新增，
- 修订号：当你做了向下兼容的问题修正。

先行版本号及版本编译信息可以加到“主版本号.次版本号.修订号”的后面，作为延伸。

基本上的用法:

|包迭代状态|阶段(Stage)| 版本号更新规则 |范例|用途|
|---|---|---|---|---|
| 第一次发布 | New product | 从 1.0.0 开始 | 1.0.0 | 第一次发布上传包|
| bug 修改并且向下兼容 | Patch release | 增加修订号(patch) | 1.0.1 | bug 小修复 |
| 新功能增加并且向下兼容 | Minor release | 增加次版本号(Minor),并且将修订号重置为 0 | 1.1.0 | 新的功能出现，并且不影响现有旧版本应用 |
| 可能会有无法向下兼容 | Major release| 增加主版本号(Major)并且后面的次版本号和修订号都重置为 0 | 2.0.0 | 会有破环性更新，旧版本再不调整代码的情况下，可能无法应用|

除了可以直接在 `package.json` 直接修改版本号之外，我们可以用 `npm version xxx` 的方式来更新版本号
```text
npm version [<newversion> | major | minor | patch | premajor | preminor | prepatch | prerelease | from-git]
```
比如: 原來是 `1.0.0`，用 `npm version minor` 就会变成 `1.1.0`， 后面使用包的时候，可以使用 `npm upgrade <package>` 更新包版本

然后在安装包的时候，也有一些符号或者表达式来保证，你在更新的时候，不会一下子跨大版本更新，比如
- `~` --> 只允许修订号版本迭代(如果有缺省值，缺省部分任意迭代)，比如 `~1.2.3` 相当于 `>=1.2.3 <1.3.0`, `~1.2` 相当于 `>=1.2.0 < 1.3.0`
- `^` --> 允许次版本号迭代，比如 `^1.2.3` 相当于 `>=1.2.3 <2.0.0`, `^1.x` 相当于`>=1.0.0 <2.0.0`
- `<、<=、>、>=、=` --> 指定版本范围，比如 `=1.2.7 <1.3.0` 中包括`1.2.7`、`1.2.8`、`1.2.99`等等，但不包括`1.2.6`、`1.3.0` 或`1.1.0`等等
- `x、X、*` --> 可以替代`主版本号.次版本号.修订号`三段中任意一段，表示该位置版本号没有限制, `*` 相当于 `>=0.0.0`，表示任何版本号, `1.X`或`1.x` 相当于 `>=1.0.0 <2.0.0`，匹配到主版本号


## 总结
通过使用 verdaccio 搭建私有 npm 仓库，可以让前端的内部私有包得到规范的开发和流通，大大提供了前端开发的效率。

接下来说一下团队协作上的私有包的开发和发布规范: {% post_link npm-publish %}

---
参考资料:
- [What is Verdaccio?](https://verdaccio.org/docs/what-is-verdaccio)
- [NPM Version Management Specification](https://www.cnblogs.com/skylor/p/9675646.html)
- [语义化版本 2.0.0](https://semver.org/lang/zh-CN/)


