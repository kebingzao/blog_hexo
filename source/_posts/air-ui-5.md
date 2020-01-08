---
title: 自建vue组件 air-ui (5) -- 创建第一个组件 Button
date: 2020-01-06 16:09:57
tags: js
categories: 
- 前端相关
- 自建vue组件 air-ui
---
## 前言
通过 {% post_link air-ui-4 %} 我们已经搭好了 `air-ui` 项目，接下来就是开始写组件了。
## ui 组件的三种类型
常见的 ui 组件有三种类型：
### 标签类型
这个是最常见的组件类型，比如 `button`， `checkbox` 等等， 使用的时候是这样子的：
```html
<air-button>这是一个按钮</air-button>
```
### 内置服务类型
内置服务类型的组件，其实就是绑定到 `VUE` 的全局对象中，比如 `message`， `notification`， `loading`，使用方式是这样子的：
```javascript
this.$message({
  message: '警告哦，这是一条警告消息',
  type: 'warning'
});
```
<!--more-->
### 指令方式类型
指令方式的组件，比如 `loading`， 使用方式是这样子的 `v-xxx`：
```html
<air-button
  type="primary"
  @click="openFullScreen1"
  v-loading.fullscreen.lock="fullscreenLoading">
  指令方式
</air-button>
```
但是标签方式的组件是最常见的。 所以本节就以标签方式的 `button` 组件来说明。 另外两种方式后面有时间再讲。
## 创建 button 组件
首先在创建一个组件的时候，一定要考虑清楚所要创建组件的表现形式，比如 `button` 按钮，组件的表现方式可能有两种：
1. 单个按钮呈现，标签是 `air-button`
2. 多个按钮一起的按钮组，标签是 `air-button-group`

这边多个单词要用中划线`_` 分隔开。 所以目录结构就是这样子：
```
components/
|    |--- button/
|    |     |--- src/
|    |     |     |--- button.vue
|    |     |     |--- button-group.vue
|    |     |--- index.js
```
目录结构很简单，事实上后面所有的组件的目录结构都是这样子的， 一个 `index.js` 用来导出对象， `src` 主要写组件的逻辑和定义，以本例来说， `button.vue` 就是 `air-button` 标签的实现文件， 而 `button-group.vue` 就是 `air-button-group` 标签的实现文件。

具体代码如下： `src/components/button/src/button.vue`:
```html
<template>
  <button
    class="air-button"
    @click="handleClick"
    :disabled="buttonDisabled || loading"
    :autofocus="autofocus"
    :type="nativeType"
    :class="[
      type ? 'air-button--' + type : '',
      buttonSize ? 'air-button--' + buttonSize : '',
      {
        'is-disabled': buttonDisabled,
        'is-loading': loading,
        'is-plain': plain,
        'is-round': round,
        'is-circle': circle
      }
    ]"
  >
    <i class="air-icon-loading" v-if="loading"></i>
    <i :class="icon" v-if="icon && !loading"></i>
    <span v-if="$slots.default"><slot></slot></span>
  </button>
</template>
<script>
  export default {
    name: 'AirButton',

    inject: {
      elForm: {
        default: ''
      },
      elFormItem: {
        default: ''
      }
    },

    props: {
      type: {
        type: String,
        default: 'default'
      },
      size: String,
      icon: {
        type: String,
        default: ''
      },
      nativeType: {
        type: String,
        default: 'button'
      },
      loading: Boolean,
      disabled: Boolean,
      plain: Boolean,
      autofocus: Boolean,
      round: Boolean,
      circle: Boolean
    },

    computed: {
      _elFormItemSize () {
        return (this.elFormItem || {}).elFormItemSize
      },
      buttonSize () {
        return this.size || this._elFormItemSize || (this.$ELEMENT || {}).size
      },
      buttonDisabled () {
        return this.disabled || (this.elForm || {}).disabled
      }
    },

    methods: {
      handleClick (evt) {
        this.$emit('click', evt)
      }
    }
  }
</script>
```

`src/components/button/src/button-group.vue` 的实现代码：

```html
<template>
  <div class="air-button-group">
    <slot></slot>
  </div>
</template>
<script>
  export default {
    name: 'AirButtonGroup'
  }
</script>

```
事实上，整个逻辑的大部分实现几乎跟 `element-ui` 一模一样，只有一些小细节不一样，比如：
1. 将标签名 `el-` 改成 `air-`
2. 将导出的`name` 的 `El` 改成 `Air`
3. 将一些资源的绝对路径改成相对路径 (button这个标签组件没有引入外部资源和第三方库，所以这一点在本例看不出来)

> ps: 而且 `air-ui` 最后完成的时候，从功能上来说，绝大部分组件的功能都跟`element-ui`的对应组件一模一样，只有某些组件，比如 `color-picker`, `table` 等因为业务需求，有再进行了一些参数和方法的扩展。

