---
title: 自建vue组件 air-ui (2) -- 先分析一下 element ui 项目
date: 2019-12-31 15:04:43
tags: js
categories: 
- 前端相关
- 自建vue组件 air-ui
---
## 前言
工欲善其事必先利其器，既然我们的 `air-ui` 大部分都是借鉴 `element-ui` 项目，所以本章我们就来分析一下 `element-ui` 项目源码。

## 下载源码
首先我们到这个 https://github.com/ElemeFE/element ， 把源码下载下来

## 安装 yarn
因为是基于 yarn 安装的，所以我们也全局装一下 yarn：
```text
F:\code\github\element>npm install -g yarn
C:\Program Files\nodejs\yarn -> C:\Program Files\nodejs\node_modules\yarn\bin\yarn.js
C:\Program Files\nodejs\yarnpkg -> C:\Program Files\nodejs\node_modules\yarn\bin\yarn.js
+ yarn@1.19.1
added 1 package in 7.777s
```
安装成功，试一下指令是否正常： 当前版本是 `2.12.0` (我在写这个文章的时候，其实已经更新到 `2.13.0`，但是差别不大，都是一些细节的调整)
```text
F:\code\github\element>yarn --version
1.19.1
```
<!--more-->
## 本地环境跑起来
```text
F:\code\github\element>npm run dev

> element-ui@2.12.0 dev F:\code\github\element
> npm run bootstrap && npm run build:file && cross-env NODE_ENV=development webpack-dev-server --config build/webpack.demo.js & node build/bin/template.js


> element-ui@2.12.0 bootstrap F:\code\github\element
> yarn || npm i

yarn install v1.19.1
[1/4] Resolving packages...
[2/4] Fetching packages...
info fsevents@1.2.7: The platform "win32" is incompatible with this module.
info "fsevents@1.2.7" is an optional dependency and failed compatibility check. Excluding it from installation.
[3/4] Linking dependencies...
warning " > karma-webpack@3.0.5" has incorrect peer dependency "webpack@^2.0.0 || ^3.0.0".
[4/4] Building fresh packages...
Done in 202.16s.
。。。
i ｢wdm｣: Compiled successfully.
```
第一次的时候，会先安装依赖。在本地浏览器打开 `localhost:8085`，可以看到跟官网的站点一样:

![1](1.png)

## 目录结构
首先我们先看一些目录结构：
```
element
  |---build/       构建相关的文件
  |---example/     放置element api的页面文档
  |---lib/         组件构建打包之后的存放目录
  |---packages/    放置element的组件（css样式放置在这个目录下theme-chalk下）
  |     |---button/            button 组件的目录
  |     |     |---src/         button 组件的业务代码
  |     |     |---index.js     button 组件的定义文件
  ... 这里面 N 个相同目录格式的组件 ...
  |     |---theme-chalk/       存放 scss 样式
  |     |     |---lib/         scss 打包之后的css文件
  |     |     |---src/         scss 样式文件
  |     |     |---gulpfiles    gulp 构建任务，将src目录中的 scss 打包成 css并放到 lib 目录
  |---src/
  |     |---directives/     放置自定义指令
  |     |---locale/         放置语言的配置文件
  |     |     |---lang/     放置多语言
  |     |---mixins/         放置组件用的混合文件
  |     |---transitions/    放置动画配置文件
  |     |---utils/          放置用到工具函数文件
  |     |---index.js        组件注册的入口文件
  |---test/        测试文件
  |---types/       typescript 的数据类，用来给 IDE 在写 ts 代码时候的提示
  |---components.json   存放单独导出组件的json文件
  |---package.json
```
可以看到几个有点奇怪的地方：
1. 存放组件的目录，竟然不是在 src 的 components 目录中，而是单独抽出来
2. 存放css的目录，竟然跟组件是同级，而且藏在了一个叫做`theme-chalk`的目录，辨识度太低，不注意看，根本找不到，为啥不放 src 目录的 styles 目录呢?
3. 文档站点所在的 example 的目录结构，也有点乱，至少第一眼看过去，都是看不懂

