---
title: 自建vue组件 air-ui (11) -- vuepress 写文档 (爬坑版)
date: 2020-01-02 16:20:24
tags: js
categories: 
- 前端相关
- 自建vue组件 air-ui
---
## 前言
通过上节 {% post_link air-ui-10 %} 其实关于 `vuepress` 写文档的事情，也基本上能说的也差不多了。本节主要是在这个基础上再进行一下补充，将一些在 `air-ui` 组件库写文档中遇到的一些奇葩的问题，以及怎么解决的，这个也列一下，做一下备忘录。

## 第三方库 Popper 的报错
之前有遇到一个很奇怪的情况，就是写`autocomplete` demo 的时候，在本地运行中，是可以 work 的：

![1](1.png)

而且打包到另外一个项目用第三方库的方式引用，也是可以的：
```javascript
<air-col :span="12">
    <div class="sub-title">激活即列出输入建议</div>
    <air-autocomplete
            class="inline-input"
            v-model="state1"
            :fetch-suggestions="querySearch"
            placeholder="请输入内容"
            @select="handleSelect"
    ></air-autocomplete>
</air-col>
```

![1](2.png)

但是一旦放到  `vuepress`  文档里面，就会报错？？
<!--more-->
![1](3.png)

看错误，好像是读一个不存在的 `Popper` 属性，后面查了一下，发现在 `popper.js` 这个第三方库的时候，到 `root` 分支的时候，发现是 `undefined` ，导致报错：

![1](4.png)

这个 `root` 是 `undefined` ，所以会报错。 但是在项目中引用又是正常的， 估计是 `vuepress` 机制的问题？？？

