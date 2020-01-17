---
title: 自建vue组件 air-ui (12) -- 国际化机制
date: 2020-01-04 18:38:55
tags: js
categories: 
- 前端相关
- 自建vue组件 air-ui
---
## 前言
`air-ui` 组件库的某些组件也会用到一些提示语，除了允许用传参来自定义之外，肯定会有预设值。 比如 `air-select` 组件的时候，可以允许搜索的，

![1](1.png)

如果搜索不到，会有一个默认提示： `无匹配数据`， 这个就是这个组件的其中一个提示语预设值。而且这个预设值其实是中文。接下来我们看看是怎么实现的？
<!--more-->
## 组件接入语言文件
有些组件的交互会用到一些预设的提示语，比如 `select`, `data-picker`, `color-picker`, `messagebox` 这些组件。我们以 `select` 组件为例：
> 这个组件的代码不少，这边不全部列出来，如果想看源码的，可以参照 `element-ui` 组件的 `select.vue` 代码, 两者的代码除了引用路径不太一样之外(`element-ui`有些资源的引入是绝对路径，而`air-ui`全部用相对路径引入)，其他几乎都一样

```javascript
<script type="text/babel">
import Locale from '../../../../src/mixins/locale';
...
import { t } from '../../../../src/locale';
...
export default {
  ...
  computed: {
    ...
    emptyText() {
      if (this.loading) {
        return this.loadingText || this.t('air.select.loading');
      } else {
        if (this.remote && this.query === '' && this.options.length === 0) return false;
        if (this.filterable && this.query && this.options.length > 0 && this.filteredOptionsCount === 0) {
          return this.noMatchText || this.t('air.select.noMatch');
        }
        if (this.options.length === 0) {
          return this.noDataText || this.t('air.select.noData');
        }
      }
      return null;
    },
  }
}
</script>
```
他有一个计算属性 `emptyText`, 这个属性经常被使用，每次使用的时候，都会去判断各种状态：
- 如果是 loading 状态，就会显示 loading 提示语，如果没有额外对这个提示语进行传参，就使用默认的这个词条: `air.select.loading`
- 如果是允许搜索的，就匹配选项值，如果匹配不到，并且没有额外对这个提示语进行传参，就使用默认的这个词条: `air.select.noMatch`
- 如果没有选项值，并且没有额外对这个提示语进行传参，就使用默认的这个词条: `air.select.noData`

我们可以看到默认预设值的提取是通过 `t()` 这个方法来处理的，我们看下这个方法，这个方法在 `src/mixins/local.js` 这个文件里面有定义(这个文件在上面有引入)：
```javascript
import { t } from '../locale';

export default {
  methods: {
    t(...args) {
      return t.apply(this, args);
    }
  }
};
```
他返回了一个 t 的方法，并且因为是用 mix 的方式引入的，所以实例里面就可以使用这种方式调用: `this.t(xxx)`:
```javascript
mixins: [..., Locale, ...],
```
而这个 t 的方法，其实是从 `src/locale/index.js` 这个文件导出的：
```javascript
import defaultLang from '../lang/zh-CN';
import Vue from 'vue';
import deepmerge from 'deepmerge';
import Format from './format';

const format = Format(Vue);
let lang = defaultLang;
let merged = false;
let i18nHandler = function () {
  // 如果存在 vue-i18n 组件，那么就将本ui组件的多语言文件合并到 i18n 组件对应的语言文件中，这样子后面要用本ui组件的词条的时候，就直接调用 i18n 组件的方法就行了
  const vuei18n = Object.getPrototypeOf(this || Vue).$t;
  // 如果是 vue-i18n 组件，并且有 Vue.locale 这个方法，那么就直接内置兼容，其实就是用他原有的 lang 对象再跟 air-ui 的 lang 对象进行覆盖合并
  if (typeof vuei18n === 'function' && !!Vue.locale) {
    if (!merged) {
      merged = true;
      Vue.locale(
        Vue.config.lang,
        deepmerge(lang, Vue.locale(Vue.config.lang) || {}, {clone: true})
      );
    }
    return vuei18n.apply(this, arguments);
  }
};

export const t = function (path, options) {
  // 先从 i18n 里面找，找不到再从 本ui组件的语言文件中查找
  let value = i18nHandler.apply(this, arguments);
  if (value !== null && value !== undefined) return value;

  const array = path.split('.');
  let current = lang;

  for (let i = 0, j = array.length; i < j; i++) {
    const property = array[i];
    value = current[property];
    if (i === j - 1) return format(value, options);
    if (!value) return '';
    current = value;
  }
  return '';
};

export const use = function (l) {
  lang = l || lang;
};

export const i18n = function (fn) {
  i18nHandler = fn || i18nHandler;
};

export default {use, t, i18n};
```
这个文件的逻辑也很简单:
1. 首先导入默认的中文的语言包文件，得到 `lang` 这个 map 对象
2. 我们先看 t 方法，这个方法就是根据传入的多语言的key，先通过 `i18nHandler` 这个方法看能不能找到对应的值，如果还找不到， 那么就从默认语言包的 `lang` 这个map对象里面得到对应的 value 并返回。