## 1. dev 环境的构建
```text
npm run dev
```
想要更快的了解一个项目，除了将环境跑起来之后，另一个就是分析构建的方式了，所以接下来我们分析一下 dev 环境的构建方式，首先从指令来看：
```text
"dev": "npm run bootstrap && npm run build:file && cross-env NODE_ENV=development webpack-dev-server --config build/webpack.demo.js & node build/bin/template.js",
```
这个其实是一连串指令结合的，我们一个一个分析：
### step 1.1
```text
"bootstrap": "yarn || npm i"
```
这个就是纯粹的安装依赖，没有实际操作

### step 1.2
```text
"build:file": "node build/bin/iconInit.js & node build/bin/build-entry.js & node build/bin/i18n.js & node build/bin/version.js"
```
这个又是一连串指令的结合，继续拆开分析：
#### stpe 1.2.1
```text
node build/bin/iconInit.js
```
简单的来说，就是把`packages/theme-chalk/src/icon.scss` 这个 css 文件中的所有的 icon 描述字符都提取出来,比如这个：
```text
.el-icon-turn-off:before {
  content: "\e734";
}
```
其实就是把`turn-off`这个子串提取出来，然后放入到这个`examples/icon.json` 文件里面，这个文件其实就是一个大的 json 数组：

![1](2.png)

后面这个文件会在显示字体图标的时候，会拿出来做映射。 这样的好处就是，以后添加新的字体图标的时候，写文档的时候，不用手动去添加，它会自动生成这张映射表。

在启动dev模式的时候，入口的 js 文件`examples/entry.js` 就会把这个 json 文件引入进入：
```text
import icon from './icon.json';

Vue.prototype.$icon = icon; // Icon 列表页用
```
会在 icon 列表的时候，用上这个映射表： 以中文为例，那么文件就在 `examples/docs/zh-CN/icon.md` 这个展示页用到
```html
### 图标集合

<ul class="icon-list">
  <li v-for="name in $icon" :key="name">
    <span>
      <i :class="'el-icon-' + name"></i>
      <span class="icon-name">{{'el-icon-' + name}}</span>
    </span>
  </li>
</ul>
```
展示页就是下图：

![1](3.png)

#### step 1.2.2
```text
node build/bin/build-entry.js
```
生成入口文件 `index.js`, 简单的来说，就是通过这个任务，来生成 `src` 目录下的 `index.js` (每次打包，这个 `index.js` 都会重新生成), 这个也是整个组件库的入口文件。 他这个有一个模板，里面有一些组件是内置会有的，比如：
```html
Vue.prototype.$loading = Loading.service;
Vue.prototype.$msgbox = MessageBox;
Vue.prototype.$alert = MessageBox.alert;
Vue.prototype.$confirm = MessageBox.confirm;
Vue.prototype.$prompt = MessageBox.prompt;
Vue.prototype.$notify = Notification;
Vue.prototype.$message = Message;

export default {
  version: '{{version}}',
  locale: locale.use,
  i18n: locale.i18n,
  install,
  CollapseTransition,
  Loading,
{{list}}
};
```
他是通过获取根目录下的`components.json` 这个文件。这个文件是所有可以配置的组件的列表。一旦这个组件在列表里面，那么我打包的时候，就会把这个组件打进去。反之，如果里面有些组件，比如 `drawer` 和 `divider` 这种，我的项目根本不会用到。那么我就把这两个组件从 `components.json` 中剔除，那么生成的 `src/index.js` 这个 js，就不会包含这两个组件了。 从而实现可定制化组件的方式。
#### step 1.2.3
```text
node build/bin/i18n.js
```
生成这个站点的 i18n 多语言的静态页面, 通过这个任务，我们可以生成你想要支持的多语言文件，这个多语言文件其实就是文档站点的多语言的每一个页面的词条，跟组件的多语言没有关系

![1](4.png)

每一个页面用到的多语言都有，从 `json` 来看，目前就只支持 4 种多语言：

![1](5.png)

具体看生成的站点就知道了：

![1](6.png)

