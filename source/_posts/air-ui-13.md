---
title: 自建vue组件 air-ui (13) -- 国际化机制(进阶版)
date: 2020-01-06 17:16:08
tags: js
categories: 
- 前端相关
- 自建vue组件 air-ui
---
## 前言
通过上节 {% post_link air-ui-12 %} 我们已经知道怎么在 `air-ui` 进行国际化配置了。但是还遗留几个问题没有解决：
1. 我们现在是有默认引入 `zh-CN.js` 这个文件的，这样会导致我们后面打包出来的 `air-ui.common.js` 体积变大，我们要怎么将之抽出来？？
2. 组件单独打包出来的话，引入的时候，要怎么接入多语言机制？？

## 将语言包文件引用单独抽出来
我们知道在打 `air-ui.common.js` 包的时候，之所以会把语言包的 `zh-CN.js` 打进去是因为，只要入口文件有引用的话，那么默认就会打包进去，打包链是这样子的：
<!--more-->
![1](1.png)

```javascript
components/index.js -> components/select/index.js -> components/select/select.vue -> mixins/locale -> local/index.js -> lang/zh-CN.js
```
所以如果我们要把 `lang/zh-CN.js` 不打进去的方式，就是在 webpack 打包的时候，将这个文件忽略掉，不打进去，而是当做外部资源包去引用，`externals` 这个字段就是用来 webpack 打包的时候，将一些资源进行忽略 (具体整个构建，我后面会再开一节来讲，本节只要知道这个过程即可), 所以在打 `air-ui.common.js` 文件的时候，要将这个资源进行忽略(vue 相关的资源也要忽略，不要打进去)：
```javascript
const nodeExternals = require('webpack-node-externals');
```
```javascript
  // 这边将一些文件不打包进去，不然 common 的体积会变得很大，尤其是语言文件。 不过这样一来的后果就是，如果 common 不包含这些文件的话，那么这些文件就要单独打出来，而且路径还不能变， 而且一旦不打包进去，那么引用就要变成 绝对路径引用，不能再用
  externals: [Object.assign({
    vue: 'vue'
  }, {
    '../lang/zh-CN': 'air-ui/lib/lang/zh-CN'
  }), nodeExternals()],
```
这样子就会变成，当打包的时候找到这个 `../lang/zh-CN` 这个资源的时候， webpack 不会打包进去，而是换成对应的绝对引用地址 `air-ui/lib/lang/zh-CN` ，所以一旦我们这样处理之后，我们的 `lang` 目录在打 dist 组件包的时候，就要单独复制出来，让 `air-ui.common.js` 在查找语言包的时候，可以通过这个绝对路径来找到。 我们可以看到在打完包的 `air-ui.common.js` 关于语言包的引用就变成绝对路径了：

![1](2.png)

这样子打包的时候，语言包就会当做外部资源的绝对路径来引用了。就不会打进去了。

## element-ui 的做法
`element-ui` 也是将语言包抽出来，但是他抽的是 `src/locale/index.js` 这个文件，把这个文件当做外部资源，然后用绝对路径来引入：

![1](3.png)

但是这个资源不仅仅只是引入语言文件，还有其他的资源，比如 utils 里面的方法。 一旦这个资源不打包进去而是变成外部资源去引用，那么它引用其他依赖资源，也必须也要用绝对路径来引用
举个例子，比如 locale 是外部资源，然后要引用 utils 里面的方法，如果 utils 不也变成外部资源的话，而是当做内部资源打入 common.js 的话，就会导致 locale 引用不到了，所以用绝对路径相互依赖的情况就是所有的依赖全部变成外部资源，全部变成绝对路径引用了。而 `element-ui` 就是这样做的，他把除组件的其他资源全部变成外部资源，然后用绝对路径来引用了。

![1](4.png)

当然从逻辑上来说，这样并不会报错。也是可以的。但是会导致的一个现象就是，我有些组件在引用资源的时候，一会用相对路径，一会儿用绝对路径，事实上我当初看源码的时候，这一点就会疑惑。而且更奇葩的是哪怕在同一个目录下，也必须要用绝对路径引用，打包才不会出错。

所以我那时候在处理 `air-ui` 多语言打包的时候，就要吸取这个教训，代码里面全部用相对路径， 然后在打包的时候，只针对语言文件，当做外部资源来用绝对路径来引用。而其他的资源的引用，还是一样打进去 `air-ui.common.js` 中。这样子我一个是可以保证我在开发的时候，全部用相对路径来引用，而不需要去管打包的事情。另一个就是打完包的目录结构也比较好看。除了 `air-ui.common.js` 文件，就一个 lang 目录存放语言包文件：