接下来我们看下，这个默认的中文的语言包里面是怎么存的 `src/lang/zh-CN.js`: 
```javascript
export default {
  air: {
    colorpicker: {
      confirm: '确定',
      clear: '清空'
    },
    datepicker: {
      now: '此刻',
      today: '今天',
      cancel: '取消',
      clear: '清空',
      confirm: '确定',
      selectDate: '选择日期',
      selectTime: '选择时间',
      startDate: '开始日期',
      startTime: '开始时间',
      endDate: '结束日期',
      endTime: '结束时间',
      prevYear: '前一年',
      nextYear: '后一年',
      prevMonth: '上个月',
      nextMonth: '下个月',
      year: '年',
      month1: '1 月',
      month2: '2 月',
      month3: '3 月',
      month4: '4 月',
      month5: '5 月',
      month6: '6 月',
      month7: '7 月',
      month8: '8 月',
      month9: '9 月',
      month10: '10 月',
      month11: '11 月',
      month12: '12 月',
      // week: '周次',
      weeks: {
        sun: '日',
        mon: '一',
        tue: '二',
        wed: '三',
        thu: '四',
        fri: '五',
        sat: '六'
      },
      months: {
        jan: '一月',
        feb: '二月',
        mar: '三月',
        apr: '四月',
        may: '五月',
        jun: '六月',
        jul: '七月',
        aug: '八月',
        sep: '九月',
        oct: '十月',
        nov: '十一月',
        dec: '十二月'
      }
    },
    select: {
      loading: '加载中',
      noMatch: '无匹配数据',
      noData: '无数据',
      placeholder: '请选择'
    },
    cascader: {
      noMatch: '无匹配数据',
      loading: '加载中',
      placeholder: '请选择',
      noData: '暂无数据'
    },
    pagination: {
      goto: '前往',
      pagesize: '条/页',
      total: '共 {total} 条',
      pageClassifier: '页'
    },
    messagebox: {
      title: '提示',
      confirm: '确定',
      cancel: '取消',
      error: '输入的数据不合法!'
    },
    upload: {
      deleteTip: '按 delete 键可删除',
      delete: '删除',
      preview: '查看图片',
      continue: '继续上传'
    },
    table: {
      emptyText: '暂无数据',
      confirmFilter: '筛选',
      resetFilter: '重置',
      clearFilter: '全部',
      sumText: '合计'
    },
    tree: {
      emptyText: '暂无数据'
    },
    transfer: {
      noMatch: '无匹配数据',
      noData: '无数据',
      titles: ['列表 1', '列表 2'],
      filterPlaceholder: '请输入搜索内容',
      noCheckedFormat: '共 {total} 项',
      hasCheckedFormat: '已选 {checked}/{total} 项'
    },
    image: {
      error: '加载失败'
    },
    pageHeader: {
      title: '返回'
    }
  }
};
```
所以如果 key 是 `air.select.noMatch` 那么就会找到 air 对象中的 select 对象中的 noMatch 这个key，然后得到对应的 value，并且返回。 这样就可以使用语言包的语言了。