就四种，那么是怎么替换的呢？ 其实很简单，根据不同的语言的不同的页面，然后把对应的词条 json 传进去替换就行了。
```javascript
Object.keys(lang.pages).forEach(page => {
  var templatePath = path.resolve(__dirname, `../../examples/pages/template/${ page }.tpl`);
  var outputPath = path.resolve(__dirname, `../../examples/pages/${ lang.lang }/${ page }.vue`);
  var content = fs.readFileSync(templatePath, 'utf8');
  var pairs = lang.pages[page];

  Object.keys(pairs).forEach(key => {
    content = content.replace(new RegExp(`<%=\\s*${ key }\\s*>`, 'g'), pairs[key]);
  });

  fs.writeFileSync(outputPath, content);
});
```
模板就是 `.tpl`，结果页就是 `.vue` 文件， 举个例子，就以 design 这个页面来说：

![1](7.png)

他其实就是把中文下的`design`页面的词条取出来，然后塞到`design.tpl`这个模板里面：

![1](8.png)

最后生成的结果页就是 `design.vue` 这个页面：

![1](9.png)

这样就把这个站点的多语言相关的页面都替换完了。

#### step 1.2.4
```text
node build/bin/version.js
```
生成对应的版本列表文件 `examples/version.json`, 这个没啥好说的，硬编码写入到一个文件而已。 这个文件会在 `changelog` 这个页面的时候， 动态用 `ajax` 来请求：
```javascript
xhr.open('GET', '/versions.json');
```
然后再结合根目录下的 `CHANGELOG.{lang}.md` 来显示出真正的 log 列表, 这个 md 文件会在 `changlog.vue` 文件中被引用进去：
```javascript
import ChangeLog from '../../../CHANGELOG.zh-CN.md';
```
### step 1.3
```text
cross-env NODE_ENV=development webpack-dev-server --config build/webpack.demo.js
```
到这一步才开始进行`webpack`构建了，对应的`webpack`的配置文件是`webpack.demo.js`。 `cross-env` 是为了兼容各个OS系统的环境设置的方式。

因为我们的 `env` 是 dev，并且是非 `play` 模式， 所以 js 执行的入口文件是  `example/entry.js` 这个文件。 这个也是这个项目的 入口 vue 文件 （就是初始化 vue 的那个文件，而入口文件是 `index.tpl`）,然后经过 `webpack` 打包之后，这个文件就会变成 `[name].js`，然后被嵌进去到 `index.tpl` 这个模板文件里面：

![1](10.png)

从源码上可以看到，他生成了入口的 js 文件之后，就嵌进去了 `index.tpl` 这个文件里面。而`index.tpl` 这个模板文件，也会在打包的时候，进行 `webpack` 的参数填充，从而变成 真正的入口文件 `index.html`：
```javascript
new HtmlWebpackPlugin({
  template: './examples/index.tpl',
  filename: './index.html',
  favicon: './examples/favicon.ico'
}),
```
至于为什么这个名字 `[name]` 是 `main`，这个确实有点奇怪，按照道理说，我的入口文件是 `entry.js` ，如果`output` 没有特别指定的话， 那么`output` 的 `filename` 中的 `name`，也应该是 `entry` 才对，怎么会变成 `main` ???

