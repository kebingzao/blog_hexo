---
title: 官网构建优化流程(8) - gulp打包构建在ie8会报错
date: 2018-09-13 16:12:04
tags: 
- js
- ie
- gulp
categories: 
- 前端相关
- 官网构建优化流程
---
## 原因
用gulp打包之后，后面发现在ie8下会报错。原因是 **framework.min.js**  的一个错误。 后面发现是因为在进行 **uglify** 打包的时候，在ie8下会有问题。
## 解决
后面查了一下，在用 uglify 压缩js的时候，加上这个配置就可以了
```javascript
{    
    compress: { screw_ie8: false },   
     mangle: { screw_ie8: false },   
      output: { screw_ie8: false }
}
```
<!--more-->
所以后面这个构建的任务就改成：
```javascript
gulp.task('useref', function(){
    return gulp.src(global.config.htmlFiles.map(item => {
        return `${global.config.workDir}/${item}`
    }), {base: './html'})
        //gulp.src(`${global.config.workDir}/${global.config.langDir}/**/*.html`)
        .pipe(useref({searchPath: "."}))
        // Minifies only if it's a CSS file
        .pipe(gulpIf('*.css', minifyCSS()))
        .pipe(gulpIf('*.css', autoprefixer()))
        // Uglifies only if it's a Javascript file
        .pipe(gulpIf('*.js', uglify({
            compress: { screw_ie8: false },
            mangle: { screw_ie8: false },
            output: { screw_ie8: false }
        })))
        // 这时候生成的html文件会在 build/<version>/workspace/html 里面，这个是临时的，后面要去掉
        .pipe(gulp.dest(global.config.buildVersionDir))
});
```
这样就 ok了。

---
系列文章
{% post_link www-history-1 %}
{% post_link www-history-2 %}
{% post_link www-history-3 %}
{% post_link www-history-4 %}
{% post_link www-history-5 %}
{% post_link www-history-6 %}
{% post_link www-history-7 %}
{% post_link www-history-8 %}
{% post_link www-history-9 %}
{% post_link www-history-10 %}
{% post_link www-history-11 %}
{% post_link www-history-12 %}
{% post_link www-history-13 %}