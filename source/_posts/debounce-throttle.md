---
title: 前端工具集(15) -- 节流和防抖
date: 2019-04-17 10:15:13
tags: js
categories: 前端工具集
---
## 前言
防抖和节流的作用都是防止函数多次调用。区别在于，假设一个用户一直触发这个函数，且每次触发函数的间隔小于wait，防抖的情况下只会调用一次，而节流的 情况会每隔一定时间（参数wait）调用函数。
## 防抖 debounce
防抖的原理就是：你尽管触发事件，但是我一定在事件触发 n 秒后才执行，如果你在一个事件触发的 n 秒内又触发了这个事件，那我就以新的事件的时间为准，n 秒后才执行，总之，就是要等你触发完事件 n 秒内不再触发事件，我才执行。
一般试用的场景就是： 
- window 的 resize 和 scroll
- input 的 keyup 或者 input 事件
- 其他，比如按钮的防二次点击

<!--more-->
### 简单版
来个简单版的：
```html
// func是用户传入需要防抖的函数
// wait是等待时间
const debounce = (func, wait = 50) => {
  // 缓存一个定时器id
  let timer = 0
  // 这里返回的函数是每次用户实际调用的防抖函数
  // 如果已经设定过定时器了就清空上一次的定时器
  // 开始一个新的定时器，延迟执行用户传入的方法
  return function(...args) {
    if (timer) clearTimeout(timer)
    timer = setTimeout(() => {
      func.apply(this, args)
    }, wait)
  }
}
```
不难看出如果用户调用该函数的间隔小于wait的情况下，上一次的时间还未到就被清除了，并不会执行函数。
这是一个简单版的防抖，但是有缺陷，这个防抖只能在最后调用。一般的防抖会有 immediate 选项，表示是否立即调用。这两者的区别，举个栗子来说：
- 例如在搜索引擎搜索问题的时候，我们当然是希望用户输入完最后一个字才调用查询接口，这个时候适用延迟执行的防抖函数，它总是在一连串（间隔小于wait的）函数触发之后调用。
- 例如用户在点击支付按钮的时候，我们希望用户点第一下的时候就去调用接口，并且成功调出支付页面，用户就可以立马开始支付了，这个情况适用立即执行的防抖函数，它总是在第一次调用，并且下一次调用必须与前一次调用的时间间隔大于wait才会触发。其实就是按钮的防二次点击。

### 立即执行
接下来就实现一下带立即执行的函数：
```html
/**
* 防抖函数，返回函数连续调用时，空闲时间必须大于或等于 wait，func 才会执行
*
* @param  {function} func        回调函数
* @param  {number}   wait        表示时间窗口的间隔
* @param  {boolean}  immediate   设置为ture时，是否立即调用函数
* @return {function}             返回客户调用函数
*/
function debounce (func, wait = 50, immediate = true) {
  let timer, context, args

  // 延迟执行函数
  const later = () => setTimeout(() => {
    // 延迟函数执行完毕，清空缓存的定时器序号
    timer = null
    // 延迟执行的情况下，函数会在延迟函数中执行
    // 使用到之前缓存的参数和上下文
    if (!immediate) {
      func.apply(context, args)
      context = args = null
    }
  }, wait)

  // 这里返回的函数是每次实际调用的函数
  return function(...params) {
    // 如果没有创建延迟执行函数(later)，就创建一个
    if (!timer) {
      timer = later()
      // 如果是立即执行，调用函数
      // 否则缓存参数和调用上下文
      if (immediate) {
        func.apply(this, params)
      } else {
        context = this
        args = params
      }
    // 如果已有延迟执行函数（later），调用的时候清除原来的并重新设定一个
    // 这样做延迟函数会重新计时
    } else {
      clearTimeout(timer)
      timer = later()
    }
  }
}
```
这边是执行的截图：
![1](1.png)
可以看到如果 immediate 为 false的话，并不会马上输出 3。 但是如果为 true 的话，就会马上输出 3。
整体函数实现的不难，总结一下。
- 对于按钮防点击来说的实现：如果函数是立即执行的，就立即调用，如果函数是延迟执行的，就缓存上下文和参数，放到延迟函数中去执行。一旦我开始一个定时器，只要我定时器还在，你每次点击我都重新计时。一旦你点累了，定时器时间到，定时器重置为 null，就可以再次点击了。
- 对于延时执行函数来说的实现：清除定时器ID，如果是延迟调用就调用函数

