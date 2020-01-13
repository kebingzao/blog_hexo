---
title: 自建vue组件 air-ui (9) -- 用 vuepress 写文档
date: 2020-01-09 16:17:03
tags: js
categories: 
- 前端相关
- 自建vue组件 air-ui
---
## 前言
从前面的章节中，我们知道怎么创建组件，但是我们测试组件属性的时候，都写在 `home.vue` 中，而且只有 demo，没有配上对应的属性描述。所以接下来我们要做的就是开始为这些已完成的组件来写文档。
## element-ui 的文档
我们可以看到 `element-ui` 的文档还是写的非常好的。

![1](1.png)

不过他这个文档的项目，其实就是他们自己写的。感兴趣的可以自己去看代码，就在源码项目的 `examples` 目录，执行 `yarn dev` 就可以把这个站点跑起来。它有例子也有对应的代码和注释。可以说非常的清晰。但是不适合 `air-ui`, 原因有两个:
1. 自己从头再去搞一个文档项目，性价比太低了，没有那么时间。
2. 他们是先写 demo 代码，再把代码通过 `markdown-it` 编译成 vue 文件，然后再嵌进去的，而且样式也是写在一起的。导致后面如果要新加一个组件的说明文档，得去调代码，调完代码再写进去。无论是扩展还是维护，其实成本都不低。

所以总的来说，看起来效果不错，但是真的不适合我，性价比太低。可以先看一下，最后 `air-ui` 成形的文档效果：
<!--more-->
![1](2.gif)