后面查了一下，原来`entry` 配置是：
```javascript
entry: './examples/entry.js'
```
那么在 `webpack` 的解析中，就会变成：
```javascript
entry: {
　　main: './examples/entry.js'
}
```
所以名字就会变成 `main` 了, 同时因为 mode 是 dev，所以还会开启本地服务：
```javascript
devServer: {
  host: '0.0.0.0',
  port: 8085,
  publicPath: '/',
  hot: true
},
```
这样子 webpack 打包的部分就完成了。 但是还有最后一步呢。
### step 1.4
```javascript
node build/bin/template.js
```
其实这一步就是`watch`，针对所有的页面模板，进行监听,其实就是针对 `examples/pages/template` 这个目录下的所有的模板，进行监听，如果有修改的话，就重新执行 `npm run i18n` 任务，也就是 
```javascript
"i18n": "node build/bin/i18n.js",
```
其实就是 `1.2.3` 的任务，就是为模板插入多语言， 注意，这时候的 `__dirname` 就是 webpack 运行环境的内存中的目录了,一旦 tpl 文件变了，就会导致对应的 vue 文件改变，从而导致 `webpack` 重新 reload，然后界面就刷新了。因为用的是 `webpack-dev-server`  这个 server， 他有自带监听机制。`rules` 里面的 loader 相关的文件一旦修改的话，是会热更新的：
```javascript
{
  test: /\.vue$/,
  loader: 'vue-loader',
  options: {
    compilerOptions: {
      preserveWhitespace: false
    }
  }
},
```
### dev 打包总结
最后总结一下：通过打包 dev 模式，我们可以知道，根目录是`examples/index.tpl` ， 入口 js 文件是 `examples/entry.js` （这个其实就是vue 的入口文件）, 然后通过 `webpack-dev-server` 启动一个热更新服务 （打包的文件都是放在内存里面的）。事实上，以上这些都没有涉及到 `element-ui` 是怎么编写 ui 组件库的。只是将其 api 站点运行起来而已。

## 2.文档站点的 build 版本
```text
npm run deploy:build
```
之前是文档站点的本地环境版本，也就是运行在内存的，接下来这个是线上部署版本，就是打包成静态文件：
```text
F:\code\github\element>npm run deploy:build

> element-ui@2.12.0 deploy:build F:\code\github\element
> npm run build:file && cross-env NODE_ENV=production webpack --config build/webpack.demo.js && echo element.eleme.io>>examples/element-ui/CNAME
...
```
跟开发模式有点像， 只不过这时候就输出到 `examples` 目录下了，而不是输出到内存了。 整个静态文件都输出到 `examples/element-ui/` 这个目录里面，相当于是一个完整的站点。如果放到 `webserver` 下面。就是一个静态站点。 这个就不分析，因为原理跟 dev 差不多。

## 3.打包组件库
```text
npm run dist
```
具体log：
```text
F:\code\github\element>npm run dist

> element-ui@2.12.0 dist F:\code\github\element
> npm run clean && npm run build:file && npm run lint && webpack --config build/webpack.conf.js && webpack --config build/webpack.common.js && webpack --config build/webpack.component.js && npm run build:utils && npm run build:um
d && npm run build:theme
...
```
目标目录就是 `lib` 目录，原先的项目是没有 `lib` 目录的， 打完包，才会有 `lib` 目录。

![1](11.png)

总的指令是：
```text
"dist": "npm run clean && npm run build:file && npm run lint && webpack --config build/webpack.conf.js && webpack --config build/webpack.common.js && webpack --config build/webpack.component.js && npm run build:utils && npm run build:umd && npm run build:theme",
```
里面有些指令前面已经分析过了，这边就不细讲了：
### step 3.1
```text
"clean": "rimraf lib && rimraf packages/*/lib && rimraf test/**/coverage",
```
其实就是先清理目录：

- 清理 `lib` 目录
- 清理 `packages` 下的 `lib` 目录
- 清理 `test` 目录下的测试目录

### step 3.2
```text
npm run build:file
```
同上面 dev 构建的 `step 1.2` 一样，不在重复讲，无非就是 `初始化 icon`， `生成入口文件`， `生成 多语言静态页`， `生成版本列表文件`

### step 3.3
```text
"lint": "eslint src/**/* test/**/* packages/**/* build/**/* --quiet"
```
这个就是跑代码检测工具

### step 3.4
```text
webpack --config build/webpack.conf.js
```
用 `webpack` 打包，他是把整个 webpack 的打包任务分成了好几块，这个是第一块，这一块的处理其实很简单, 就是将 `src/index.js` 这个入口文件进行打包：
```javascript
mode: 'production',
entry: {
  app: ['./src/index.js']
},
output: {
  path: path.resolve(process.cwd(), './lib'),
  publicPath: '/dist/',
  filename: 'index.js',
  chunkFilename: '[id].js',
  libraryTarget: 'umd',
  libraryExport: 'default',
  library: 'ELEMENT',
  umdNamedDefine: true,
  globalObject: 'typeof self !== \'undefined\' ? self : this'
},
```
这时候我的 lib 目录就生成了，采用 `umd` 模块化加载的方式来打包入口文件 `index.js` 文件。