### 返回值
需要注意的一点就是，func 有时候也是会有返回值的，所以我们也要返回函数的执行结果，但是当 immediate 为 false 的时候，因为使用了 setTimeout ，我们将 func.apply(context, args) 的返回值赋给变量，最后再 return 的时候，值将会一直是 undefined，所以我们只在 immediate 为 true 的时候返回函数的执行结果。
所以加上返回值的函数如下：
```html
function debounce (func, wait = 50, immediate = true) {
  let timer, context, args, result=null

  // 延迟执行函数
  const later = () => setTimeout(() => {
    // 延迟函数执行完毕，清空缓存的定时器序号
    timer = null
    // 延迟执行的情况下，函数会在延迟函数中执行
    // 使用到之前缓存的参数和上下文
    if (!immediate) {
      func.apply(context, args)
      context = args = null
    }
  }, wait)

  // 这里返回的函数是每次实际调用的函数
  return function(...params) {
    // 如果没有创建延迟执行函数(later)，就创建一个
    if (!timer) {
      timer = later()
      // 如果是立即执行，调用函数
      // 否则缓存参数和调用上下文
      if (immediate) {
        result = func.apply(this, params)
      } else {
        context = this
        args = params
      }
    // 如果已有延迟执行函数（later），调用的时候清除原来的并重新设定一个
    // 这样做延迟函数会重新计时
    } else {
      clearTimeout(timer)
      result = null
      timer = later()
    }
    return result
  }
}
```
只有首次立即执行才有返回值。
这边是执行的截图：
![1](2.png)
### 取消
有时候会有一个需求，就是希望能取消 debounce 函数，比如说我 debounce 的时间间隔是 10 秒钟，immediate 为 true，这样的话，我只有等 10 秒后才能重新触发事件，现在我希望有一个按钮，点击后，取消防抖，这样我再去触发，就可以又立刻执行啦。
所以还可以这样写：
```html
function debounce (func, wait = 50, immediate = true) {
  let timer, context, args, result=null

  // 延迟执行函数
  const later = () => setTimeout(() => {
    // 延迟函数执行完毕，清空缓存的定时器序号
    timer = null
    // 延迟执行的情况下，函数会在延迟函数中执行
    // 使用到之前缓存的参数和上下文
    if (!immediate) {
      func.apply(context, args)
      context = args = null
    }
  }, wait)

  // 这里返回的函数是每次实际调用的函数
  let debounced = function(...params) {
    // 如果没有创建延迟执行函数(later)，就创建一个
    if (!timer) {
      timer = later()
      // 如果是立即执行，调用函数
      // 否则缓存参数和调用上下文
      if (immediate) {
        result = func.apply(this, params)
      } else {
        context = this
        args = params
      }
    // 如果已有延迟执行函数（later），调用的时候清除原来的并重新设定一个
    // 这样做延迟函数会重新计时
    } else {
      clearTimeout(timer)
      result = null
      timer = later()
    }
    return result
  }
  // 取消函数
  debounced.cancel = () => {
      clearTimeout(timer)
      timer = null
  }

  return debounced
}
```
这边是执行的截图：
![1](3.png)

