---
title: 自建vue组件 air-ui (3) -- css 开发规范
date: 2019-12-05 11:38:43
tags: js
categories: 
- 前端相关
- 自建vue组件 air-ui
---
## 前言
之前分析过了 `element-ui` 的项目({% post_link air-ui-2 %})，包括目录结构，构建以及项目开发的整体思路，今天打算还是以 `element-ui`这个项目的源码为主，我们来聊一聊开发一个ui组件的时候，我们应该怎么设计 css 的开发规范。 这一点我觉得 `element-ui` 它们封装的非常好，该抽象的有抽象，该封装的有封装。所以后续我开发 `air-ui` 直接把这一套挪过去用了。

## 什么是 BEM
关于 `BEM` 的更详细的介绍，可以看它的官网: [BEM官网](http://getbem.com/)

`BEM` 是 `Block`（块） `Element`（元素） `Modifier`（修饰器）的简称, 它可以帮助你创建出可以复用的前端组件和前端代码, 他有以下三个特点：

- `重用性` 不同方式组织独立的块，并智能地重用它们，可以减少必须维护的CSS代码量。通过一系列风格指南，您可以构建一个块库，使您的CSS超级有效。
- `单元性` 块的样式从来不依赖同页面其它的元素，所以你永远不会遇到级联问题。您还可以将完成的项目中的块转移到新项目中。
- `结构化` BEM方法可以使得你的CSS代码结构性很好，从而更加容易理解。

使用BEM规范来命名CSS，组织HTML中选择器的结构，利于CSS代码的维护，使得代码结构更清晰。 当然也有弊端，比如名字会稍长, 而且因为大部分都是只有一层结构，还要注意样式覆盖问题
<!--more-->
## 如何使用 BEM
1. 一个独立的（语义上或视觉上），可以复用而不依赖其它组件的部分，可作为一个块（Block）
2. 属于块的某部分，可作为一个元素（Element）
3. 用于修饰块或元素，体现出外形行为状态等特征的，可作为一个修饰器（Modifier）

## element ui 的使用
接下来我要讲的就是 `element-ui` 如何利用sass，编写具有可读性和可维护性的BEM规则的css代码。
### 命名空间定义
在 `package/theme-chalk/src/mixins/config.scss` 这个文件中，有一个针对 BEM 的规范定义：
```scss
$namespace: 'el';
$element-separator: '__';
$modifier-separator: '--';
$state-prefix: 'is-';
```

1. 双下划线 `__` 来作为块和元素的间隔, 比如 `el-form-item__content`
2. 双中划线 `--` 来作为块和修饰器 或 元素和修饰器 的间隔, 比如 `el-form--inline`
3. 中划线 `-` 来作为 块|元素|修饰器 名称中多个单词的间隔
4. 状态的前缀用 `is`, 比如是否选中，就是 `is-checked`

命名空间 `$namespace` 其实就是 BEM 的前缀，也就是说，后面我在做 `air-ui` 的时候，直接设置`$namespace: 'air';` 就可以了。

### BEM 在 sass 中的定义
在 `packages/theme-chalk/src/mixins/mixins.scss` 有对 BEM 的宏定义：
#### block 的定义
```scss
@mixin b($block) {
  $B: $namespace+'-'+$block !global;

  .#{$B} {
    @content;
  }
}
```
逻辑看起来不难理解，结合上面的 `config.scss` 的定义，这时就能很清楚的看到`block`的生成就是基于BEM规范中，块是设计或布局的一部分，具有一定的意义，利用`命名空间el`加上中划线，以及传入的`block`的名字，构建出`block`的样式，例如`alert`组件，在渲染完成后是`el-alert`，体现出它的唯一性。而在块的内部，再来编写跟这个块关联的其他样式代码。

这边注意两个细节：
 - `!global` 表示变量提升，将局部变量 `B` 提升为全局变量，这样子在其他函数体内也能访问到此变量 (这样子 b 里面的 e 才能访问到 b 的选择器 `B`)
 - `#{}` 表示插值，可以通过 `#{}` 插值语法在选择器和属性名中使用 `SassScript` 变量, 比如原先是这样子：
 
 
 ```scss
$name: foo;
$attr: border;
p.#{$name} {
      #{$attr}-color: blue;
}
```
经过编译之后，就会变成：
```scss
p.foo {
  border-color: blue;
}
```

#### element 的定义
```scss
@mixin e($element) {
  $E: $element !global;
  $selector: &;
  $currentSelector: "";
  @each $unit in $element {
    $currentSelector: #{$currentSelector + "." + $B + $element-separator + $unit + ","};
  }

  @if hitAllSpecialNestRule($selector) {
    @at-root {
      #{$selector} {
        #{$currentSelector} {
          @content;
        }
      }
    }
  } @else {
    @at-root {
      #{$currentSelector} {
        @content;
      }
    }
  }
}
```
上面的 `each` 很好理解，因为有可能传进去的多个参数，比如在 `table` 中，就有这种写法：
```scss
@include b(table) {
  position: relative;
  // something table content
  @include e((header-wrapper, body-wrapper, footer-wrapper)) {
    width: 100%;
  }
}
```
编译成 css 就是：
```scss
.el-table__body-wrapper, .el-table__footer-wrapper, .el-table__header-wrapper {
    width: 100%
}
```
当然这边也要注意一个细节，就是 `@at-root` 将父级选择器直接暴力的改成根选择器。

接下来我们将看一下这个 `if` 和 `else` 分支,为什么会出现 `hitAllSpecialNestRule` 函数判断的分支，原因是在修饰符或者其他mixin中(比如 伪类或者状态) 嵌套一个元素element，会出现修饰符在前，而元素在后的编译结果，所以我们用 `hitAllSpecialNestRule` 函数来判断是否存在特殊的嵌套，如果存在的话，就将 element 元素嵌套在里面，如果不存在，则原样输出(改成根选择器输出)

接下来看一下 `hitAllSpecialNestRule` 的定义, 在 `packages/theme-chalk/src/mixins/function.scss` 中
```scss
@import "config";

/* BEM support Func
 -------------------------- */
@function selectorToString($selector) {
  $selector: inspect($selector);
  $selector: str-slice($selector, 2, -2);
  @return $selector;
}

@function containsModifier($selector) {
  $selector: selectorToString($selector);
  @if str-index($selector, $modifier-separator) {
    @return true;
  } @else {
    @return false;
  }
}

@function containWhenFlag($selector) {
  $selector: selectorToString($selector);
  @if str-index($selector, '.' + $state-prefix) {
    @return true
  } @else {
    @return false
  }
}

@function containPseudoClass($selector) {
  $selector: selectorToString($selector);
  @if str-index($selector, ':') {
    @return true
  } @else {
    @return false
  }
}

@function hitAllSpecialNestRule($selector) {
  @return containsModifier($selector) or containWhenFlag($selector) or containPseudoClass($selector);
}

```
第一个函数 `selectorToString` ，就是将我们的选择器转换成一个字符串，而接下来的三个函数，分别判断了:
1. 是否存在 修饰符, 就是 `m`, 通过上级选择器是否含有标记为 `m` 的 `--` 子串
2. 是否存在 flag，比如 `.is-checked`,通过上级选择器是否含有 `is-` 子串
3. 是否存在伪类，比如 `:hover`，通过上级选择器是否含有 `:` 子串

最后综合在一起返回结果，避免嵌套。

##### m 嵌套 e 的情况
比如在 `message-box.scss` 当为居中布局的时候，就会出现这种情况：
```scss
@include b(message-box) {
  //...
  // centerAlign 布局
  @include m(center) {
    //...
    @include e(header) {
      padding-top: 30px;
    }
  }
}
```
编译成 css 就是：
```scss
.el-message-box--center .el-message-box__header {
    padding-top: 30px
}
```
可以看到有嵌套进去了
##### 包含状态 b 嵌套 e
比如在 `step.scss` 当为横向展示的时候，就会出现这种情况：
```scss
@include b(step) {
  //...
  @include when(horizontal) {
    //...
    @include e(line) {
      height: 2px;
      top: 11px;
      left: 0;
      right: 0;
    }
  }
}
```
编译成 css 就是：
```scss
.el-step.is-horizontal .el-step__line {
    height: 2px;
    top: 11px;
    left: 0;
    right: 0
}
```
其中 `when` 也是一个自定义的宏，主要是用来添加状态：
```scss
@mixin when($state) {
  @at-root {
    &.#{$state-prefix + $state} {
      @content;
    }
  }
}
```

##### 包含伪类 b 嵌套 e
比如还是在 `step.scss` 也会出现这种情况：
```scss
@include b(step) {
  //...
  @include pseudo(last-of-type) {
    @include e(line) {
      display: none;
    }
  }
}
```
编译成 css 就是:
```scss
.el-step:last-of-type .el-step__line {
    display: none
}
```
其中 `pseudo` 也是一个自定义的宏，主要是用来添加伪类状态：
```scss
@mixin pseudo($pseudo) {
  @at-root #{&}#{':#{$pseudo}'} {
    @content
  }
}
```
#### modifier 的定义
相对于 e 的定义， m 的定义就会比较好懂了：
```scss
@mixin m($modifier) {
  $selector: &;
  $currentSelector: "";
  @each $unit in $modifier {
    $currentSelector: #{$currentSelector + & + $modifier-separator + $unit + ","};
  }

  @at-root {
    #{$currentSelector} {
      @content;
    }
  }
}
```
这边要注意一个细节，在拼接 `currentSelector` 字符串时，使用了`$`父级选择器，而没有使用`全局变量B` + `全局变量E` 来拼接，因为结构不一定是`B-E-M`,有可能是`B-M`。

同时也有存在一次定义多个 m 的情况，比如在 `table.scss` 中就有：
```scss
@include b(table) {
  @include m((group, border)) {
    border: $--table-border;
  }
}
```
编译成 css 就是:
```scss
.el-table--border, .el-table--group {
    border: 1px solid #EBEEF5
}
```
基本上关于 `BEM` 的分析也就这些了，接下来就是在具体写组件的时候，熟练应用了。 接下来用一个例子来回顾一下。

## BEM 例子
要写例子，直接在 [scss在线编译](https://www.sassmeister.com/) 直接模拟，非常方便：
首先先把一些 参数和用到的宏定义，预填进去：
```scss
$namespace: 'el';
$element-separator: '__';
$modifier-separator: '--';
$state-prefix: 'is-';

@mixin b($block) {
  $B: $namespace+'-'+$block !global;
  .#{$B} {
    @content;
  }
}
@mixin e($element) {
  $E: $element !global;
  $selector: &;
  $currentSelector: "";
  @each $unit in $element {
    $currentSelector: #{$currentSelector + "." + $B + $element-separator + $unit + ","};
  }
  @if hitAllSpecialNestRule($selector) {
    @at-root {
      #{$selector} {
        #{$currentSelector} {
          @content;
        }
      }
    }
  } @else {
    @at-root {
      #{$currentSelector} {
        @content;
      }
    }
  }
}
@mixin m($modifier) {
  $selector: &;
  $currentSelector: "";
  @each $unit in $modifier {
    $currentSelector: #{$currentSelector + & + $modifier-separator + $unit + ","};
  }
  @at-root {
    #{$currentSelector} {
      @content;
    }
  }
}


//@debug inspect("Helvetica"); unquote('"Helvetica"')
@function selectorToString($selector) {
    $selector: inspect($selector);
    $selector: str-slice($selector, 2, -2);
    @return $selector;
}
// 判断选择器（.el-button__body--active） 是否包含 '--'
@function containsModifier($selector) {
    $selector: selectorToString($selector);
    @if str-index($selector, $modifier-separator) {
        @return true;
    } @else {
        @return false;
    }
}
// 判断选择器（.el-button__body.is-active） 是否包含 'is'
@function containWhenFlag($selector) {
    $selector: selectorToString($selector);
    @if str-index($selector, '.' + $state-prefix) {
        @return true;
    } @else {
        @return false;
    }
}
//  判断选择器（.el-button__body:before） 是否包含伪元素（:hover）
@function containPseudoClass($selector) {
    $selector: selectorToString($selector);
    @if str-index($selector, ':') {
        @return true;
    } @else {
        @return false;
    }
}
// hit：命中 nest:嵌套
@function hitAllSpecialNestRule($selector) {
    @return containsModifier($selector) or containWhenFlag($selector) or containPseudoClass($selector);
}
```
接下来就可以在下面直接写 sass 代码了：
```scss
.container {
    @include b('button') {
        width: 210px;
        height: 200px;
        @include e('body') {
            color: #ccc;
            @include m('success'){
              background-color: #fff;
            }
        }
    }
}
.container--fix {
    @include b('button') {
        width: 200px;
        height: 200px;
        @include e('body') {
            color: #fff;
            @include m('success'){
              background-color: #fff;
            }
        }
    }
}
```
右边就会生成对应的 css 代码:

![1](1.png)

## 实现响应式布局的 宏 res
其实 `element-ui` 在 sass 定义的宏和函数不多，大部分都是为了 `BEM` 服务的。 只有一个 `宏 res` 是为了实现响应式布局用的:
```scss
@mixin res($key, $map: $--breakpoints) {
  // 循环断点Map，如果存在则返回
  @if map-has-key($map, $key) {
    @media only screen and #{inspect(map-get($map, $key))} {
      @content;
    }
  } @else {
    @warn "Undefeined points: `#{$map}`";
  }
}
```
用过 `element-ui` 应该知道在使用 `col` 组件的时候，是允许设置根据不同的屏幕尺寸来设置不同的响应尺寸的，比如以下这个:
```html
<el-row :gutter="10">
  <el-col :xs="8" :sm="6" :md="4" :lg="3" :xl="1"><div class="grid-content bg-purple"></div></el-col>
  <el-col :xs="4" :sm="6" :md="8" :lg="9" :xl="11"><div class="grid-content bg-purple-light"></div></el-col>
  <el-col :xs="4" :sm="6" :md="8" :lg="9" :xl="11"><div class="grid-content bg-purple"></div></el-col>
  <el-col :xs="8" :sm="6" :md="4" :lg="3" :xl="1"><div class="grid-content bg-purple-light"></div></el-col>