### step 3.5
```javascript
webpack --config build/webpack.common.js
```
```javascript
mode: 'production',
entry: {
  app: ['./src/index.js']
},
output: {
  path: path.resolve(process.cwd(), './lib'),
  publicPath: '/dist/',
  filename: 'element-ui.common.js',
  chunkFilename: '[id].js',
  libraryExport: 'default',
  library: 'ELEMENT',
  libraryTarget: 'commonjs2'
},
```
这个是另外一种模块化方式的打包， 这个也是将入口文件`src/index.js` 打包成 `element-ui.common.js` 这个文件。

![1](12.png)

这时候应该会有一个疑问？？？ 这两个任务打出来的文件，明显大小不一样？？ 那为啥同一个入口文件，可以通过不同的配置，打出两个不一样的 出口文件？？

后面发现原来是有差别的，这两个其实都是入口文件，只不过 `element-ui.common.js` 这个是 `ES6 Module` 模块化加载方式的入口文件。
而 `index.js` 这个是 `umd` 通用模块化加载方式的入口文件。 他的 `webpack` 的 `output` 配置有一个这个配置：
```javascript
libraryTarget: 'umd',
```
标明要打包成 `umd` 的模块化方式的入口文件。这是一种可以将你的 `library` 能够在所有的模块定义下都可运行的方式。它将在 `CommonJS`, `AMD` 环境下运行，或将模块导出到 `global` 下的变量,也可以用`script`方式引入，像 `jquery`, `lodash`，`underscore`这些工具库, 都是这种打包方式。

打包出来其实是这样子：以jquery 为例：
```javascript
(function (root, factory) {
    if (typeof define === 'function' && define.amd) {
        // AMD
        define(['jquery'], factory);
    } else if (typeof exports === 'object') {
        // Node, CommonJS之类的
        module.exports = factory(require('jquery'));
    } else {
        // 浏览器全局变量(root 即 window)
        root.returnExports = factory(root.jQuery);
    }
}(this, function ($) {
    //    方法
    function myFunc(){};
 
    //    暴露公共方法
    return myFunc;
}));
```
其实就是判断环境，以兼容各种模块加载方式。 所以事实上，打完包的这个 `index.js` 开头就是这个：
```javascript
!function (e, t) {
  "object" == typeof exports && "object" == typeof module ? module.exports = t(require("vue")) : "function" == typeof define && define.amd ? define("ELEMENT", ["vue"], t) : "object" == typeof exports ? exports.ELEMENT = t(require("vue")) : e.ELEMENT = t(e.Vue)
}("undefined" != typeof self ? self : this, function (e) {
  return function (e) {
    var t = {};
```
而本步骤的这种方式，其实就是打包成类似于 `CommonJs` 的方式，只需要将所有的依赖文件都打包成一个文件即可。 而上面的 `libraryTarget: commonjs2` 其实跟 `libraryTarget: commonjs` 差不多，只不过 `commonjs2` 导出的是 `module.exports.default`, 按照我的理解，他其实就是 `es6` 提出的模块加载机制 `ES6 Module`。

事实上，在我自己的理解中，关于 `webpack` 的 `libraryTarget: commonjs` 其实对应就是 `CommonJS` 的模块化加载方式，而 `libraryTarget: commonjs2` 其实对应的是 es6 的 `ES6 Module` 的模块化加载方式，他俩的不同之处在于：
1. `CommonJs` 导出的是变量的一份拷贝，`ES6 Module` 导出的是变量的绑定（export default 是特殊的）
2. `CommonJs`是单个值导出，`ES6 Module`可以导出多个, 一般不提倡 `export default` 和 `export {a, b, c}` 混用。(**事实上，我后面这点踩坑了**)
3. `CommonJs`是动态语法可以写在判断里，`ES6 Module`静态语法只能写在顶层, 而且 `ES6 Module` 是只读的，不能被改变
4. `ES6 Module` 默认就是严格模式

