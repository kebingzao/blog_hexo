---
title: 官网构建优化流程(9) - windows 下打zip包到服务器没有执行权限的问题
date: 2018-09-13 16:28:28
tags: 
- js
- gulp
categories: 
- 前端相关
- 官网构建优化流程
---
## 前言
前段时间，在用 gulp 打包的时候，出现了一个问题： 
使用了gulp-zip组件将build目录打包成zip包，并且使用了 gulp-scp2 上传到服务器，最后用 ssh2shell 在服务器上执行zip包解压命令。但是出现了一个问题。就是在windows下，使用了gulp-zip 打的zip包，上传到服务器解压的时候，发现里面的目录会没有执行权限。
![1](www-history-9/1.png)
<!--more-->
也就是解压之后，里面的目录会变成这个样子，显示是绿色，而且权限比之上面蓝色的目录，少了执行权限 x。 而在mac或者Linux下打的zip包都正常，可以正常解压并执行。
## 解决 使用chmod命令
最简单的方式就是就是上传到服务器的时候，用脚本再执行这个指令：
```javascript
chmod 755 xxx.zip
```
## 解决 gulp-tap
最好的方式，其实就是在打包的时候，就要把zip包的权限设置对。这样一了百了。
刚开始寻找的解决方案是，用这个组件 [gulp-chmod](https://www.npmjs.com/package/gulp-chmod)
```javascript
'use strict';

var gulp = require('gulp');
var zip = require('gulp-zip');
var chmod = require('gulp-chmod');

// 打成zip包
gulp.task('zip', function() {
    var task_name = this.seq.slice(-1)[0] === "r" ? "release" : "test";
    global.config.zipName = `${global.config.static_base_url.replace(/\//g,"").replace(/\./g,"_")}_${global.config.version}_${task_name}.zip`;
    return gulp.src(`${global.config.buildDir}/**`)
        .pipe(zip(global.config.zipName))
        .pipe(chmod(0o755))
        .pipe(gulp.dest('.'));
});
```
将他设置为 **755**， 但是后面发现还是不行，还是会出现这个权限的问题。
后面是找了另一个 [gulp-tap](https://www.npmjs.com/package/gulp-tap) 来解决这个问题，这个插件很有作用，它可以用来遍历 gulp.src() 指定的那些文件；利用这个特性，以及npm下自带的 path 插件，即可获取到每个文件的文件名；在特定场景需求里，它帮了我很大忙。
所以改成：
```javascript
'use strict';

var gulp = require('gulp');
var zip = require('gulp-zip');
var tap = require('gulp-tap');

// 打成zip包
gulp.task('zip', function() {
    var task_name = this.seq.slice(-1)[0] === "r" ? "release" : "test";
    global.config.zipName = `${global.config.static_base_url.replace(/\//g,"").replace(/\./g,"_")}_${global.config.version}_${task_name}.zip`;
    return gulp.src(`${global.config.buildDir}/**`)
        .pipe(tap(function(file) {
            if (file.isDirectory()) {
                file.stat.mode = parseInt('40775', 8);
            }
        }))
        .pipe(zip(global.config.zipName))
        .pipe(gulp.dest('.'));
});
```
这样子就成功解决了zip包权限的问题了。
![1](www-history-9/2.png)



