而且 `index.js` 其实就是组件的导出，代码如下，`src/components/button/index.js`：
```javascript
import AirButton from './src/button'

AirButton.install = function (Vue) {
  Vue.component(AirButton.name, AirButton)
}

export default AirButton
```
如果排除掉那个 `install`, 这个文件简直可以去掉了，因为它就是把 `button.vue` 导出的对象，重新又导出了一次。 而 `install` 的存在才显得这个文件有意义。 `install` 这个方法定义是因为后续如果这个组件是要允许单独被 vue 引用的，也是单独被打包成文件出来的。 vue 设置全局组件的话，是这个 API: 
```javascript
Vue.component(yourModule.name, yourModule)
```
但是如果 vue 加载第三方库的话，是这个 API:
```javascript
Vue.use(yourModule)
```
这时候就会去执行 `yourModule` 对象的 `install` 方法。 所以 `index.js` 文件实现 `install` 方法，只是为了后面让 vue 单独设置全局组件的时候，除了可以用标准 `component` 语法之外，还可以用 `use` 语法，而 `use` 语法会触发 `install` 方法，然后再在 `install` 方法去实现 `component` 语法。所以对于 button 组件来说，如果要用 vue 设置为全局组件的话，有这两种引用方式：

```javascript
import Vue from 'vue';
import Button from './components/button'
Vue.component(Button.name, Button)
```
或者是
```javascript
import Vue from 'vue';
import Button from './components/button'
Vue.use(Button);
```
这两个效果其实是一样的。反正最后都实现了用 `component` 方法注册成全局组件。

## 测试 button 组件
既然逻辑代码写好了，接下来是测试了。 所以修改首页 `hoome.vue` 改成这样子

```html
<template>
  <div class="hello">
    <h1>{{ msg }}</h1>
    <air-button disabled>默认按钮</air-button>
    <air-button type="primary">主要按钮</air-button>
    <br>
    <air-button-group>
      <air-button type="primary">主要按钮</air-button>
      <air-button type="success">成功按钮</air-button>
      <air-button type="info">信息按钮</air-button>
      <air-button type="warning">警告按钮</air-button>
      <air-button type="danger">危险按钮</air-button>
    </air-button-group>
  </div>
</template>

<script>
  import AirButton from '../components/button/src/button'
  import AirButtonGroup from '../components/button/src/button-group'
  export default {
    data () {
      return {
        msg: `AIR-UI  -  基于vue2.x，可复用UI组件`
      }
    },
    components: {
      AirButton,
      AirButtonGroup
    },
  }
</script>
```
有效果了，但是发现并没有样式：

![1](1.png)

所以我们写好样式： `src/styles/button.scss`， 写好之后把样式加上去：
```javascript
import AirButton from '../components/button/src/button'
import AirButtonGroup from '../components/button/src/button-group'
import '../styles/button.scss'
```
但是发现编译的时候，报错：

![1](2.png)

看起来是 `sass` 模块和对应的 `loader` 没有装的缘故， 这时候要手动安装：

```vue
yarn add sass-loader node-sass --dev
```

但是发现还是报错：

![1](3.png)

网上查了一下资料，发现原来是`sass-loader` 的版本过高导致，因其最新版本为 `8.0.0`，此会导致编译出错, 我重新换成 `6.0.7` 就可以了：
```vue
yarn remove sass-loader
yarn add sass-loader@6.0.7 --dev
```

这样子，编译就可以了。

![1](4.png)

这样子这个组件就完成了。但是发现好像还可以再优化：

## 注册成全局组件对象
上面虽然组件已经做好了，但是可以看到再引用的时候，是作为局部组件来使用的，每次都要在 `home.vue` 引入 vue 文件和 css 文件。所以我们要注册成 vue 的全局组件，这样子在写 demo 的时候，就不用引入资源了。
所以 `src/main.js` 要改一下，加上以下代码：

```javascript
import Button from './components/button/src/button'
import ButtonGroup from './components/button/src/button-group'
import './styles/button.scss'

Vue.component(Button.name, Button)
Vue.component(ButtonGroup.name, ButtonGroup)
```
这样子 `button` 和 `button-group` 就会被注册成全局组件了。 所以这时候我 `home.vue` 就可以简写为这样子， `src/views/home.vue`:
```html
<template>
  <div class="hello">
    <h1>{{ msg }}</h1>
    <air-button disabled>默认按钮</air-button>
    <air-button type="primary">主要按钮</air-button>
    <br>
    <air-button-group>
      <air-button type="primary">主要按钮</air-button>
      <air-button type="success">成功按钮</air-button>
      <air-button type="info">信息按钮</air-button>
      <air-button type="warning">警告按钮</air-button>
      <air-button type="danger">危险按钮</air-button>
    </air-button-group>
  </div>
</template>

<script>
  export default {
    data () {
      return {
        msg: `AIR-UI  -  基于vue2.x，可复用UI组件`
      }
    },
  }
</script>
```
去掉了组件的导入 和 `components` 的声明。直接默认使用全局组件。

