---
title: 自建vue组件 air-ui (4) -- air-ui 环境搭建和目录结构
date: 2019-12-07 18:13:54
tags: js
categories: 
- 前端相关
- 自建vue组件 air-ui
---
## 前言
前面花了两个篇幅来详细分析了一下 `element-ui` 这个项目：

- {% post_link air-ui-2 %}
- {% post_link air-ui-3 %}

终于进入到 `air-ui` 的篇幅了。其实前面已经说过，在`组件逻辑方面` 和 `css 方面`，`element-ui` 做的很好，我也挑不出大毛病，而且它们也不是我要重写 ui 组件的主要原因。`air-ui` 主要还是以下几个方面跟 `element-ui` 有比较大的差异，也主要是以这些方面来写文章：
1. 目录结构
2. 文档方面
3. 构建方面
4. 主题定制
5. 多语言方面以及在整个过程中，遇到的一些坑

本节主要讲 `air-ui` 环境搭建和目录结构。
<!--more-->
## 基于 vue-cli 的搭建
如果说现在最流行的 vue 项目的脚手架是什么，毫无疑问是 vue-cli, 所以 vue-cli 初始化的项目的目录结构也是最普遍的，也是最好理解的，因为受众最多。我们很多 vue 项目刚开始都是基于 vue-cli 来搭建， `air-ui` 作为一个 vue 项目，显然也是这么干的。 (事实上，`element-ui` 的目录结构为啥那么奇葩，我猜可能也是早期 vue-cli 不够普及的原因，因为早期那时候都是自建的 vue 项目)

### 初始化项目
搭建脚手架的指令：
```text
npm install -g vue-cli
vue init webpack projectName
cd projectName
npm install
npm run dev
```
当然这边的 `projectName` 就是 `air-ui`。 这样子一个简单的基于 `vue-cli` 的 vue 项目就搭建起来了。 当然 `air-ui` 是用 `yarn` 安装的，所以具体的指令是这样子的：
```text
npm install -g vue-cli
vue init webpack air-ui
cd air-ui
yarn
yarn dev
```
创建项目的时候，一路默认选项就行了。只不过安装依赖的时候，用 `yarn` 安装就行了。 最后本地测试跑起来就可以了：

![1](1.png)

`air-ui` 用的 `vue-cli` 版本是 `3.0.1`
```text
F:\code\air-ui>vue --version
3.0.1
```

### webpack 的版本
默认搭载的是 `webpack 3.X` 版本，从 `package.json` 和 `yarn.lock` 来看， 当前具体的 `webpack` 版本是 `3.12.0`, 说明`vue-cli 3.x` 搭载的 `webpack` 并没有升级到 `webpack 4.x` 版本。(这一点跟 `element-ui` 不太一样，`element-ui`  从 `2.4.11` 之后的版本，他们就将 `webpack` 升级到了 `4.x` 版本了, 当然这个只涉及到打包效率和速度，后面完全可以手动将其升级到 `4.x` 版本)

### vue 的版本
`package.json` 和 `yarn.lock` 来看, `vue-cli 3.x` 搭载的 `vue` 肯定也不是最新的 `vue 3.x`, 具体的版本是 `2.6.11`。 现在最新的 `vue 3.x` 也出来了，后面也是可以手动将其升级到 `vue 3.x` 的。

## air-ui 的完整目录对比
初始化的用 `vue-cli` 搭建的项目的目录结构如下：

---
| air-ui
| --- build/    `打包逻辑`
| --- | --- build.js 
| --- | --- check-version.js 
| --- | --- logo.png
| --- | --- utils.js
| --- | --- vue-loader.conf.js
| --- | --- webpack.base.conf.js
| --- | --- webpack.dev.conf.js
| --- | --- webpack.prod.conf.js
| --- config/ `打包逻辑配置文件`
| --- | --- dev.env.js
| --- | --- index.js
| --- | --- prod.env.js
| --- | --- test.env.js
| --- src/ `业务逻辑`
| --- | --- assets/ `存放静态资源`
| --- | --- | --- logo.png
| --- | --- components/ `存放组件文件`
| --- | --- | --- HelloWorld.vue
| --- | --- router/ `路由文件`
| --- | --- | --- index.js
| --- | --- App.vue `vue 入口模板`
| --- | --- main.js `入口js文件`
| --- static/ `存放前端静态资源，默认为空`
| --- test/   `单元测试文件`
| --- .babelrc   `babel 配置文件`
| --- .eslintignore   `eslint 的忽略配置文件`
| --- .eslintrc.js   `eslint 的配置文件`
| --- .postcssrc.js   `postcss 的配置文件`
| --- index.html   `项目首页入口`
| --- package.json   `打包入口文件`
| --- README.md   `readme 文件`
| --- yarn.lock   `yarn lock 文件`

---

接下来是完整的 `air-ui` 最后的目录结构：