而之所以要用 `ES6 Module` 的模块化加载方式，主要是因为这边涉及到一个模块化的分类，比如：
1. 浏览器端的模块化有两个代表：
  1. `AMD(Asynchronous Module Definition,异步模块定义)`，代表产品为：`Require.js`
  2. `CMD(Common Module Definition,通用模块定义)`，代表产品为：`Sea.js`
2. 服务器端的模块化规范是使用`CommonJS`规范，具体规范是：
  1. 使用`require`引入其他模块或者包
  2. 使用`exports`或者`module.exports`导出模块成员
  3. 一个文件就是一个模块，都拥有独立的作用域
3. 大一统的模块化规范 – `ES6模块化`。`ES6` 语法规范中，在语言层面上定义了 ES6 模块化规范，是浏览器端与服务器端通用的模块化开发规范。规范如下：
  1. 每一个js文件都是独立的模块
  2. 导入模块成员使用`import`关键字
  3. 暴露模块成员使用`export`关键字
  
总结就是：推荐使用`ES6模块化`，因为`AMD`，`CMD`局限使用于浏览器端，而`CommonJS`在服务器端使用。`ES6模块化`是浏览器端和服务器端通用的规范。事实上，后面在做 `air-ui`的时候，我直接抛弃 `umd` 的通用模块化加载方式，直接采用 `ES6 Module` 的方式来引用, 就是 `import` 的方式来引入（当然如果后面业务的扩展真的需要有类似于 `umd` 的引用方式，再把这种打包方式加上去，但是如果只是用来做 vue 项目的话，基本上都是用`ES6 Module` 的方式来引用）。

### step 3.6
```javascript
webpack --config build/webpack.component.js
```
接下来将单独的组件也打包出来
```javascript

const Components = require('../components.json');

mode: 'production',
entry: Components,
output: {
  path: path.resolve(process.cwd(), './lib'),
  publicPath: '/dist/',
  filename: '[name].js',
  chunkFilename: '[id].js',
  libraryTarget: 'commonjs2'
},
```

![1](13.png)

生成的这些文件其实就是单独组件的`es6 Module` 的模块化加载方式，也就是说，如果我们要单独用 `element-ui` 的哪一个组件和样式的话，那么是可以单独引用。不需要引用那个 `element-ui.common.js` 文件，那个太大了。

### step 3.7
```javascript
"build:utils": "cross-env BABEL_ENV=utils babel src --out-dir lib --ignore src/index.js",
```
这个其实就是将 `src` 目录下的 除了 `src/index.js` 这个文件之外的其他所有的文件都先经过 `babel` 转换，然后输出到 `lib` 目录下：

![1](14.png)

因为这几个目录在打包的时候，是通过 `externals` 的方式打包的，并没有打进去 `element-ui.common.js` 中里面去，所以这边要另外处理。所以才用这种方式 , 其实就是这 5 个目录。

### step 3.8
```javascript
"build:umd": "node build/bin/build-locale.js",
```
针对 `umd` 的加载方式，对词条的多语言进行打包。其实就是把 `src/locale/lang` 这个目录下的文件拷贝到 `umd` 目录下，不过为了兼容 `umd` 的加载方式，就做了一些兼容。

### step 3.9
```javascript
"build:theme": "node build/bin/gen-cssfile && gulp build --gulpfile packages/theme-chalk/gulpfile.js && cp-cli packages/theme-chalk/lib lib/theme-chalk"
```
这个其实就是将 `scss` 文件编译成 `css` 文件 并且跟字体文件一起拷贝到 `lib` 目录下的 `theme-chalk` 目录下。

这样子，整个库的 打包就完成了。 

## 4.打包chrome插件和发布chrome插件
```javascript
"deploy:extension": "cross-env NODE_ENV=production webpack --config build/webpack.extension.js",
"dev:extension": "rimraf examples/extension/dist && cross-env NODE_ENV=development webpack --watch --config build/webpack.extension.js",
```
这个是关于组件对应的 chrome 插件。用来配合做主题自定义的。在本文分析中不重要。