## 再优化-兼容 vue.use 语法
虽然可以注册全局组件了。但是还是不够优雅(明明是两个组件的声明，但是引入的路径竟然是在 `button` 的src 目录下，都没有分开), 而且也不能使用 `vue.use` 语法，接下来我们来兼容一下 `use` 语法，我们知道要使用 `use` 语法，一个很重要的方式就是组件要有定义 `install` 方法，比如 `button` 的 `src/components/button/index.js` 就有:
```javascript
import AirButton from './src/button'

AirButton.install = function (Vue) {
  Vue.component(AirButton.name, AirButton)
}

export default AirButton
```
但是 `button-group` 没有啊，所以我们要为 button-group 再创建一个独属于他的组件目录：并且在目录下创建一个 `index.js` 文件，完整的文件路径就是 `src/components/button-group/index.js`, 内容如下：
```javascript
import AirButtonGroup from '../button/src/button-group'

AirButtonGroup.install = function (Vue) {
  Vue.component(AirButtonGroup.name, AirButtonGroup)
}

export default AirButtonGroup
```
是的，我们不需要再去写 `button-group` 组件的逻辑代码了，因为 `button` 组件的 src 里面帮我们写好了，我们只需要引用这个 vue 文件，并且定义好 `install` 方法，最后导出就行了。 最后再调整一下 `main.js` 的写法：

```javascript
import Button from './components/button'
import ButtonGroup from './components/button-group'
import './styles/button.scss'

Vue.use(Button)
Vue.use(ButtonGroup)
```
写成这样子，就更优雅了。 

当然这边还有一个细节，还可以再优化一下，因为我们从目录结构上来说， `button` 和 `button-group` 这两个是单独的组件，那么 `button-group` 也应该也有自己的 scss 文件，但是事实上 `button-group` 相关的样式都写在 `button.scss` 上了，所以我们为了一致性，我们就新建了一个空的 scss 文件 `button-group.scss`, 所以`main.js` 可以再加上引入 `button-group.scss`:

```javascript
import Button from './components/button'
import ButtonGroup from './components/button-group'
import './styles/button.scss'
import './styles/button-group.scss'

Vue.use(Button)
Vue.use(ButtonGroup)
```

## 导入全局组件库
现在我们有两个组件了，`button` 和 `button-group`, 后面我们还会增加非常多的组件。总不可能我们后面在注册组件的时候，都要在 `main.js` 中导入这些组件的 js 和 css 文件吧，那可是太多了。所以这些组件合起来应该是一个组件库，就像 `element-ui` 一样，我们只需要引入这个组件库对象，那么就可以使用到这个组件库所包含的所有组件。

`air-ui` 也是一样的做法，因此我们也要做自己的组件库，然后将我们写的这些组件都放到这个组件库里面，然后再到 `main.js` 中去引用这个组件库，就相当于注册了这个组件库的所有的组件了。

所以 `src/components` 增加了一个文件  `src/components/index.js`:
```javascript
import Button from './button'
import ButtonGroup from './button-group'

const components = {
  Button,
  ButtonGroup
}

const install = function (Vue) {
  Object.keys(components).forEach(key => {
    Vue.component(components[key].name, components[key])
  })
}

export default {
  install
}
```
这个很好理解，因为要用 `vue.use` 来加载这个组件库，所以导出的对象，要有 `install` 方法，并且因为要把这些组件都注册成 vue 全局组件，所以这一步就在 `install` 方法里面来处理。

然后在 `main.js` 将之前的去掉，改成这样子：
```javascript
import AirUI from './components/index'

Vue.use(AirUI)
```

这样就更完美了，不过后面调试的时候，发现没有样式，查了一下，发现样式没有导入， 所以我们样式的导入方式也应该改一下，换成导入组件库样式的方式，而不是在 `main.js` 中一个组件样式一个组件样式的导入。 所以我们在 `src/styles` 目录新增加了 `index.scss`, 然后把当前的组件的样式都导入进去, `src/styles/index.scss`:
```scss
@import "./button.scss";
@import "./button-group.scss";
```
然后在  main.js 中导入这个总的组件库的 scss 文件即可了。
```javascript
import AirUI from './components/index'
import './styles/index.scss'

Vue.use(AirUI)
```
哇，这样就真的很完美了。 基本上一个最小规模的组件库的雏形就出来了。

## 总结
我们现在已经可以很好的写一些标签组件了，下节我们讲一下怎么写 指令组件和内置服务组件。






































