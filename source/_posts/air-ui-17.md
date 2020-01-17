---
title: 自建vue组件 air-ui (17) -- 开发爬坑篇以及总结
date: 2020-01-16 13:53:14
tags: js
categories: 
- 前端相关
- 自建vue组件 air-ui
---
## 前言
前面讲了很多关于 `air-ui` 组件的东西，包括多语言，主题定制，构建等等。但是其实一个组件库包含那么多的组件，在开发的时候，难免会遇到很多奇奇怪怪的问题，本节就讲一下在开发这些组件的时候，遇到的一些问题，以及怎么解决。(虽然很大程度上复用了 `element-ui` 的组件逻辑代码，但是难免还会有其他的坑)。

## 兼容 jsx 语法
之前在处理 jsx 语法的时候，有报错：
```javascript
const wrap = (
  <div
    ref="wrap"
    style={ style }
    onScroll={ this.handleScroll }
    class={[this.wrapClass, 'air-scrollbar__wrap', gutter ? '' : 'air-scrollbar__wrap--hidden-default']}
  >
    { [view] }
  </div>
);
```
<!--more-->
导致编译的时候报错了，后面查了一下，发现是 `.babelrc` 并没有兼容 jsx， 所以 `.babelrc` 要改下, `plugins` 数组要加上 `transform-vue-jsx`
```javascript
"plugins": ["transform-runtime","transform-vue-jsx"],
```
然后安装对应的依赖：
```javascript
"babel-plugin-syntax-jsx": "^6.18.0",
"babel-plugin-transform-vue-jsx": "^3.7.0",
```
这样就可以了。

> 后面发现原来 vue-cli 安装的初始项目配置就有兼容 jsx 语法了，只不过我那时候将之去掉了，导致报错了，所以又加上去了。

## 没法触发 transition 类
还出现一个很神奇的情况，之前在做 `collapse-transition` 这个组件的时候，跟 `element-ui` 一样的代码，就是一直没法触发到 `Transition` 类里面的自定义方法。找了很多方式都解决不了，也不知道是不是跟 vue 的版本有关，到最后才用了另外一种方式去处理才可以：
```javascript
export default {
  name: 'AirCollapseTransition',
  functional: true,
  render(h, { children }) {
    let transitionModel = new Transition();
    const data = {
      on: {
        beforeEnter: transitionModel.beforeEnter,
        enter: transitionModel.enter,
        afterEnter: transitionModel.afterEnter,
        beforeLeave: transitionModel.beforeLeave,
        leave: transitionModel.leave,
        afterLeave: transitionModel.afterLeave
      }
    };
    /** 这边真是日了狗，element 的这种写法，到我这边竟然没法触发到类里面的方法，只能用上面那种拆分的方式来处理才行 -- by zachke
    const data = {
      on: new Transition()
    };
     */
    return h('transition', data, children);
  }
};
```
## css 样式会相互覆盖的情况
之前有发现一个问题，就是在 `index.scss` 里面的 `scss` 排列还不能随便排列？比如如果把 `@import "./date-picker.scss"`  排在 `@import "./cascader-panel.scss"` 这个后面，就会出现 css 优先级被覆盖的情况（`element-ui` 也是一样的情况）：

![png](1.png)

这个是因为他们都有导入 `@import "./scrollbar"`  这 css。如果 `data-picker` 在后面加载引入的话，就会导致同及层下， `el-scrollbar__warp`  又再一次的覆盖了 `el-cascader-minu__wrap` 这个类，导致界面出问题了：

![png](2.png)

所以写 scss 的时候，一定要注意相互覆盖的问题，因为 `BEM` 规则一般只有一层，如果这个组件被其他组件引用的话，就很有可能会出现优先级覆盖的问题。

## inline-block 的问题
之前遇到了一个非常奇怪的问题，就是在做 `time-picker` 的时候，demo 代码是这样子的:
```javascript
<template>
  <air-time-picker
    is-range
    v-model="value1"
    range-separator="至"
    start-placeholder="开始时间"
    end-placeholder="结束时间"
    placeholder="选择时间范围">
  </air-time-picker>
  <air-time-picker
    is-range
    arrow-control
    v-model="value2"
    range-separator="至"
    start-placeholder="开始时间"
    end-placeholder="结束时间"
    placeholder="选择时间范围">
  </air-time-picker></template>
<script>
  export default {
    data() {
      return {
        value1: [new Date(2016, 9, 10, 8, 40), new Date(2016, 9, 10, 9, 40)],
        value2: [new Date(2016, 9, 10, 8, 40), new Date(2016, 9, 10, 9, 40)]
      }
    }
  }</script>
```
然后 css 是默认设置宽度各 50%的:

![png](3.png)

但是却被分成两行:

![png](4.png)

调成 49% 就一行，一旦换成 50% 就两行。 而且 `element-ui` 设置为 50% 不会变成两行。而且我查了一下， css 全部都一样，并没有存在覆盖的问题？？？
直到我设置了一个背景颜色才发现端倪:

![png](5.png)

发现两个中间竟然有一个间隙。后面查了一下，发现是因为 `inline-block` 引起的？？ 间隙产生的原因是`inline-block`对外是`inline`，对内是`block`。而`inline` 会将连续的空白符解析为一个空格（如：下面示例的两个`div`之间的后面的换行空格）。
之前是写成这样子的：

![png](6.png)

所以两个 div 之间的换行就变成空格了。所以只要把换行去掉，写成一行就行了：

![png](7.png)

这样之后设置为 50%， 就没有问题了。当然这个只是在开发环境下有问题， 打包因为有压缩，所以这个空格会被压缩掉，所以是正常的。

![png](8.png)

## provide/inject
这个不是 bug，而是我看 `element-ui` 源码的时候，有用了非常多的这个特性。尤其是表单和表单组件之间的联动。都是通过这两个来进行。
具体文档:[provide / inject](https://cn.vuejs.org/v2/api/#provide-inject), 文档也说的很清楚:
> provide 和 inject 主要在开发高阶插件/组件库时使用。并不推荐用于普通应用程序代码中。

简单的来说，就是在父组件中通过provider来提供变量，然后在子组件中通过inject来注入变量。

>需要注意的是这里不论子组件有多深，只要调用了inject那么就可以注入provider中的数据。而不是局限于只能从当前父组件的prop属性来获取数据。

举一下 `air-ui` 里面的例子:

`air-form` 这个组件就会提供 provider 这个属性:
```javascript
  provide() {
    return {
      elForm: this
    };
  },
```
然后 `air-form-item` 就会用 inject 属性引入 elForm 对象，同时将自己的 elFormItem 对象 provider 出来:
```javascript
  provide() {
    return {
      elFormItem: this
    };
  },

  inject: ['elForm'],
```
接下来只要在 `air-form-item` 里面的表单组件，比如 `air-checkbox`, `air-button`, `air-input` 的 vue 定义文件里面都会有这个:
```javascript
    inject: {
      elForm: {
        default: ''
      },
      elFormItem: {
        default: ''
      }
    },
```
然后就可以跟 form 或者 form-item 进行联系了，比如一些计算属性：
```javascript
    computed: {
      _elFormItemSize() {
        return (this.elFormItem || {}).elFormItemSize;
      },
      validateState() {
        return this.elFormItem ? this.elFormItem.validateState : '';
      },
      needStatusIcon() {
        return this.elForm ? this.elForm.statusIcon : false;
      },
      isDisabled() {
        return this.isGroup
          ? this._checkboxGroup.disabled || this.disabled || (this.elForm || {}).disabled || this.isLimitDisabled
          : this.disabled || (this.elForm || {}).disabled;
      },
```
这样就可以跟表单的状态联系在一起了。

## 总结
因为有 `element-ui` 的代码珠玉在前，所以老实说 `air-ui` 的同样组件的逻辑代码几乎不需要太多的修改。当然该爬的坑还是要爬出来。
到现在，`air-ui` 其实总体的框架已经很完善了。除了一些 `element-ui` 没有的组件的继续开发，当然还有一些缺少, 比如:
1. webpack 打包升级到 4.x
2. vue 版本升级到 3.x
3. 真正按需加载的优化
4. ts 定义文件的开发
5. 各种主题定制的颗粒度细化，以及出对应的设计规范文档

尤其是第五点，其实我们现在就在继续做了，从主题定制，再到出对应的 ui 设计规范，然后后面让 ui 出图的时候，严格按照这一份设计规范来处理。 当然这些免不了我可爱的组员们的添砖加瓦，事实上当我将 `air-ui` 的第一版搭出来之后，其实后面的新组件开发，主题定制，出设计规范，都是他们在处理，我又变成甩手掌柜了。 谢谢 `志杰`,`涵涵`，`鸿鑫`，`皓宇`, `韫毅` 接下来对 `air-ui` 组件库的继续添砖加瓦和持续优化。