---
| air-ui
| --- build/    `打包逻辑`
| --- | --- build.js 
| --- | --- check-version.js 
| --- | --- <font color=red>git-release.sh `pub发布的脚本`</font>
| --- | --- logo.png
| --- | --- utils.js
| --- | --- vue-loader.conf.js
| --- | --- webpack.base.conf.js
| --- | --- <font color=red>webpack.common.js `打common组件包的任务`</font>
| --- | --- <font color=red>webpack.component.js `打单个组件的任务`</font>
| --- | --- webpack.dev.conf.js
| --- | --- webpack.prod.conf.js
| --- config/ `打包逻辑配置文件`
| --- | --- dev.env.js
| --- | --- index.js
| --- | --- prod.env.js
| --- | --- test.env.js
| --- <font color=red>docs/ `文档`</font>
| --- <font color=red>lib/ `打组件包的 dist 目录`</font>
| --- src/ `业务逻辑`
| --- | --- assets/ `存放静态资源`
| --- | --- | --- logo.png
| --- | --- components/ `存放组件文件`
| --- | --- | --- ~~~HelloWorld.vue~~~
| --- | --- | --- <font color=red>button/ `具体的组件`</font>
| --- | --- | --- <font color=red>xxxxx/ `以下省略 N 个组件目录`</font>
| --- | --- <font color=red>directives/ `自定义的指令`</font>
| --- | --- <font color=red>lang/ `多语言文件夹`</font>
| --- | --- <font color=red>locale/ `多语言逻辑相关目录`</font>
| --- | --- <font color=red>mixins/ `共用 mix 方法`</font>
| --- | --- router/ `路由文件`
| --- | --- | --- index.js
| --- | --- <font color=red>styles/ `存放 sass 样式文件`</font>
| --- | --- <font color=red>theme/ `存放 theme 文件`</font>
| --- | --- <font color=red>transitions/ `存放 transitions 组件的文件`</font>
| --- | --- <font color=red>utils/ `存放公共方法文件`</font>
| --- | --- <font color=red>views/ `存放开发环境的模板文件`</font>
| --- | --- App.vue `vue 入口模板`
| --- | --- main.js `入口js文件`
| --- static/ `存放前端静态资源，默认为空`
| --- test/   `单元测试文件`
| --- .babelrc   `babel 配置文件`
| --- .eslintignore   `eslint 的忽略配置文件`
| --- .eslintrc.js   `eslint 的配置文件`
| --- .postcssrc.js   `postcss 的配置文件`
| --- <font color=red>components.json   `组件的json文件`</font>
| --- <font color=red>gulpfile.js   `gulp 打包文件`</font>
| --- index.html   `项目首页入口`
| --- package.json   `打包入口文件`
| --- README.md   `readme 文件`
| --- yarn.lock   `yarn lock 文件`

---

可以看到 `air-ui` 就是基于 `vue-cli` 项目的初始目录结构的。 只不过在这个基础上多了一些新的目录扩展，就是以上 红色标出来的目录结构：

1. `build` 目录增加了打包组件和发布的相关脚本和任务
2. 增加了 `docs` 用来存放文档和demo例子相关
3. `lib` 就是打包组件后的 `dist` 目录，跟 `element-ui` 打包 dist 一样
4. `components` 目录还是只存放组件相关的，当然之前默认的 `HelloWorld.vue` 就删掉了
5. `src` 下的 `directives`,`locale`, `mixins`, `transitions`, `utils` 跟 `element-ui` 的 `src` 的那几个目录是一样的作用
6. `src` 下的 `lang` 就是多语言词条文件的目录，跟`element-ui` 的 `src/locale/lang` 的这个目录性质一样，只不过我们抽到了 `src` 下目录，其实这样是有好处的，后面我们讲到多语言机制的时候，会说道为啥要这么做。
7. `src` 下的 `styles` 就是存放 sass 样式文件，这个目录下的子目录结构，跟 `element-ui` 的 `packages/theme-chalk/src` 的目录结构一样，是借鉴过来的，一样有子目录 `common`, `date-picker`, `fonts`, `mixins`， 还有各个组件的对应的 sass 文件。
8. `src` 下的 `theme` 存放主题的一些配置文件，后面讲到主题的时候会讲到，这个目录也是 `element-ui` 没有的。
9. `src` 下的 `views` 存放本地开发环境下的模板文件，相当于之前的 `HelloWorld.vue`
10. 根目录下的 `components.json` 跟 `element-ui` 根目录下的 `components.json` 文件是一样的，都是一些需要单独再打包的组件的列表
11. `gulpfile.js` 就是 gulp 配置文件，`air-ui` 的构建包含了很多 gulp 任务，这一点跟 `element-ui` 还不太一样。

## 改成新的目录结构跑起来
改成新的目录结构之后，接下来一样本地跑起来，`yarn dev`, 构建方式还是一样，不需要调整，不过因为 `vue-router` 那边有调整。
之前在 `src/router/index.js` 首页展示是去读取 `src/components/HelloWorld.vue` 这个文件的:
```vue
import Vue from 'vue'
import Router from 'vue-router'
import HelloWorld from '@/components/HelloWorld'

Vue.use(Router)

export default new Router({
  routes: [
    {
      path: '/',
      name: 'HelloWorld',
      component: HelloWorld
    }
  ]
})
```
现在这个文件以及不存在了。换成是 `src/views/home.vue` 这个文件了。所以代码要改成这样子:
```vue
import Vue from 'vue'
import Router from 'vue-router'

Vue.use(Router)

export default new Router({
  routes: [
    {
      path: '/',
      name: 'home',
      component: (resolve) => {
        require(['@/views/home'], resolve)
      }
    }
  ]
})
```
这样子就可以了。 如果要修改首页的内容，直接修改 `src/views/home.vue` 就行了，比如改成这样子:
```vue
<template>
  <div class="hello">
    <h1>{{ msg }}</h1>
  </div>
</template>

<script>
export default {
  data () {
    return {
      msg: `AIR-UI  -  基于vue2.x，可复用UI组件`
    }
  }
}
</script>
```
那么首页就是

![1](2.png)

当然要是觉得 logo 太显眼，要换或者去掉，直接去 `App.vue` 把他干掉。

## 总结
本节主要是讲怎么基于 `vue-cli` 来搭建 `air-ui` 的项目雏形。并成功运行起来。下一节讲怎么在 `air-ui` 怎么创建第一个属于自己的组件。

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