这样就完成了一个完整的防抖函数了，事实上 underscore 的防抖函数就是这样子的。
## 节流 throttle 
节流的原理很简单：
如果你持续触发事件，每隔一段时间，只执行一次事件。根据首次是否执行以及结束后是否执行，效果有所不同，实现的方式也有所不同。 我们用 leading 代表首次是否执行，trailing 代表结束后是否再执行一次。
### 简单版
```html
function throttle(fn) {
    // 通过闭包保存一个标记
    let canRun = true; 
    return function () {
        // 在函数开头判断标记是否为true，不为true则return
        if (!canRun) return; 
        // 立即设置为false
        canRun = false; 
        // 将外部传入的函数的执行放在setTimeout中
        setTimeout(() => { 
            fn.apply(this, arguments);
            // 最后在setTimeout执行完毕后再把标记设置为true(关键)表示可以执行下一次循环了。当定时器没有执行的时候标记永远是false，在开头被return掉
            canRun = true;
        }, 500);
    };
}
```
### 完整版
当然这个是简单版本的，接下来我们就将 underscore 的 throttle 的函数列出来, 这个就是完整版，当然我重新将一些依赖抽了一下：
```html
function getNow() {
  return +new Date()
}

/**
* underscore 节流函数，返回函数连续调用时，func 执行频率限定为 次 / wait
*
* @param  {function}   func       回调函数
* @param  {number}     wait       表示时间窗口的间隔
* @param  {object}     options    如果想忽略开始函数的的调用，传入{leading: false}。
*                                 如果想忽略结尾函数的调用，传入{trailing: false}
*                                 两者不能共存，否则函数不能执行
* @return {function}              返回客户调用函数
*/
function throttle (func, wait, options) {
    var context, args, result;
    var timeout = null;
    // 之前的时间戳
    var previous = 0;
    // 如果 options 没传则设为空对象
    if (!options) options = {};
    // 定时器回调函数
    var later = function() {
      // 如果设置了 leading，就将 previous 设为 0
      // 用于下面函数的第一个 if 判断
      previous = options.leading === false ? 0 : getNow();
      // 置空一是为了防止内存泄漏，二是为了下面的定时器判断
      timeout = null;
      result = func.apply(context, args);
      if (!timeout) context = args = null;
    };
    return function() {
      // 获得当前时间戳
      var now = getNow();
      // 首次进入前者肯定为 true
      // 如果需要第一次不执行函数
      // 就将上次时间戳设为当前的
      // 这样在接下来计算 remaining 的值时会大于0
      if (!previous && options.leading === false) previous = now;
      // 计算剩余时间
      var remaining = wait - (now - previous);
      context = this;
      args = arguments;
      // 如果当前调用已经大于上次调用时间 + wait
      // 或者用户手动调了时间
      // 如果设置了 trailing，只会进入这个条件
      // 如果没有设置 leading，那么第一次会进入这个条件
      // 还有一点，你可能会觉得开启了定时器那么应该不会进入这个 if 条件了
      // 其实还是会进入的，因为定时器的延时
      // 并不是准确的时间，很可能你设置了2秒
      // 但是他需要2.2秒才触发，这时候就会进入这个条件
      if (remaining <= 0 || remaining > wait) {
        // 如果存在定时器就清理掉否则会调用二次回调
        if (timeout) {
          clearTimeout(timeout);
          timeout = null;
        }
        previous = now;
        result = func.apply(context, args);
        if (!timeout) context = args = null;
      } else if (!timeout && options.trailing !== false) {
        // 判断是否设置了定时器和 trailing
        // 没有的话就开启一个定时器
        // 并且不能不能同时设置 leading 和 trailing
        result = null;
        timeout = setTimeout(later, remaining);
      } else {
        result = null;
      }
      return result;
    };
  };
```
这边是执行的截图：
![1](4.png)
从截图可以看到，第一次肯定是会执行的，然后在计时周期，接下来都不会再执行，但是一旦计时时间到了，这时候不管有没有再调用，都会执行最后一下。
如果不想要最后一下的被动执行，那么就这样写：
```html
var ss3 = throttle(function(){console.log(3); alert(1); return new Date();}, 10000, {trailing:false})
```
![1](5.png)
这样子，就算最后时间到了，也不会执行最后一下。 相同的道理，如果要忽略第一下的主动执行，那么就这样：
```html
var ss3 = throttle(function(){console.log(3); alert(1); return new Date();}, 10000, {leading:false})
```
但是需要注意一点的是，leading 和 trailing 不能同时为 false。
当然还可以跟 debounce 那样再加上一个 cancel 函数，差不多的写法。
## 参考资料
[面试图谱-防抖](https://yuchengkai.cn/docs/frontend/#%E9%98%B2%E6%8A%96)
[JavaScript专题之跟着underscore学防抖](https://github.com/mqyqingfeng/Blog/issues/22)
[JavaScript专题之跟着 underscore 学节流](https://github.com/mqyqingfeng/Blog/issues/26)