![1](5.png)

## 如果引用绝对路径的写法会遇到什么坑
虽然我最后采取了全部走相对路径引用的方式。但是其实是有一个过程的，在早期在尝试的时候，还是参照 `element-ui` 的写法，在代码里面引用绝对路径的写法：

![1](6.png)

然后开发环境直接报错了:
```javascript
This dependency was not found:

* air-ui/src/lang/zh-CN in ./src/locale/index.js

To install it, you can run: npm install --save air-ui/src/lang/zh-CN
```
所以构建那边要处理一下， `webpack.base.conf.js` 要加一个配置，让全局路径有引用:

![1](7.png)

这样子构建就没有问题了。 当然因为`默认 @ 就是 src 目录`了，那么也可以写成
```javascript
import defaultLang from '@/lang/zh-CN';
```
这样也是可以的。 不过我倾向于用上面那种方式。

这样子测试环境就没有问题了。 接下来就处理 `dist` 打包的 `externals` 了， 因为只处理 `lang` 语言文件，所以要改成：
```javascript
const nodeExternals = require('webpack-node-externals');

// 这边将一些文件不打包进去，不然 common 的体积会变得很大，尤其是语言文件。 不过这样一来的后果就是，如果 common 不包含这些文件的话，那么这些文件就要单独打出来，而且路径还不能变， 而且一旦不打包进去，那么引用就要变成 绝对路径引用，不能再用
externals: [Object.assign({
  vue: 'vue'
}, {
  'air-ui/src/lang/zh-CN': 'air-ui/lib/lang/zh-CN'
}), nodeExternals()],
```
相当于就是把这个绝对路径，改成打包后的路径，接下来就开始打包： `yarn dist` 打包完成之后，我们再搜一下，`air-ui.common.js` 是否还有中文。 发现没有中文了，语言包彻底被抽离出来了。

但是我们发现  lang 文件夹并没有输出到 lib 目录中，所以这边还得处理一下，在打包的过程中，将 lang 目录也同样输出到 lib 目标目录：
```javascript
//============= 处理语言文件，挪到 lib 目录 start===========
gulp.task('copylang', function() {
  return gulp.src('./src/lang/**')
    .pipe(gulp.dest('./lib/lang'));
});
//============= 处理语言文件，挪到 lib 目录 end=============
```
然后添加到 dist 任务后面。  这样子 lang 目录就打包到 lib 目录了。 这样好像就没啥问题了。
结果后面发现在用 vuepress 的时候，会出现这个问题：

![1](8.png)

他找不到这个路径， 就算改成 `@` 的方式，也是不行？？ 尴尬了，他就只能用相对路径；
```javascript
import defaultLang from '../lang/zh-CN';
```
所以为了兼容这个 vuepress 的文档可以使用，我们只能在代码里面使用 相对路径， 然后在构建的时候，替换成绝对路径， 所以打包的时候，改成：
```javascript
externals: [Object.assign({
  vue: 'vue'
}, {
  '../lang/zh-CN': 'air-ui/lib/lang/zh-CN'
}), nodeExternals()],
```
因为是用正则匹配的，所以要保证代码里面没有 另一个的非语言的字符串叫做`../lang/zh-CN`， 不然也会被替换掉，这样就可以了。 既可以保证 vuepress 可以用，也可以让语言文件不会打包进去。

## 组件单独打包的时候，怎么引入多语言机制
我们已经知道在打 common 包的时候，已经可以将多语言抽出来了，并且当做外部资源使用绝对路径来引用。但是当我们组件单独打包并且引用的时候，我要怎么使用多语言，并且把多语言的文件也抽出来呢？

我们可以按照一下语法，实际引用的时候应该是这样子的：
```javascript
import Vue from 'vue';
import Select from 'air-ui/lib/select';
import Row from 'air-ui/lib/row';
import 'air-ui/lib/styles/select.css';
import 'air-ui/lib/styles/row.css';
import App from './App.vue';

Vue.use(Button);
Vue.use(Row);
```
对于这种情况，要设置多语言的话，肯定要使用 locale 对象(不然这么用 use 和 i18n 方法)，但是之前 locale 对象都是加载到 `air-ui.common.js` 文件里面的，如果需要按需加载的话，那么这个对象还得重新再导出来, 所以在 `webpack.component.js` 也要把 locale 对象导出来，因此要在 `components.json` 加上这一行：

```javascript
"locale": "./src/locale/index.js",
```

![1](9.png)

这样子导出的组件对象就会包含 locale 这个对象了，那我们就可以通过这个对象去设置多语言了：