后面查了一下，可能是这个问题 [浏览器的 API 访问限制](https://vuepress.vuejs.org/zh/guide/using-vue.html#%E6%B5%8F%E8%A7%88%E5%99%A8%E7%9A%84-api-%E8%AE%BF%E9%97%AE%E9%99%90%E5%88%B6)：
> 请注意，这并不能解决一些组件或库在导入时就试图访问浏览器 API 的问题 —— 如果需要使用这样的组件或库，你需要在合适的生命周期钩子中动态导入它们：

因为其实 `root` 就是浏览器的 `window` 对象，所以就是在初始化的时候，就访问 `window` 对象，但是这时候其实还没有被导入，所以可以这样子写：
```javascript
<script>
export default {
  mounted () {
    import('./lib-that-access-window-on-import').then(module => {
      // use code
    })
  }
}
</script>
```
但是这时候就会改到代码，变得跟 `air-ui` 里面的代码不一样了，所以我这边其实就是对这个引用做了一下兼容了：

![1](5.png)

加了这一行代码，如果 root 不存在的话，就将 window 赋给他， 这样就不会报错了。 然后再试一下：

![1](6.png)

发现界面正常了，但是点击输入框之后，又会报错？？？

![1](7.png)

看样子是`PopperJS` 对象不是函数？？  后面查了一下，发现是这边的问题：

![1](8.png)

这边的这个 `PopperJS` 变成对象了，不是函数了。 而这个是在这边赋值的：
```javascript
let PopperJS = Vue.prototype.$isServer ? function () {} : require('./popper');
```
这时候是赋值到 window 对象的时候，没有返回该对象导致的。 所以这边再进行兼容，如果这个对象不是函数的话，那么就将 window 的 `Popper` 属性赋给他：

![1](9.png)

这样子就可以兼容了， 然后再试一下， 这下子终于可以了：

![1](10.png)

## 第三方库 date.js 的报错
在写到 `date-picker` 也有出现另一个第三方库在项目dev环境没有问题，但是在接入 vuepress dev 环境的时候，有问题。就是 `date.js` 这个第三方库：

![1](11.png)

这个方式跟上面的 `PopperJS` 一样，这边就不多了，直接改代码， 在 `date.js` 里面补上：

![1](12.png)

然后在 `data-util.js` 也要改：
```javascript
import fecha from './date';
/* todo 这边是新增，原来的 element ui 的代码没问题，但是如果在 vuepress 文档写 vue demo 的时候，有用到这个js的组件都会报错，原因是找不到 fecha 对象，所以这边就做一下兼容，并不会影响到原来组件的逻辑 -- by zach */
let fechaModel = fecha || window.fecha;
```
因为 `ES6 import` 导出来的变量 只能 `read-only`， 不能再被赋值，因此这边要在初始化一个变量 `fechaModel` 来进行处理, 所以这个文件涉及到的相关引用都要将 `fecha` 改成 `fechaModel`:
```javascript
export const formatDate = function(date, format) {
  date = toDate(date);
  if (!date) return '';
  return fechaModel.format(date, format || 'yyyy-MM-dd', getI18nSettings());
};
export const parseDate = function(string, format) {
  return fechaModel.parse(string, format || 'yyyy-MM-dd', getI18nSettings());
};
```
然后用这个变量来处理就可以了。这样子 `vuepress` 才能 work：

![1](13.png)

## vuepress build 报错
当我组件全部写完的时候，并且本地 `vuepress` 运行的时候，例子也没有问题的时候， 这时候我要打静态文件 `build` 的时候，报错了
```javascript
F:\code\air-ui>yarn docs:build
```

![1](14.png)

`date.js` 的 `window` 对象不存在，导致报错? 但是 `vuepress` 的本地环境是正常的啊?? 为啥 `vuepress` 本地环境可以，一旦打包到静态文件就不行 ???

### 尝试用 webpack 配置解决
首先我的想法是，是不是跟 `air-ui` 组件打包的时候一样， dev 环境下，其实都不用去管， 但是打 release 包的时候， 是要排除两个文件的：
```javascript
{
  test: /\.(jsx?|babel|es6)$/,
  include: process.cwd(),
  exclude: /node_modules|utils\/popper\.js|utils\/date.\js/,
  loader: 'babel-loader'
},
```
这两个文件都是第三方库文件，在打 `dist` 包的时候，进行 `babel` 解析的时候，是要排除掉的。那么 `vuepress` 的 `dist` 模式是不是也要排除这种文件呢。这时候就要配置 `webpack` 配置了，后面查了一下，发现在 `.vuepress/config.js` 中是可以配置 `webpack` 的， 可以通过:
```javascript
chainWebpack: config => {

},
configureWebpack: (config, isServer) => {
  if (process.env.NODE_ENV === 'production') {
    // 为生产环境修改配置...
  } else {
    // 为开发环境修改配置...
  }
},
```
来配置，而且有两种方式，具体看文档这个：[vuepress 配置webpack](https://vuepress.vuejs.org/config/#configurewebpack), 我用的是 `chainWebpack` 的方式来排除这个文件：
```javascript
chainWebpack: config => {
  config.module.rule('js')
    .exclude.add('/src/utils/date.js')
},
```
但是发现这样做还是不行，还是照样报同样的错。

### 还是改代码
`yarn docs:build` 的时候会报 `window is not defined` 错误，是因为 `date.js` 这个第三方库引入了 `window` 对象, 而出现问题的原因是当开发 `VuePress` 应用时，由于所有的页面在生成`静态 HTML` 时都需要通过 `Node.js` 服务端渲染，因此需确保只在 `beforeMount`或者 `mounted` 访问`浏览器 / DOM 的 API`。
所以归根究底，就是在打 build 包的时候，这时候要生成`静态 html 文件`，所以需要通过 `Node.js` 服务端渲染，所以这时候的环境就是`服务端环境`(难怪 `window` 对象找不到)。那么既然是 `node 环境`，那么肯定是有 `global` 对象的，因此我就把 `window` 改成 `global 对象`，所以 `date.js` 中将 window 换成 global：
```javascript
if(typeof main === "undefined"){
  main = global;
}
```
后面真的打包成功了，而且不报错了。 但是发现打开页面的时候，报了这个错，

![1](15.png)

原来把 `global` 对象打进去了， 而 `global` 对象是没有在浏览器环境下的。 因此就报错了。这样就蛋疼了，`build` 的时候，没有 `window` 对象， 浏览器环境的时候，没有 `global`。 那么有没有可能打包的时候，有办法区分是 node 环境呢？
是有的，因为我想起了，另一个第三方库 `popper`，好像 `build` 的时候就没有报错？？ 他也有引用 window 对象， 为啥他就没有报错呢？？？
后面看了一下他的引用代码，原来他真的有区分，如果是node环境下的编译的时候，是不进行这个js文件引用的，也就不会触发到 window 对象了。
```javascript
let PopperJS = Vue.prototype.$isServer ? function () {} : require('./popper');
/* todo 这边是新增，原来的 element ui 的代码没问题，但是如果在 vuepress 文档写 vue demo 的时候，有用到这个js的组件都会报错，原因是找不到 root 对象，所以这边就做一下兼容，并不会影响到原来组件的逻辑 -- by zach */
if (typeof PopperJS !== 'function') {
  PopperJS = window.Popper;
}
```
所以 `Vue.prototype.$isServer` 这个方法，可以帮助我们判断当前打包环境是不是 node 环境，从而可以不进行引用。因此我们也改成一样的：
```javascript
import Vue from 'vue';
import { t } from '../locale';
// import fecha from './date';
/* todo 这边是新增，原来的 element ui 的代码没问题，但是如果在 vuepress 文档写 vue demo 的时候，有用到这个js的组件都会报错，原因是找不到 fecha 对象，所以这边就做一下兼容，并不会影响到原来组件的逻辑 -- by zach */
let fecha = Vue.prototype.$isServer ? {format: function() {}, parse: function() {}} : require('./date');
let fechaModel = fecha;
if (!Vue.prototype.$isServer && !fecha.format) {
  fechaModel = window.fecha;
}
```
不进行 `import` 引用，而是改成 `require 引用`（import 只能写在文件头）。
最后改成这样子就可以了。 如果是 node 环境，就直接给一个 空对象， 然后两个当下会用的 `format` 和 `parse` 的空函数，这个一定要设置，不然会报函数未定义的错误：

![1](16.png)

然后要判断如果不是 node 环境，这个 `fecha` 对象是否有 `format` 方法，如果没有这个方法的话，那么就要用 `window.fecha` 来代替。
其实这个逻辑就是用来 `vuepress` 本地运行的时候用的，因为那时候 `require date` 的时候，走的是 `window` 那一层，但是导出来是一个空对象，所以并没有 `format` 方法，所以才需要 `window.fecha` 对象来替代。但是跑 `air-ui` 本地项目的话，根本不会走到那边去，而是直接 `export` 导出来完整的 `fecha` 对象，不会走到那边去。

这样子，以下这种环境都可以打包成功:
- `vuepress` 的 本地运行环境 和 `build` 生成静态文件环境
- `air-ui` 的本地运行环境 和 生成 `dist` 文件正式环境。

## 总结
到这一节结束，关于 `air-ui` 用 `vuepress` 写文档遇到的问题和需要注意的问题，基本上都描述清楚了。接下来我们讲一下 `air-ui` 的多语言机制，以及跟 `element-ui` 的多语言机制有啥不同。

---
系列文章:
{% post_link air-ui-1 %}
{% post_link air-ui-2 %}
{% post_link air-ui-3 %}
{% post_link air-ui-4 %}
{% post_link air-ui-5 %}
{% post_link air-ui-6 %}
{% post_link air-ui-7 %}
{% post_link air-ui-8 %}
{% post_link air-ui-9 %}
{% post_link air-ui-10 %}
{% post_link air-ui-11 %}
{% post_link air-ui-12 %}
{% post_link air-ui-13 %}
{% post_link air-ui-14 %}
{% post_link air-ui-15 %}
{% post_link air-ui-16 %}
{% post_link air-ui-17 %}