## 组件使用多语言
我们已经知道怎么在组件里面接入语言文件了，并使用语言文件的词条了。但是还有一个问题，就是 `air-ui` 默认加载的语言文件是`中文`,那么我怎么加载其他的多语言文件？？ 所有的多语言文件都放在了 `src/lang` 这个目录里面。

这边就涉及到上面所列文件 `src/locale/index.js` 文件里面的 use 方法了:
```javascript
export const use = function (l) {
  lang = l || lang;
};
```
我们知道其实 t 方法在获取对应 key 的 value 的时候，是从 lang 这个对象获取的，而 lang 初始化的时候，就是中文语言包对象赋值给它的。所以我们要用其他语言的语言包，比如 en，只需要将 en 的语言包导出并且调用 use 方法，重新覆盖 lang 对象就行了：
```javascript
import locale from './locale/index'
import en from './lang/en'

locale.use(en)
```
这样子，就会用 en 的语言包了，截图如下：

![1](2.png)

## 结合到 air-ui
虽然我们可以通过以上的语法来实现用其他语言的语言包，但是毕竟这种写法不够优雅，因此我们可以将这个语法嵌入到 `air-ui` 初始化中，所以我们在 `components/index.js` 改一下，在原来的基础上加上：
```javascript
import locale from '../../src/locale'

const install = function (Vue, opts = {}) {
  // 添加语言参数，让其初始化组件的时候，可以传进去其他的语言
  locale.use(opts.locale);
  locale.i18n(opts.i18n);
  ...
}

module.exports = {
  locale: locale,
  i18n: locale.i18n,
  install,
  ...其他组件...
}
```
可以看到除了将 locale 对象引入之外，还改了两个地方：
1. install 方法增加了对 use 和 i18n 的处理。然后直接从 opts 参数可以得到传进来的多语言对象和 i18n 的获取方法
2. 导出的时候，也单独导出 locale 对象和里面的 i18n方法，以便后面单独引用组件的时候，会用到多语言

所以就会引用就会变成这样子:
```javascript
import AirUI from './components/index'
import enLng from './lang/en'

Vue.use(AirUI, {'locale': enLng})
```
这样语法上更好看。

## 兼容 i18n 插件
`src/locale/index.js` 导出来的三个方法:
1. `t` 根据 key 获取 词条
2. `use` 更换新的语言包文件
3. `i18n` 自定义获取的方法

其中 1 和 2 都讲过了。接下来讲讲第三个方法 `i18n`, 也就是怎么兼容项目中已经存在的 `i18n` 插件或者自定义自己获取多语言 value 的方法？？？

有时候项目中有自己的多语言文件，然后 `air-ui` 又有自己的多语言文件，如果对于 `air-ui` 本身自带的多语言文件中的词条也要调整的话，那么就要需要同时维护两份多语言文件。而且其中一份还是第三方库自带的多语言文件。这样子就会变得很麻烦。 所以 `air-ui` 提供了一种方式，就是允许你通过在引用 `air-ui` 的时候, 可以自己定制获取多语言的语法以及对应的多语言 map， 具体的逻辑代码如下:
```javascript
let i18nHandler = function () {
  // 如果存在 vue-i18n 组件，那么就将本ui组件的多语言文件合并到 i18n 组件对应的语言文件中，这样子后面要用本ui组件的词条的时候，就直接调用 i18n 组件的方法就行了
  const vuei18n = Object.getPrototypeOf(this || Vue).$t;
  // 将 ui 的多语言列表合并到 i18n 的对应的多语言的文件中
  if (typeof vuei18n === 'function' && !!Vue.locale) {
    if (!merged) {
      merged = true;
      Vue.locale(
        Vue.config.lang,
        deepmerge(lang, Vue.locale(Vue.config.lang) || {}, {clone: true})
      );
    }
    return vuei18n.apply(this, arguments);
  }
};

export const i18n = function (fn) {
  i18nHandler = fn || i18nHandler;
};
```
因为在 t 方法的时候，会先去通过 i18nHandler 方法去获取 value，只有当获取不到的时候，才用自带的语言包来处理。 所以我们只需要对 i18nHandler 方法进行赋值就可以自定义获取多语言的方法了，比如这样子：
```javascript
const customLangMap = {
  'air.select.noMatch': '自定义的找不到'
}
Vue.use(AirUI, {
  i18n: key => customLangMap[key]
});
```
这样的效果就是调用 t 方法的时候，就会使用我们自定义的 i18n 的方法，然后得到值，只有当返回值为 null 或者 undefined 的时候，才会用原来的方式再去获取一次。所以结果如下:

![1](3.png)

可以说也是非常的方便。

## 兼容 vue-i18n 插件的情况
上面通过定制 i18n 方法可以随便定制获取的方法。但是其实 vue 项目中用到的最多的 i18n 组件还是属于 [vue-18n](https://github.com/kazupon/vue-i18n), 所以 `air-ui` 也是有对 i18n 的不同的版本有进行不同程度的兼容

### vue-i18n@5.0.3
如果你用的是 `vue-i18n` 的 `5.0.3` 这个版本，就是有 `Vue.locale` 这个方法的。那么就不太需要对 i18n 方法进行定制了。因为我们有在 i18nHandler 方法中内置了对这个版本的兼容了：
```javascript
let i18nHandler = function () {
  // 如果存在 vue-i18n 组件，那么就将本ui组件的多语言文件合并到 i18n 组件对应的语言文件中，这样子后面要用本ui组件的词条的时候，就直接调用 i18n 组件的方法就行了
  const vuei18n = Object.getPrototypeOf(this || Vue).$t;
  // 如果是 vue-i18n 组件，并且有 Vue.locale 这个方法，那么就直接内置兼容，其实就是用他原有的 lang 对象再跟 air-ui 的 lang 对象进行覆盖合并
  if (typeof vuei18n === 'function' && !!Vue.locale) {
    if (!merged) {
      merged = true;
      Vue.locale(
        Vue.config.lang,
        deepmerge(lang, Vue.locale(Vue.config.lang) || {}, {clone: true})
      );
    }
    return vuei18n.apply(this, arguments);
  }
};
```
所以调用直接变成这样子：
```javascript
import Vue from 'vue'
import VueI18n from 'vue-i18n'
import AirUI from 'air-ui'
import enLocale from 'air-ui/lib/lang/en'
import zhLocale from 'air-ui/lib/lang/zh-CN'

Vue.use(VueI18n)
Vue.use(AirUI)

Vue.config.lang = 'zh-cn'
Vue.locale('zh-cn', zhLocale)
Vue.locale('en', enLocale)
```

### vue-i18n@6.x
如果是最新版本的 6.x，那么是不兼容的，还是要调用 i18n 去自定义，比如:
```javascript
import Vue from 'vue'
import AirUI from 'air-ui'
import VueI18n from 'vue-i18n'
import enLocale from 'air-ui/lib/lang/en'
import zhLocale from 'air-ui/lib/lang/zh-CN'

Vue.use(VueI18n)

const messages = {
  en: {
    message: 'hello',
    ...enLocale // 或者用 Object.assign({ message: 'hello' }, enLocale)
  },
  zh: {
    message: '你好',
    ...zhLocale // 或者用 Object.assign({ message: '你好' }, zhLocale)
  }
}
// Create VueI18n instance with options
const i18n = new VueI18n({
  locale: 'en', // set locale
  messages, // set locale messages
})

Vue.use(AirUI, {
  i18n: (key, value) => i18n.t(key, value)
})

new Vue({ i18n }).$mount('#app')
```

## 总结
本节主要是讲 `air-ui` 怎么接入 i18n 多语言机制的。 但是还存在以下几个问题:
1. 我们现在是有默认引入 `zh-CN.js` 这个文件的，这样会导致我们后面打包出来的 `air-ui.common.js` 体积变大，我们要怎么将之抽出来？？
2. 组件单独打包出来的话，引入的时候，要怎么接入多语言机制？？

这个将在下一节来解决。

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
