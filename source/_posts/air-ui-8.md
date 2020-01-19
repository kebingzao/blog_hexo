---
title: 自建vue组件 air-ui (8) -- 实现部分引入组件
date: 2019-12-18 11:31:56
tags: js
categories: 
- 前端相关
- 自建vue组件 air-ui
---
## 前言
在之前的文章中，我们已经知道怎么去创建一个组件了，并且怎么初始化，但是有没有发现一个细节，就是 `element-ui` 除了完整引入的方式，比如:
```javascript
import ElementUI from 'element-ui';
import 'element-ui/lib/theme-chalk/index.css';

Vue.use(ElementUI);
```
还支持按需引入的方式：
```javascript
import { Button, Select } from 'element-ui';

Vue.component(Button.name, Button);
Vue.component(Select.name, Select);
/* 或写为
 * Vue.use(Button)
 * Vue.use(Select)
 */
```
听说这种方式可以减少项目体积。 如果本节讲一下 `air-ui` 怎么做到按需引入。
<!--more-->
## 按需引入
说道按需引入，我的想法就是先将 `components/index.js` 这个入口文件，除了导出 `intall` 方法, 其他的组件也应该可以被导出来：
所以原先是这样子的:
```javascript
export default {
  install
}
```
改成这样子:
```javascript
export default {
  install,
  Button,
  ButtonGroup,
  Notification,
  Loading
}
```
这样子现有的几个已经做好的组件也可以一起导出来了。 然后我们改一下 `main.js` 的方式, 改成：
```javascript
// import AirUI from './components/index'
import { Button } from './components/index'
import './styles/index.scss'

// Vue.use(AirUI)
Vue.use(Button)
```
但是编译的时候，报错了：

![1](1.png)

后面查了一下，好像 `export default` 导出的方式，不支持导出多个命名模块，只能导出一个默认模块。 所以如果要导出多个命名模块的话，那么只能用 `export` 来导出模块，所以改成这样子：
```javascript
module.exports = {
  install,
  Button,
  ButtonGroup,
  Notification,
  Loading
}
```
但是发现还是报一样的错误，好像没有起效果？？ 后面在网上查了一下，好像改成这样还不够，还需要 `babel` 兼容 `es2015`, 所以我在 `.babelrc` 加上这一行:
```javascript
["es2015", { "loose": true }],
```

![1](2.png)

然后还得安装对应的 babel 插件:
```javascript
"babel-preset-es2015": "^6.24.1",
```
```javascript
yarn add babel-preset-es2015 --dev
```
这样子构建就正常了。 可以看到 

![1](3.png)

按钮是显示正常的，但是原先的那个按钮组显示确实有问题的？？ 这个是因为我没有在 `main.js` 引用这个 `button-group` 组件,所以 vue 找不到这个组件：

![1](4.png)

所以只要再把这个加上去就行了，所以改成
```javascript
import { Button, ButtonGroup } from './components/index'
import './styles/index.scss'

// Vue.use(AirUI)
Vue.use(Button)
Vue.use(ButtonGroup)
```
这样就正常了

![1](5.png)

**这边需要注意一点的是，要完成这种语法糖，上面两个条件都要满足:**
1. `export default` 改成 `module.exports`
2. 安装`babel-preset-es2015` 并修改 `.babelrc`

如果只加了 `es2015` 的 `babel` 支持，但是 `export` 没有改成 `module.exports` 的方式，虽然构建不会失败，但是界面是有问题的：

![1](6.png)

## 没法实现按需引入？
首先需要恭喜的是，我们完成了组件库中部分引入的语法糖。但是下一个问题就是，我们要实现部分引入的写法是为了按需引入从而减少项目体积。但是上面那种写法是根本不能减少项目体积，他的原理还是将整个组件库都引入，只不过使用的是导出的那几个组件。并不会减少体积。那么 `element ui` 是怎么实现按需引入的，又是怎么在不将`export default` 改成 `module.exports` 的情况下，实现部分引入的语法糖的？

## element ui 怎么实现部分导出的语法糖？
我们通过上述的实践发现，要满足两个条件，才能实现这种部分组件导出的语法糖 
```javascript
import { Button, ButtonGroup } from './components/index'
```
那为啥 `element-ui` 在 `src/index.js` 中用的也是默认的 `export default {` 的语法，那么为啥他也可以实现部分导出的语法糖？？

我们来做个试验，在 `element-ui` 的源码项目，对 `examples/entry.js` 这个入口文件做一下调整，也换成这个这种部分导出的方式:
```javascript
// import Element from 'main/index.js';
import {
  Button,
  ButtonGroup
} from 'main/index.js';

// Vue.use(Element);
Vue.use(Button);
Vue.use(ButtonGroup);
```
其他代码不变，还是用 `export default` 导出组件，然后运行一下,发现还是报错了：
```javascript
WARNING in ./examples/entry.js 26:8-14
"export 'Button' was not found in 'main/index.js'
 @ multi (webpack)-dev-server/client?http://0.0.0.0:8085 (webpack)/hot/dev-server.js ./examples/entry.js

WARNING in ./examples/entry.js 27:8-19
"export 'ButtonGroup' was not found in 'main/index.js'
 @ multi (webpack)-dev-server/client?http://0.0.0.0:8085 (webpack)/hot/dev-server.js ./examples/entry.js

```
还是显示找不到组件，这个说明一个问题，就是源码的 dev 环境的 demo 下，也是不能用这种部分导出的语法糖，但是下载的第三方包的 `element-ui` 确实是支持这种语法的。所以我怀疑它是在打组件包的过程中，是有进行一些处理的。至于是什么处理，这个后面我再说。