## 选中 vuepress
首先排除掉自己写的可能性，用网上现成的文档框架，在网上找了一圈，后面锁定了两个：
- [docsify](https://docsify.js.org/#/zh-cn/)
- [vuepress](https://vuepress.vuejs.org/zh/guide/#%E5%AE%83%E6%98%AF%E5%A6%82%E4%BD%95%E5%B7%A5%E4%BD%9C%E7%9A%84%EF%BC%9F)

这两个文档框架，其实都是用 vue 写的，都挺不错的，而且我们团队的文档站点就是用 `docsify` 搭的，简单易用，又方便。 但是后面还是选择了 `vuepress`， 最大的原因就是 `vuepress` 可以嵌入 `vue demo`，而且很容易扩展，无论是主题还是插件。

最后看一下 `air-ui` 现在的文档站点的截图:

![1](2.png)

跟 `element-ui` 很像，该有的都有。而且无论是扩展还是维护，都非常的简单。

## 接入 vuepress
回到之前的那个项目，接下来我们要针对之前做的那几个组件，开始接入文档了。
文档写的很清楚了：[vuepress 现有项目快速安装](https://vuepress.vuejs.org/zh/guide/getting-started.html#%E7%8E%B0%E6%9C%89%E9%A1%B9%E7%9B%AE), 我这边不再多说：
```javascript
# 将 VuePress 作为一个本地依赖安装
yarn add -D vuepress # 或者：npm install -D vuepress

# 新建一个 docs 文件夹
mkdir docs
```
在 `package.json` 里加一些脚本:
```javascript
"scripts": {
  ...
  "docs:dev": "vuepress dev docs",
  "docs:build": "vuepress build docs"
  }
```
接下来启动 `vuepress` 开发环境:
```javascript
yarn docs:dev
```
可以在浏览器看到效果了：

![1](3.png)

不过因为没有文件，所以是 404， 所以我们接下来在 `docs` 增加一个 `README.md` 文件， 内容如下:
```javascript

---
title: 介绍
---
# AIR-UI 
> 基于vue 2.x 开发的的UI组件库

## 简介
内部项目组件库， 基础组件是基于 `Element UI` 为基础的。 目录结构是通过 `vue-cli` 这个脚手架搭起来的：

todo...
```
因为有热更新，所以浏览器会自动刷新，可以看到效果

![1](4.png)

但是感觉还是不太对，好像少了什么，原来顶部和左边的导航栏都没有？？ 那么导航栏要怎么写，看了一下文档，原来要写在 `docs/.vuepress/config.js` 这个文件中。所以第一步我们要先把目录结构给建起来，具体看 [vuepress 目录结构](https://vuepress.vuejs.org/zh/guide/directory-structure.html),所以目录结构就是:
```
docs/
|    |--- .vuepress/
|    |     |--- components/ 存放组件 vue demo
|    |     |--- public/ 存放公共静态资源
|    |     |--- config.js 配置文件的入口文件
|    |--- component/  存放各组件的wiki文档
|    |--- guide/  存放引导相关的文档
|    |     |--- start.md 存放快速上手的文档
|    |--- README.md  首页文档
```
目前这个目录结构就可以满足我们的文档要求。首先我们要配置 `config.js` 来生成顶部和侧边的导航栏：
```javascript
module.exports = {
  title: 'AIR-UI',
  description: 'A UI library, built with Vue.js 2.0',
  head: [
    ['link', { rel: 'icon', href: '/logo.png' }]
  ],
  base: '/',
  themeConfig: {
    nav: [
      { text: '首页', link: '/' },
      { text: '指南', link: '/guide/start' },
      { text: '文档', link: 'http://air-ui.xxx.com' },
      { text: 'GitLab', link: 'http://gitlab.xxx.com' }
    ],
    sidebar: [
      {
        title: '开发指南',
        collapsable: false,
        children: [
          '/',
          '/guide/start',
        ]
      }
    ]
  }
};
```
这边要注意一个细节，如果修改了 `config.js` 这个配置文件，那么是不会触发热更新的，要重新执行指令才行。可以看到有效果了：

![1](5.png)

## 开始为组件写文档
接下来我们开始为组件写文档了。 首先为 `button` 写文档, 这边要注意一个细节，就是虽然我们有 `button` 和 `button-group` 这两个组件，但是其实他们都属于 `button` 这个 `html` 标签范畴的，所以文档写一起就可以了，不需要为每一个单独的组件写独立的文档。事实上好几个组件如果有关联的话，是可以文档写在一起的，这样子看也会比较容易理解。

所以我们在 `config.js` 的 `sidebar` 增加这个:
```javascript
{
  title: '组件',
  collapsable: false,
  children: [
    {
      title: '通用',
      collapsable: false,
      children: [
        '/component/Button',
      ]
    }
  ]
}
```
然后在 `docs/component` 目录下增加 `Button.md` 文件:
```javascript
# Button 按钮
常用的操作按钮
## 基础用法
基础的按钮用法
```
这样子我们就可以看到这个组件的 wiki 页面了：

![1](6.png)

## 写 demo
之前我们写组件的`demo`实例都是写在 `home.vue` 上的，现在有了文档，我们当然要像 `element-ui` 那样在文档里面嵌入 demo 和对应的代码，这样才显得有诚意。

`vuepress` 是支持嵌入 vue 例子的，用这种语法：[vuepress 在 Markdown 中 使用 Vue](https://v1.vuepress.vuejs.org/zh/guide/using-vue.html):
```javascript
<ClientOnly>
  <NonSSRFriendlyComponent/>
</ClientOnly>
```
所以我们接下来先写一个 vue demo，然后放入之前建的 `Button.md` 看看有没有效果。在 `vuepress` 中写 demo 是有路径要求的，文件要在 `docs/.vuepress/components`目录中，因为每一个组件都会有很多的 demo 来解释不同属性和参数，所以我们在这个目录下建一个 `button` 目录，然后在底下建一个 `demo-base.vue` 的 vue demo:

![1](7.png)

代码如下：
```javascript
<template>
  <div>
    <air-button>默认按钮</air-button>
    <air-button type="primary">主要按钮</air-button>
    <air-button type="success">成功按钮</air-button>
    <air-button type="info">信息按钮</air-button>
    <air-button type="warning">警告按钮</air-button>
    <air-button type="danger">危险按钮</air-button>
  </div>
</template>

<script>
  import Button from '../../../../src/components/button'
  import '../../../../src/styles/index.scss'
  export default {
    name: 'button-demo-base',
    components: {
      [Button.name]: Button,
    }
  }
</script>
```
因为运行的环境是在 `vuepress` 这个项目，所以这里面就不是注册的全局组件了，所以要当做局部组件引用。而且要注意一点的是 `template` 标签下一定要有一个 `div` 将内容包住，不然内容会变成空。接下来 `Button.md` 改成:
```javascript
# Button 按钮
常用的操作按钮
## 基础用法
基础的按钮用法

<ClientOnly>
 <button-demo-base></button-demo-base>
</ClientOnly>
```
这边要注意一个细节就是查找 `button-demo-base` 对应的 vue 文件的时候, `vuepress` 会先将 `button-demo-base` 字符串按照中划线`-`分成三个单词 `button`, `demo`, `base`,  接下来的检索模式就是去找 `.vuepress/components/` 目录下：
1. 先优先在该目录下找 `button` 目录下的 `demo` 目录下的 `base.vue`
2. 如果找不到，再找 `button` 目录下的 `demo-base.vue`
3. 如果再找不到，最后找 `button-demo-base.vue`

如果三个文件都存在，就优先找第一种情况。后面两种情况忽略。所以我们可以知道他是通过目录结构是检索 vue 例子的，跟你 vue demo 里面的 name 属性没有关系，事实上在本例中，如果把例子中的 export 中的 name 属性去掉，也是没有影响的。 `vuepress` 照样可以找到对应的 vue 文件。之所以我们把它加进去是为了后续要在 `home.vue` 中进行引用，那时候就需要根据 name 属性了。

当然对于本例子来说，第一种情况肯定是找不到的，因此找到的是第二种情况。 接下来我们看下效果：

![1](8.png)

可以看到在文档中出现 demo 了。

## 做一个通用的包裹 demo 的外层
虽然现在我们已经能够在 `vuepress` 写 demo 了，但是我们发现其实写 demo 还是很不方便的，而且 demo 的范围并没有跟下面的文字说明隔离开，所以我们需要有一个额外的专门用于包裹 demo 的组件： `demo-block`, 后面我们写 demo 的时候，就可以用这个组件包裹起来，而且可以实现一些界面的调整，比如 demo 的边距，边框等等。

本质上来说 `demo-block` 也是一个组件，只不过这个组件不是 `air-ui` 组件库里面的组件，但是他的目录结构应该跟其他的 `air-ui` 组件一样：

```
components/
|    |--- demo-block/
|    |     |--- src/
|    |     |     |--- demo-block.vue
|    |     |--- index.js
```
就两个文件，其中一个是定义的 vue 文件，一个是初始化引入的 js 文件。 `demo-block.vue`:
```javascript
<template>
  <div class="demo-block"
       :class="[
       customClass ? customClass : '',
       { 'demo-not-bordered': noBorder }]"
  >
    <div class="source">
      <slot></slot>
    </div>
  </div>
</template>

<script type="text/babel">
  export default {
    name: 'DemoBlock',
    props: {
      noBorder: Boolean,
      customClass: String
    },
  }
</script>

<style lang="scss">
  .demo-block {
    border: solid 1px #ebebeb;
    border-radius: 3px;
    margin-top: 10px;
    transition: .2s;
    .air-row {
      margin-bottom: 20px;
      &:last-child {
         margin-bottom: 0;
       }
    }
    .air-button-group {
      .air-button+.air-button {
        margin-left: 0;
      }
    }
    .source {
      padding: 24px;
    }
  }
  .demo-not-bordered{
    border: 0;
    .source {
      padding: 0;
    }
  }
</style>
```
逻辑其实很简单，就是多包了两层 div，然后设置边距，并且根据一些参数判断是否要显示边框，或者要加入自定义的类名。 而且可以看到这边是 scss 样式是写在一起的，没有单独分开，这个是因为这个组件本来就不是 `air-ui` 组件库里面的组件，也不会在打包的时候被导出。所以 scss 直接写在一起，不用再写到 styles 目录里面去。

而 `index.js` 就是初始化定义， 一样可以用 use 的语法来注册全局组件:
```javascript
import DemoBlock from './src/demo-block'

DemoBlock.install = function (Vue) {
  Vue.component(DemoBlock.name, DemoBlock)
}

export default DemoBlock
```
既然 `demo-block` 写好了，那么接下来就可以将 `button/demo-base.vue` 改成：
```javascript
<template>
  <demo-block>
    <air-button>默认按钮</air-button>
    <air-button type="primary">主要按钮</air-button>
    <air-button type="success">成功按钮</air-button>
    <air-button type="info">信息按钮</air-button>
    <air-button type="warning">警告按钮</air-button>
    <air-button type="danger">危险按钮</air-button>
  </demo-block>
</template>

<script>
  import Button from '../../../../src/components/button'
  import DemoBlock from '../../../../src/components/demo-block'
  import '../../../../src/styles/index.scss'
  export default {
    name: 'button-demo-base',
    components: {
      [Button.name]: Button,
      [DemoBlock.name]: DemoBlock,
    }
  }
</script>
```
引入 `demo-block` 组件，并且包裹内容。我们看下效果:

![1](9.png)

果然看到 demo 有边框了，很明显的跟文档的其他分开了。 接下来我们再在 `Button.md`下面补上对应的代码和对应的提示,不需要 demo 里面所有的代码都贴上去，比如那些 import 引用就没啥意义，因为那是为 demo 特意写的，对阅读的人来说，是一种阅读障碍，所以只需要取跟 demo 特殊化没关系的代码，比如：

![1](10.png)

实际效果如下：

![1](11.png)

这个效果已经很像 `element-ui` 效果了。 这个只是最简单的一个示例 demo，事实上最后单单 button 的文档的例子，就有 7 个例子：

![1](12.png)

最后的效果如下：

![1](1.gif)

## prod 打包静态文件
如果后面要通过 `CI` 构建将文档更新到站点的话，就必须打包成静态文件，那么指令就是:
```javascript
yarn docs:build
```
```javascript
"docs:build": "vuepress build docs"
```
默认的生成路径是： `docs\.vuepress\dist` , 这样就可以了。 

不过这边要注意一个细节，就是打包的静态文件的资源的相对 url 的配置，是在 `docs/.vuepress/config.js` 里面配置的：
```html
base: '/',
```
这个很重要，因为我默认设置的是根目录。如果要跑起来的话，那么就要创建一个以 dist 为根目录的 webserver。

假设你有安装了[http-server](https://www.npmjs.com/package/http-server)的情况下：
```html
npm install http-server -g
```
那么我们就可以通过执行这个指令把服务启动起来：
```html
http-server ./docs/.vuepress/dist -p 8085
```
这样就可以了：
```html
F:\code\vue-ui>http-server ./docs/.vuepress/dist -p 8085
Starting up http-server, serving ./docs/.vuepress/dist
Available on:
  http://192.168.40.51:8085
  http://192.168.111.1:8085
  http://192.168.218.1:8085
  http://192.168.56.1:8085
  http://192.168.99.1:8085
  http://127.0.0.1:8085
Hit CTRL-C to stop the server
```


## 总结
本节主要讲 `air-ui` 怎么选用 `vuepress` 作为写文档的框架。 并且初步完成了一个简单的 button 的文档页面。 但是这个只是最基础的，事实上哪有那么简单，而且还有很多东西要优化和要解决，比如：

1. 每次写 demo 都要手动导入很多组件和对应的css，太麻烦了，能不能注册成全局的
2. `button` 这种标签组件还能用导入的方式来写，那 `notification` 这种内置服务组件呢？ 怎么写？
3. `vuepress` 中间的宽度太窄了，怎么修改？
4. `vuepress` 默认的 table 的样式边框太丑了，我要怎么改成跟 `element-ui` 的文档表格那样好看？

下一节我们将继续讲 `vuepress` 写文档的进阶版。