## 5.发布到线上
```javascript
"pub": "npm run bootstrap && sh build/git-release.sh && sh build/release.sh && node build/bin/gen-indices.js && sh build/deploy-faas.sh",
```
这个就通过几个脚本：
1. `sh build/git-release.sh`  这个脚本主要是为了检查 git 状态，如果状态是异常的，那么就继续，否则就退出
2. `sh build/release.sh` 这个其实就是将 dev 分支 合并到 master 分支,然后将这个合并的 commit 提交上去，最后提交到 master 分支。
3. `node build/bin/gen-indices.js`  这个就是替换多语言的文本，主要是替换索引
4. `sh build/deploy-faas.sh` 更新文档站点

这个在本文分析中，其实也不重要。

## 使用一下
接下来我们试着装一下这个第三方库：
```javascript
F:\code\vue-airdroid-comps>npm install element-ui
```

![1](15.png)

这个包下载下来是这样子的， 跟下面完整项目比的话， 大部分的资源还是在的

![1](16.png)

但是也可以看出，这个用来被安装的分支，其实不是 master 分支的代码，而是另外一个分支的代码。

具体在项目中引用是这样的：

```javascript
import Vue from 'vue'
import Element from 'element-ui'

Vue.use(Element)

// orimport {
  Select,
  Button
  // ...
} from 'element-ui'

Vue.component(Select.name, Select)
Vue.component(Button.name, Button)
```
这时候代码引用的 `element-ui` 其实就是 `package.json` 中的:
```javascript
"main": "lib/element-ui.common.js",
"name": "element-ui",
```
这两个配置，也就是实际引用的是 `element-ui.common.js` 这个 js，我们看下这个 js 打包后是什么样子的，应该是可以导出各种组件的, 看了一下，他的入口还是原来的 `src/index.js` 这个文件：
```javascript
  version: '2.12.0',
  locale: lib_locale_default.a.use,
  i18n: lib_locale_default.a.i18n,
  install: src_install,
  CollapseTransition: collapse_transition_default.a,
  Loading: packages_loading,
  Pagination: packages_pagination,
  Dialog: dialog,
  Autocomplete: packages_autocomplete,
  Dropdown: packages_dropdown,
  DropdownMenu: packages_dropdown_menu,
  DropdownItem: packages_dropdown_item,
  Menu: packages_menu,
  Submenu: packages_submenu,
  MenuItem: packages_menu_item,
  MenuItemGroup: packages_menu_item_group,
  Input: packages_input,
  InputNumber: packages_input_number,
  Radio: packages_radio,
  RadioGroup: packages_radio_group,
  RadioButton: packages_radio_button,
  Checkbox: packages_checkbox,
  CheckboxButton: packages_checkbox_button,
  CheckboxGroup: packages_checkbox_group,
  Switch: packages_switch,
  Select: packages_select,
  Option: packages_option,
  OptionGroup: packages_option_group,
  Button: packages_button,
  ButtonGroup: packages_button_group,
  Table: packages_table,
  TableColumn: packages_table_column,
  DatePicker: packages_date_picker,
  TimeSelect: packages_time_select,
  TimePicker: packages_time_picker,
  Popover: popover,
  Tooltip: packages_tooltip,
  MessageBox: message_box,
  Breadcrumb: packages_breadcrumb,
  BreadcrumbItem: packages_breadcrumb_item,
  Form: packages_form,
  FormItem: packages_form_item,
  Tabs: packages_tabs,
  TabPane: packages_tab_pane,
  Tag: packages_tag,
  Tree: packages_tree,
  Alert: packages_alert,
  Notification: notification,
  Slider: slider,
  Icon: packages_icon,
  Row: packages_row,
  Col: packages_col,
  Upload: packages_upload,
  Progress: packages_progress,
  Spinner: packages_spinner,
  Message: packages_message,
  Badge: badge,
  Card: card,
  Rate: rate,
  Steps: packages_steps,
  Step: packages_step,
  Carousel: carousel,
  Scrollbar: scrollbar,
  CarouselItem: carousel_item,
  Collapse: packages_collapse,
  CollapseItem: packages_collapse_item,
  Cascader: packages_cascader,
  ColorPicker: color_picker,
  Transfer: transfer,
  Container: packages_container,
  Header: header,
  Aside: aside,
  Main: packages_main,
  Footer: footer,
  Timeline: timeline,
  TimelineItem: timeline_item,
  Link: packages_link,
  Divider: divider,
  Image: packages_image,
  Calendar: calendar,
  Backtop: backtop,
  InfiniteScroll: infinite_scroll,
  PageHeader: page_header,
  CascaderPanel: packages_cascader_panel,
  Avatar: avatar,
  Drawer: drawer
});

/***/ })
/******/ ])["default"];
```
如果是直接导出的 `Element`，其实就是包含了这些组件的一个大的对象。当然还得导入 css：
```javascript
import ElementUI from 'element-ui';
import 'element-ui/lib/theme-chalk/index.css';
```
至于为啥要单独引用 css， 而不是把 css 也通过 `webpack` 打到 js 里面去。这个是因为组件式的开发， 所涉及到的所有的样式都不会写在 vue 文件里面。这个最大的原因就是后面如果要自定义主题的话，可以直接引用自定义主题的 css 文件。而不需要管这个默认主题的 css 文件。