![1](10.png)

这样就出来了。 后面导入这个即可：

```javascript
import Vue from 'vue'

import Select from 'air-ui/lib/select';
import Option from 'air-ui/lib/option';
import Row from 'air-ui/lib/row';
import locale from 'air-ui/lib/locale';
import 'air-ui/lib/styles/select.css'
import 'air-ui/lib/styles/row.css'
import 'air-ui/lib/styles/option.css'
import lang from 'air-ui/lib/lang/en'

locale.use(lang)
Vue.component(Row.name, Row)
Vue.component(Select.name, Select)
Vue.component(Option.name, Option)
```
这样子就是按需加载，并导入对应的多语言了。  但是这样发现并不行， 这个是因为 select 里面的 local 是相对路径的，并且在构建 dist 的时候，并没有抽出来，

导致读的 locale 对象并不是后面引用的，而是 select.js 里面的 locale：

![1](11.png)

所以要改成，如果要单独构建组件的时候，要把里面引用的 local，也要换成绝对路径，所以在 `webpack.componnent.js` 要加这一行：
```javascript
  // 这边将一些文件不打包进去，不然 common 的体积会变得很大，尤其是语言文件。 不过这样一来的后果就是，如果 common 不包含这些文件的话，那么这些文件就要单独打出来，而且路径还不能变， 而且一旦不打包进去，那么引用就要变成 绝对路径引用，不能再用
  externals: [Object.assign({
    vue: 'vue'
  }, {
    '../lang/zh-CN': 'air-ui/lib/lang/zh-CN',
    // 这边注意，只有在打包单独组件的时候，才需要进行 locale 的路径替换，因为单独加载组件，也需要多语言的支持，如果是打包common，那么就不需要，因为都集成了
    '../../../../src/locale': 'air-ui/lib/locale',
    '../locale': 'air-ui/lib/locale',
  }), nodeExternals()],
```
这样子单独构建 select 的时候，引用的 locale ，就会变成打包后的 locale 了。这样子对象就能匹配上了。

这边要特别注意，我们只在 `webpack.components.js` 打包单独组件的时候，才需要对 locale 对象进行抽离，如果是打包`air-ui.common.js` 是不需要的，所以不要搞混了。

## 国际化的几种方式
最后我们来总结一下，国际化的几种引入方式:
### 完整引入
```javascript
// 完整引入 AirUI
import Vue from 'vue'
import AirUI from 'air-ui'
import locale from 'air-ui/lib/lang/en'

Vue.use(AirUI, { locale })
```
### 部分引入(全部加载)
```javascript
import Vue from 'vue'
import { Row, Select, Option, locale } from 'air-ui'
import 'air-ui/lib/styles/index.css'
import lang from 'air-ui/lib/lang/en'

locale.use(lang)
Vue.component(Row.name, Row)
Vue.component(Select.name, Select)
Vue.component(Option.name, Option)
```
这边还需要注意一个细节，因为 locale 是单独被导出来的，因此在 `components/index.js` 这个文件在导出的时候，也要将 locale 对象导出来:
```javascript
import locale from '../../src/locale'

module.exports = {
  locale: locale,
  ...
}
```
这样才能在这种方式引用到。
### 按需加载并引入
```javascript
// 按需加载并引入 AirUI
import Vue from 'vue'
import Select from 'air-ui/lib/select';
import Option from 'air-ui/lib/option';
import Row from 'air-ui/lib/row';
import locale from 'air-ui/lib/locale';
import 'air-ui/lib/styles/select.css'
import 'air-ui/lib/styles/row.css'
import 'air-ui/lib/styles/option.css'
import lang from 'air-ui/lib/lang/en'

locale.use(lang)
Vue.component(Row.name, Row)
Vue.component(Select.name, Select)
Vue.component(Option.name, Option)
```
### 替换默认的中文语言包

如果使用其它语言，默认情况下中文语言包依旧是被引入的，可以使用 webpack 的 `NormalModuleReplacementPlugin` 替换默认语言包：
```javascript
{
  plugins: [
    new webpack.NormalModuleReplacementPlugin(/air-ui[\/\\]lib[\/\\]lang[\/\\]zh-CN/, 'air-ui/lib/lang/en')
  ]
}
```
## 总结
本节主要讲打包的时候，多语言文件的抽离和处理。包括打 common 文件和打单独组件包的时候是怎么做的，以及跟 `element-ui` 的差别在哪里。 本节已经涉及到打包构建的一些情况了。下一节我们来详细讲一下 `air-ui` 的打包构建，尤其是打 dist 包是怎么处理的。

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