</el-row>
```
其实这个就是通过 res 这个宏实现的：具体配置是这样子的，这个是针对大屏幕的：
```scss
@include res(lg) {
  .el-col-lg-0 {
    display: none;
  }
  @for $i from 0 through 24 {
    .el-col-lg-#{$i} {
      width: (1 / 24 * $i * 100) * 1%;
    }

    .el-col-lg-offset-#{$i} {
      margin-left: (1 / 24 * $i * 100) * 1%;
    }

    .el-col-lg-pull-#{$i} {
      position: relative;
      right: (1 / 24 * $i * 100) * 1%;
    }

    .el-col-lg-push-#{$i} {
      position: relative;
      left: (1 / 24 * $i * 100) * 1%;
    }
  }
}
```
然后再结合这些规定的参数，就可以实现不同的媒体查询的样式：
```scss
$--sm: 768px !default;
$--md: 992px !default;
$--lg: 1200px !default;
$--xl: 1920px !default;

$--breakpoints: (
  'xs' : (max-width: $--sm - 1),
  'sm' : (min-width: $--sm),
  'md' : (min-width: $--md),
  'lg' : (min-width: $--lg),
  'xl' : (min-width: $--xl)
);
```
我们可以在线实现一下： 具体实现：

![1](2.png)

## 参数定制 var.scss
`element-ui` 中所有可定制的参数，全部在 `var.scss` 文件里面。这个文件也是后面做主题定制的关键，因为只要重写这个文件的某些参数，重新打包成 css，就相当于做了一个新的主题了。 这个在后面的文章中会再讲到

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