## element ui 按需引入是否真的能节省体积?
接下来我们在实验一下用 element ui 的按需引入是否真的可以节省体积？
### 实验基础
先用 `vue-cli` 搭建一个基础项目，然后安装 element 依赖。 然后在项目默认自带的 `HelloWorld.vue` 加上这些组件引用：
```html
<el-button type="primary">主要按钮</el-button>
<el-button type="success">成功按钮</el-button>
<el-button type="info">信息按钮</el-button>
<el-button type="warning">警告按钮</el-button>
<el-button type="danger">危险按钮</el-button>
```
接下来会有各种条件，但是都是打成 pord 环境的包，然后对比 js 和 css 的大小，从而得出是否有减少体积。其中 prod 打包指令为:
```javascript
npm run build
```
然后为了验证界面正常，我们也会启用静态 webserver 将 dist 目录开起来：
```javascript
http-server ./dist -p 8089
```
而且打完包 `dist/static` 目录下的 js 目录会有三个文件， `app.xxx.js`, `manifest.xxx.js`, `vendor.xxx.js`, 而 `element-ui` 是属于第三方工具包，所以我们只看 `vendor.xxx.js` 和 css 目录下的`app.xxx.css` 的体积变化。
### 1.不引入 element 打包
啥都不引入，打完之后:
```text
vendor.xxx.js -> 119kb
app.xxx.css -> 1kb
```

### 2.全部引入 element
```javascript
import ElementUI from 'element-ui'
import 'element-ui/lib/theme-chalk/index.css'
```
打完之后：
```text
vendor.xxx.js -> 803kb
app.xxx.css -> 229kb
```
可以很明显的看到，体积都大了

### 3.部分引入 button, css 全部引入
```javascript
import { Button } from 'element-ui'
import 'element-ui/lib/theme-chalk/index.css'
```
打完之后：
```text
vendor.xxx.js -> 803kb
app.xxx.css -> 229kb
```
发现没有任何变化？？ 这个说明按需引入和全部引入没有半毛钱差别？？ 后面查了一下，发现我们漏了一个环境，原来按需引入是要配置一下环境的：

![1](7.png)

### 4.配置后的部分引入 button, css 全部引入
我们按照它的指导配置一下， 安装插件:
```javascript
yarn add  babel-plugin-component --dev
```
然后修改 `.babelrc`:
```javascript
{
  "presets": [["es2015", { "modules": false }]],
  "plugins": [
    [
      "component",
      {
        "libraryName": "element-ui",
        "styleLibraryName": "theme-chalk"
      }
    ]
  ]
}
```
改为之后，发现打包会报错:

![1](8.png)

后面查了一下，发现少了一个依赖，然后再装:
```javascript
yarn add  babel-preset-es2015 --dev
```
这下子终于打包正常了，打完包之后：
```text
vendor.xxx.js -> 672kb
app.xxx.css -> 229kb
```
我们发现 vendor 的体积真的变少了，从 `803kb` 减少到 `672kb`。 样式还是没有变。 但是感觉减的不是很多啊，我只引入了一个 `button` 组件而已，js 的体积也从 `119kb` 变成 `672kb`，感觉也不划算，我还不如全部引入呢!!
### 5.配置后的部分引入 button, css 不引入
接下来我们试下一样部分引入，但是不显示的引入 css 文件了。结果打完包之后发现：
```text
vendor.xxx.js -> 672kb
app.xxx.css -> 229kb
```
跟上面的结果一样。这个说明 `element-ui` 按需引入的时候，会自动加载 `index.css` 文件，而不管 `main.js` 是否有引入 css 文件。

### 结论
通过上面的测试来看，`element-ui` 配合一些配置的话，还是可以实现按需引入，从而减少体积的。

## air-ui 的按需引入
`air-ui` 其实也是有按需引入的，只不过是组件单独引入的方式，以目前第一版的来看，还是在一个初始化的 `vue-cli` 项目中，我们在 `dependencies` 中加入 air-ui：
```javascript
"air-ui": "git+ssh://git@gitlab.xxx.com:air-ui.git#v0.0.30"
```
然后执行 `yarn` 安装依赖，这时候就安装成功了

![1](9.png)

我们还是一样只引入 `button` 组件，同时在默认的 `HelloWorld.vue` 补上对应的 html 代码：
```html
<air-button type="primary">主要按钮</air-button>
<air-button type="success">成功按钮</air-button>
<air-button type="info">信息按钮</air-button>
<air-button type="warning">警告按钮</air-button>
<air-button type="danger">危险按钮</air-button>
```
然后部分引入的代码如下：
```javascript
import Button from 'air-ui/lib/button'
import 'air-ui/lib/styles/button.css'

Vue.use(Button)
```
最后打包之后的体积如下：
```javascript
vendor.xxx.js -> 121kb
app.xxx.css -> 11kb
```
可以看到如果只是少数的组件的话，同样的界面，但是这种单独引入的方式，无论是 js 的体积，还是 css 的体积，都会少非常多，因为这个才是真正的只引入组件相关的逻辑和样式，其他不相干的，全部不加载。 而且还不需要对环境进行额外的配置。这一点 `element-ui` 是需要的，不然他的这种语法糖也没能起到节省体积的作用。

关于单个组件怎么打包出来的，这一点会在讲打包构建的时候，一起讲。
## 总结
目前 `air-ui` 通过部分导入的语法糖并不能真正的实现按需引入和节省体积。这一点后面可以改成跟 `element-ui` 一样，甚至更方便。 (作为一个 todo 项)

但是说真的，如果真的对体积太过 care 的话，其实 `air-ui` 的那种组件单个引入的方式更能节省体积，就是写法上没有那么优雅就是了。

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




