---
title: 自建vue组件 air-ui (1) -- 为啥我要自建一个类 element ui 的组件 
date: 2019-12-01 13:28:19
tags: js
categories: 
- 前端相关
- 自建vue组件 air-ui
---
## 前言
最近的一个月大部分时间都在忙这个事情，就是以 `element ui` 为基础，做一个属于我们自己的 vue 组件库，并且扩展。 现在已经完成了。并且已经成功替换原项目里面的 `element ui` 组件，并且开始要编写其他的 `element ui` 组件没有的组件。

## 效果如下
几个效果如下： wiki 文档截图如下：

![1](2.gif)

因为要完整兼容我们之前用的 `element ui` 组件，所以这个 `air-ui` 的组件列表也会跟 `element ui` 的组件一样。 以下是存放所有组件的效果图，这个页面也是我们后面做主题自定义的时候的对比页面，存放了所有组件的 demo 的实例：

![1](1.gif)

<!--more-->
## 为啥要这么做
首先不可否认的是， `element ui` 组件真的写的非常好了。早期的项目用它完全没有问题。 但是随着项目的规模越来越大，渐渐暴露了以下几个问题：
1. 有些组件 `element ui` 没有，我们得自己封装到我们的项目。比如以下这几个组件：
  1. 图片裁剪组件 (图像根据比例裁剪，可指定分辨率裁剪)
  
  ![1](3.png)

  2. 图片拖动上传popover

  ![1](4.png)
  
  3. 文字内容与颜色popover

  ![1](5.png)

2. 有些组件用 `element ui` 的，但是有些属性我们想要，但是没有，比如以下这个`颜色选择器`

  ![1](6.png)

  我们之前是基于 `element ui` 的 `color-picker` 又做了重新封装和扩展，增加了左下角颜色预览显示，增加了隐藏时候emit事件。才满足我们的业务需求。

3. 主题方面，`element ui` 他有自己的主题定制，这一点非常棒，但是在使用过程中，我们发现可定制点还是少了，或者说有些定制我只希望在 button 组件上应用，结果它这个参数直接应用到所有组件上，这不是我想要的。 当然这也不能怪它，毕竟它已经能满足绝大部分 vue 项目的 ui 组件需求。但是比较遗憾，随着我们的项目规模越来越大，主题定制方面要求越来越细，`element ui` 已经满足不了了，我们只能在项目里面，针对 `element ui` 对应组件的 css 样式进行重写来达到我们的目的，但是这样子非常的被动

基于以上几个原因，我们萌生出了要基于现有的 `element ui` 的基础上，重新定制了我们自己的 ui 组件，才有了现在的 `air-ui`。

## air-ui 的优点
经过我一个月左右的吭哧吭哧的开发，`air-ui` 总算完成了。 当然中间有很多的代码几乎完全借鉴了 `element ui`， 包括绝大部分组件的逻辑，css 样式，util 方式，甚至包括 demo， 几乎全部借鉴了 `element ui` 里面的代码，因为就像之前说的，`element ui` 现有的组件真的写的非常好了，不太需要我去做太多的修改，只要移植过来就行了，我不可能为了表现出自己特立独行很牛逼，全部都要自己写，一定要跟别人不一样，结果自己累的要死要死的， 结果还没有人家写的好。事实上要不是大量的组件业务逻辑，样式，demo 都借鉴 `element ui` 组件，我也不可能在一个月把这个搞定， 当然如果全部都抄 `element ui` 的，那就真的没意思，也就不会有这个系列了。事实上在通过读取 `element ui` 项目源码的过程中，还是发现了一些我觉得做的不那么好的地方，而 `air-ui` 可以优化的。 比如以下几点：

1. 目录结构，`element ui` 的目录结构其实比较乱，可能是比较早期的项目吧，那时候还没有用 `vue-cli` 的目录结构，而 `air-ui` 就是基于 `vue-cli` 的目录结构来创建，可以说目录结构一看便知。
2. 文档方面，`element ui` 的文档其实做的还不错，说白了就是这个站点 https://element.eleme.io/ ， 但是我们不需要我们的文档也写的这么复杂，而且在读源码的过程中，我也觉得这个文档的项目也很复杂，而 `air-ui` 是基于 `vuepress`, 可以实现跟 `element.eleme.io` 差不多的效果，但是复杂程度就很少。
3. 引用路径问题，通过看源码，我发现 `element ui` 文档组件和方法的引用，有时候用相对路径，有时候用绝对路径引用，当然这个是有原因的，而且我也知道为啥它要这么做，这一点后面我会提到，但是在做 `air-ui` 的时候，我把这一点合理的避免了，所有的路径引用全部用相对路径就可以了。从可读性来看，会比 `element ui` 来的好。
4. 主题定制方面，既然之前是因为主题定制不够给力，才想要新写一个ui组件的，所以主题定制方法也跟 `element ui` 有点差别，更适合我们项目。这一点后面也会说道。
5. 多语言方面，多语言肯定是必须的，但是在多语言的处理上，`air-ui` 也有进行了一些优化，会比 `element ui` 的结构更好。