以 `element-ui` 为例，他所有的组件都是 js 加 vue， 而 vue 文件里面全部都没有 scss 的：

![1](17.png)

他全部的 scss 文件都放在 `package/theme-chalk/src` 这个下面：

![1](18.png)

打完包的话，其实也会在 `package/theme-chalk/lib` 下也会放一份打包后的：

![1](19.png)

这个操作，其实是通过这个指令来跑的：
```javascript
gulp build --gulpfile packages/theme-chalk/gulpfile.js
```
简单的来说，就是将 `theme-chalk` 目录下 `src` 下的 `scss` 文件编译成 `css`， 放到 `theme-chalk` 目录下的 `lib` 目录,同时将字体文件也一起挪过去。最后再把编译好的 `theme-chalk/lib` 目录 拷贝到 `lib/theme-chalk`， 就可以了
```javascript
cp-cli packages/theme-chalk/lib lib/theme-chalk
```
这样就实现了 `scss` 的编译，以及 `css` 文件的单独引入机制。 而且从代码里面看，这个入口的 scss 文件，其实就是将其他的 `scss` 文件全部引入过去：

![1](20.png)

可以理解为这个 `index.css` 就是总的样式文件了。 而我们只需要引入这个 css 就可以了。 其他的都不需要引用了。

![1](21.png)

## 组件的开发
理解了构建之后，组件的开发反而是比较好理解的。`element-ui` 每一个组件都是一个单独的目录，虽然有时候会相互引用，但是逻辑几乎是互不干扰。遵循一定的目录结构，以 `alert` 这个组件为例，在 `packages` 目录的目录结构是这样子的：
```text
alert
  |---src/       
  |      |---main.vue      组件的业务逻辑
  |---index.js    组件的声明
```
这个目录是很清晰的，所以基本上大框架搞懂了，反而组件的开发是显得比较容易的，因为耦合性很低，不会影响到其他的组件。 而且从整个构建来看，其实打包出来的都没有涉及到主题自定义的方面，打包出来的 `index.css` 就是默认主题， 所以它的主题自定义功能，其实是通过用户交互界面来让用户自己手动自定义的，这个后面讲到`air-ui`怎么做主题自定义的时候，会详细讲，其实并没有所谓的优劣，就是使用场景不一样，导致的解决方式也不一样。

## 总结
通过对构建的分析，大概就可以知道 `element-ui` 的大致脉络，当然，如果说完全的了解，那肯定是不够，事实上我也没有把所有的代码全部啃一遍，一方面是时间上不允许，另一方面，我在得到我想要的思路和流程之后，很多东西我都是按照自己的方式和习惯写的，不一定要跟 `element-ui` 一样。我看源码只是为了一个思路而已，接下来讲一下我们怎么基于 `element-ui` 来做 `air-ui` 的。