当然除了以上几点，还会有一些小的优化，比如  eslint， 构建方面等等，后面都会提到。

## air-ui 的引入方式
首先要说明的一点就是 `air-ui` 并没有开源，所以 npmjs 上面的那个 `air-ui` 不是我做的这个： 另一个 [air-ui](https://www.npmjs.com/package/air-ui)。

完整来说，`air-ui`的引入方式其实有 3 个:
### 1.完整引入
在`main.js`文件下添加如下配置:
```js
import Vue from 'vue';
import AirUI from 'air-ui'
import 'air-ui/lib/styles/index.css'
import App from './App.vue';

Vue.use(AirUI);

new Vue({
  render: h => h(App)
}).$mount('#app');
```
以上代码便完成了 `air-ui` 的引入。需要注意的是，样式文件需要单独引入。

### 2.部分引入(但是全部加载)
如果你只希望引入部分组件，比如 `Button` 和 `Row`，那么需要在 `main.js` 中写入以下内容：
```js
import Vue from 'vue';
import { Button, Row } from 'air-ui';
import 'air-ui/lib/styles/index.css'
import App from './App.vue';

Vue.use(Button)
Vue.use(Row)

new Vue({
  el: '#app',
  render: h => h(App)
});
```
### 3.部分组件按需加载
或者你也可以直接引用单独组件的 js 和 css：
```js
import Vue from 'vue';
import Button from 'air-ui/lib/button';
import Row from 'air-ui/lib/row';
import 'air-ui/lib/styles/button.css';
import 'air-ui/lib/styles/row.css';
import App from './App.vue';

Vue.use(Button);
Vue.use(Row);

new Vue({
  el: '#app',
  render: h => h(App)
});
```
第二种和第一种其实没啥差别，都是全部加载 `air-ui.common.js`, 不会省打包体积。只能算第一种的另一种语法糖的变种。为啥跟 `element-ui` 的按需加载不一样，具体可以看 {% post_link air-ui-8 %}

第三种才是真正可以省打包体积的方式。

## 暂不开源
目前 `air-ui` 是放在公司搭建私有的 gitlab 代码库的，并没有开源，一方面是因为新加的一些组件和新的定制主题都含有公司产品的特点，对其他人来说，并没有太多可借鉴的东西，另一方面也是大部分的组件都是从 `element ui` 那边挪过来的，对于其他人来说，直接用 `element ui` 效果更好。

当然这套组件也只是适用于 pc 端，不适合移动端，因为我们现在的大部分站点项目都是用于 pc 端的。 所以并没有专门去开发移动端的 ui 组件。因为市面上的第三方的移动端的 vue ui 组件已经够用了，没有必要干啥都要重造车轮。

## 写 blog 的初衷
接下来就围绕着我是怎么开发 `air-ui` 组件的，给大家讲讲我的思路和遇到的一些坑，甚至有些坑到现在都还没有完美的解决。 事实上我写得非常细，甚至都有点啰嗦了，再加上很多篇，每一篇的篇幅也不少，所以估计加起来都可以写成一本书了，哈哈。 当然之所以写的比较细的原因，也是因为我想把那段时间的思考，入坑，爬坑的过程都写清楚，毕竟每一次入坑到爬坑的过程都是一次宝贵的经验。毕竟对于程序员来说，解决问题的能力才是最重要的。

事实上之所以写这个系列是因为网上虽然有一些怎么教人写 ui 组件的教程，但是都是浅尝而止，做出来的东西根本没有办法达到大型线上项目可以用的地步，这也导致我几乎是啃着 `element ui` 源码来重写`air-ui` 组件的，虽然中间也借鉴了其他的一些成熟的开源 ui 项目的方式，但是大部分还是借鉴 `element ui`。

而写这个系列文章的原因，也是因为我觉得一旦项目到了一个规模，单独的第三方 ui 组件绝对是满足不了的，为了更好的维护和扩展，很多时候往往需要自己再建一套更符合自己项目风格和规范的 ui 组件。 那时候希望这些文章能对这群人达到一些帮助，少走点弯路， 奥利给~~~。

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















