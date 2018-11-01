---
title: 整合项目中常用的一些 gulp 组件
date: 2018-11-01 10:33:40
tags: gulp
categories: 前端相关
---
## 前言
虽然现在新的项目，大部分都用webpack来打包，但是还是有很多比较早期的项目都是用 grunt 或者 gulp 打包的，grunt 除了少数几个比较特殊的项目还在用之外， 其他的老项目也都是用gulp来打包，因为对比 grunt来说，gulp实在是优秀太多。
而且一些比较简单的新项目，也不适合用框架来写的，最后也是选择用gulp来打包，所以结合现在项目中的一些用 gulp 打包的项目，我整理了一些平时基本上都会用到的，或者实用的 gulp 插件。
ps: 这不是一篇描述那些流行的gulp组件的介绍文，而是一篇将我之前使用的一些gulp组件的整合文，以便后续我要用的时候，可以到这边来参考
<!--more-->  
## [del](https://www.npmjs.com/package/del) - 删除文件或者目录
这个基本上每次构建都会用到的，就是先清除上一次的打包目录，也可以删除某一个或者几个文件之类的， 设置可以指定那些例外不删除
```javascript
var del = require('del');
gulp.task('cleanBuild', function (cb) {
    del(['build/'], cb);
});
gulp.task('cleanTmp', function (cb) {
    del(['tmp/*.js', '!tmp/unicorn.js'], cb);
});
```
## [gutil.File](https://www.npmjs.com/package/gulp-util) - 创建一个新文件
gulp-util 其实是一个整合很多gulp常用的工具方法的库。但是平时最多用到的还是 log 和 file api。
比如我要新建一个文件，那么就是：
```javascript
var gutil = require('gulp-util');
function string_src(filename, string) {
    var src = require('stream').Readable({objectMode: true});
    src._read = function () {
        this.push(new gutil.File({
            cwd: "",
            base: "",
            path: filename,
            contents: new Buffer(string)
        }));
        this.push(null)
    };
    return src
}
gulp.task('createTmp', function () {
    return string_src("templates.js", "hello world")
            .pipe(gulp.dest('workspace/'))
});
```
这样就在 workspace 目录下新建了一个 内容为 hello world 的名为 templates.js 的文件
## [gulp-tap](https://www.npmjs.com/package/gulp-tap) - 遍历文件进行处理
这个经常被我用在打zip包的时候，对文件夹进行权限赋予
具体看这个：{% post_link www-history-9 %}， 大致的demo就是：
```javascript
var gulp = require('gulp');
var zip = require('gulp-zip');
var tap = require('gulp-tap');

gulp.task('zip', function() {
    global.config.zipName = `${global.isTest ? 'test-' : ''}nms_airdroid_com_${global.buildVersion}.zip`;
    return gulp.src(`build/**`)
        .pipe(tap(function(file) {
            if (file.isDirectory()) {
                file.stat.mode = parseInt('40775', 8);
            }
        }))
        .pipe(zip(global.config.zipName))
        .pipe(gulp.dest('.'));
});
```
## [gulp-zip](https://www.npmjs.com/package/gulp-zip) - 打zip包
用法在上面的 gulp-tap 的示例中，就用到了，这边不细说了，反正用法很简单
## [gulp-gzip](https://www.npmjs.com/package/gulp-gzip) - 对文件进行gzip压缩
有时候我们会需要对一些文件进行gzip压缩，就可以用这个
```javascript
var gulp   = require('gulp');
var gzip   = require('gulp-gzip');
gulp.task('gzip', function() {
  return gulp.src('build/**/*.{css,js}')
    .pipe(gzip({}))
    .pipe(gulp.dest('build/'));
});
```
## [gulp-usemin](https://www.npmjs.com/package/gulp-usemin) - html文件引用的js和css压缩和合并
用来将HTML 文件中（或者templates/views）中没有优化的script 和stylesheets 替换为优化过的版本
gulp-usemin根据预先在html文件（或者其它模板/视图中的文件）中声明好的blocks来执行一系列任务（例如合并文件并重全名、排除一些只在开发过程中引入的脚本以及将css和js中的代码提取出来内嵌在html文件中）来处理未优化的样式和脚本。然后我们可以通过 gulp.dest() 方法将处理的结果输出到其它目录。
首先在html中声明一些任务，重命名，内联，或者删除脚本 （在gulp中，还要配合其他的插件来使用）:
```html
<!-- build:css style.css -->
<link rel="stylesheet" href="css/clear.css"/>
<link rel="stylesheet" href="css/main.css"/>
<!-- endbuild -->

<!-- build:js js/lib.js -->
<script src="../lib/angular-min.js"></script>
<script src="../lib/angular-animate-min.js"></script>
<!-- endbuild -->

<!-- build:js1 js/app.js -->
<script src="js/app.js"></script>
<script src="js/controllers/thing-controller.js"></script>
<script src="js/models/thing-model.js"></script>
<script src="js/views/thing-view.js"></script>
<!-- endbuild -->

<!-- build:remove -->
<script src="js/localhostDependencies.js"></script>
<!-- endbuild -->

<!-- build:inlinejs -->
<script src="../lib/angular-min.js"></script>
<script src="../lib/angular-animate-min.js"></script>
<!-- endbuild -->

<!-- build:inlinecss -->
<link rel="stylesheet" href="css/clear.css"/>
<link rel="stylesheet" href="css/main.css"/>
<!-- endbuild -->
```
最后调用：
```javascript
var gulp = require('gulp');
var usemin = require('gulp-usemin');
var uglify = require('gulp-uglify');
var minifyHtml = require('gulp-minify-html');
var minifyCss = require('gulp-minify-css');
var rev = require('gulp-rev');


gulp.task('usemin', function() {
    return gulp.src('./*.html')
                .pipe(usemin({
                    css: [ rev() ],
                    html: [ minifyHtml({ empty: true }) ],
                    js: [ uglify(), rev() ],
                    inlinejs: [ uglify() ],
                    inlinecss: [ minifyCss(), 'concat' ]
                }))
                .pipe(gulp.dest('build/'));
});
```
## [gulp-useref](https://www.npmjs.com/package/gulp-useref) - html文件引用的js和css压缩和合并
这个组件的功能跟 gulp-usemin 差不多，都是对html文件引用的js和css文件进行压缩和合并。只不过功能更强大。 除了 build:js build:css, 还支持 build:remove.
引用官网的一个构建例子：
```javascript
var gulp = require('gulp');
var gulpIf = require('gulp-if');
var minifyCSS = require('gulp-minify-css');
var uglify = require('gulp-uglify');
var autoprefixer = require('gulp-autoprefixer');
var useref = require('gulp-useref');

// 针对html进行处理，即对html文件进行压缩和合并
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
        .pipe(gulpIf('*.js', uglify()))
        // 这时候生成的html文件会在 build/<version>/workspace/html 里面，这个是临时的，后面要去掉
        .pipe(gulp.dest(global.config.buildVersionDir))
});
```
## [gulp-uglify](https://www.npmjs.com/package/gulp-uglify) - 压缩js
参照上面的 gulp-useref 的用法
## [gulp-clean-css](https://www.npmjs.com/package/gulp-clean-css) - 压缩css
gulp-minify-css 的新名字， 参照上面的 gulp-useref 的用法
## [gulp-htmlmin](https://www.npmjs.com/package/gulp-htmlmin) - 压缩html
使用gulp-htmlmin压缩html，可以压缩页面javascript、css，去除页面空格、注释，删除多余属性等操作
```javascript
var gulp = require('gulp'),
htmlmin = require('gulp-htmlmin');

gulp.task('testHtmlmin', function () {
    var options = {
        removeComments: true,//清除HTML注释
        collapseWhitespace: true,//压缩HTML
        collapseBooleanAttributes: true,//省略布尔属性的值 <input checked="true"/> ==> <input />
        removeEmptyAttributes: true,//删除所有空格作属性值 <input id="" /> ==> <input />
        removeScriptTypeAttributes: true,//删除<script>的type="text/javascript"
        removeStyleLinkTypeAttributes: true,//删除<style>和<link>的type="text/css"
        minifyJS: true,//压缩页面JS
        minifyCSS: true//压缩页面CSS
    };
    gulp.src('src/html/*.html')
        .pipe(htmlmin(options))
        .pipe(gulp.dest('dist/html'));
});
```
## [gulp-rename](https://www.npmjs.com/package/gulp-rename) - 复制并重命名
比如某一个文件在打包的时候，进行重命名
```javascript
var gulp = require('gulp');
var rename = require("gulp-rename");

// 将download页面改成首页
gulp.task('set_download_to_index', function () {
    return gulp.src('build/download.html')
        .pipe(rename('build/index.html'))
        .pipe(gulp.dest('build'));
});
// 还可以修改整个目录结构
gulp.task('set_diff_dir', function () {
    return gulp.src("./src/**/hello.txt")
             .pipe(rename(function (path) {
               path.dirname += "/ciao";
               path.basename += "-goodbye";
               path.extname = ".md"
             }))
             .pipe(gulp.dest("./dist")); // ./dist/main/text/ciao/hello-goodbye.md 
});
```
## [gulp.spritesmith](https://www.npmjs.com/package/gulp.spritesmith) - 生成雪碧图
这个库需要额外依赖 spritesmith 库,
demo：
```javascript
var gulp        = require('gulp'),
    spritesmith = require('gulp.spritesmith');
//运用gulp一键生成[雪碧图]
gulp.task('sprite', function(){
    var workDirectory = 'xxx', outPut = 'yyy';
    return gulp.src( workDirectory+'/images/*.png' )
    .pipe( spritesmith({
        imgName:'sprite.png',
        cssName:'sprite.css'
    }) )
    .pipe( gulp.dest(outPut+'/images') );
});
```
## [gulp-scp2](https://www.npmjs.com/package/gulp-scp2) - 将文件上传到服务器
早期没有做 Jenkins 持续集成的时候，都是用这种方式将打包完的zip包上传到服务器，并且解压的，其中上传到服务器，就会用到这个库，
官网早期构建的一个方式，就是这个：
```javascript
var path = require('path');
var fs = require('fs');
var gulp = require('gulp');
var scp = require('gulp-scp2');


// 上传到测试服务器 （支持 test-www 和 new-www）
gulp.task('upload', cb => {
    var auth;

    if (fs.existsSync(path.join(__dirname, '../scp.auth.js'))) {
        auth = require('../scp.auth.js');
    }

    if (!auth) {
        // 要上传到服务器，需要提供 .ssh 文件做身份验证。
        console.log('要上传到服务器，需要提供 .ssh 文件做身份验证。');
        return;
    }
    // 这边的这个地址，指的是测试地址的目录
    global.config.server_folder_path = `/data/airdroid/wwwroot/${global.config.static_base_url.replace('//', '')}/_history`;
    return gulp.src(global.config.zipName)
        .pipe(scp({
            host: '59.xx.xx.xx',
            username: auth.username,
            privateKey: auth.privateKey,
            dest: global.config.server_folder_path
        })).on('error', function(err) {
            console.log(err);
        });
});
```
## [scp2](https://www.npmjs.com/package/scp2) - 将文件上传到服务器
这个是上传文件到服务器的，跟 gulp-scp2 的差别在于这个更加的底层，所以可扩展的东西比较多，比如通过这个，我们可以知道当前上传的进度是多少，什么时候会上传到 100% 之类的更详细的信息
事实上 gulp-scp2 就是对 scp2 的再封装， 因此对于体积比较大的项目zip包，有时候也会采用这个组件来上，因为这样可以查看当前上传的进度：
```javascript
'use strict';

var path = require('path');
var fs = require('fs');
var gulp = require('gulp');
var Client = require('scp2').Client;
var _ = require("underscore");
var moment = require("moment");

// 上传到测试服务器 （支持 test-nms 和 nms）
gulp.task('upload', cb => {
    var auth;
    var authFilePath = `../auth/${global.isTest ? 'scp_test' : 'scp_release'}.js`;
    if (fs.existsSync(path.join(__dirname, authFilePath))) {
        auth = require(authFilePath);
    }

    if (!auth) {
        // 要上传到服务器，需要提供 .ssh 文件做身份验证。
        console.log('要上传到服务器，需要提供 .ssh 文件做身份验证。');
        return;
    }
    global.config.server_folder_path = auth.dest;

    var client = new Client(_.assign({}, {
        readyTimeout: 10 * 60 * 1000
    }, auth));

    client.on('transfer', function (buffer, uploaded, total) {
        process.stdout.write(`\r[ ${moment().format('HH:mm:ss')} ] Uploading [ ${(100.0 * uploaded / total).toFixed(2)}% ]\n`);
    });

    client.upload(global.config.zipName, auth.dest, function (err) {
        process.stdout.write(`\r[ ${moment().format('HH:mm:ss')} ] Uploaded [ 100% ]\n`);
        client.close();
        //del(global.config.zipName);
        cb();
    });
});

```
其中 scp_test.js 是类似于这种的：
```javascript
module.exports = {
    host: 'xxx.xxx.xxx.xxx',
    // upload task need
    username: 'root',
    // shell task need
    userName: 'root',
    password: 'xx',
    dest: '/data/server/wwwroot/xx/_history'
};
```
## [ssh2shell](https://www.npmjs.com/package/ssh2shell) - 在服务器上进行shell操作
既然将zip包传到服务器了，那么就要在服务器上进行解压了，也就是要在服务器上进行shell操作了
这边把之前官网的一段task任务取出来：
```javascript
var gulp = require('gulp');
var path = require('path');
var fs = require('fs');
var SSH2Shell = require ('ssh2shell');


// 上传到测试服务器后，进行一些shell操作，解压这个上传的zip包
gulp.task('shell', cb => {
    var auth;

    if (fs.existsSync(path.join(__dirname, '../scp.auth.js'))) {
        auth = require('../scp.auth.js');
    }

    if (!auth) {
        // 要上传到服务器，需要提供 .ssh 文件做身份验证。
        console.log('要上传到服务器，需要提供 .ssh 文件做身份验证。');
        return;
    }
    var host = {
        server: {
            host: '59.xx.xx.xx',
            userName: auth.username,
            privateKey: auth.privateKey
        },
        commands: [`cd ${global.config.server_folder_path}`, `unzip  -o -d .. ${global.config.zipName}`]
    };
    //Create a new instance passing in the host object
    var SSH = new SSH2Shell(host);
    //Use a callback function to process the full session text
    var callback = function(sessionText){
        console.log(sessionText);
        cb();
    };
    //Start the process
    SSH.connect(callback);
});
```
## this.seq.slice(-1)[0] - 获取当前运行的gulp任务名
这个不是组件，而是 gulp 自带的功能，但是其实在构建中，很常见，比如：
有时候，我们需要获取当前执行的gulp任务的名称， 比如 执行的是 gulp r 
```javascript
var task_name = this.seq.slice(-1)[0] === "r" ? "release" : "test";
```
那么有时候，我们需要在其他的子任务里面去判断这个任务名是r还是其他的，比如d，从而执行不同的逻辑判断。
举个例子，在执行zip子任务的时候，如果当前任务是r的话，那么zip那么就加上release，否则就是默认的test：
```javascript
var gulp = require('gulp');
var zip = require('gulp-zip');

// 打成zip包
gulp.task('zip', function() {
    var task_name = this.seq.slice(-1)[0] === "r" ? "release" : "test";
    global.config.zipName = `${global.config.static_base_url.replace(/\//g,"").replace(/\./g,"_")}_${global.config.version}_${task_name}.zip`;
    return gulp.src(`${global.config.buildDir}/**`)
        .pipe(zip(global.config.zipName))
        .pipe(gulp.dest('.'));
});
```
## [gulp-html-beautify](https://www.npmjs.com/package/gulp-html-beautify) - html美化&格式化
这玩意儿可以美化html 代码。 包括缩进，删空行 之类的。
```javascript
var gulp = require('gulp');
var htmlbeautify = require('gulp-html-beautify');
 
gulp.task('htmlbeautify', function() {
  var options = {
    {indentSize: 2}
  };
  gulp.src('./src/*.html')
    .pipe(htmlbeautify(options))
    .pipe(gulp.dest('./public/'))

});
```
## [gulp-email-builder](https://www.npmjs.com/package/gulp-email-builder) - 邮件模板内嵌
这个东西，就是可以把css引入的文件变成style inline 内联。主要是是用来做邮件模板的。内嵌 embed 和 内联 inline是不一样的。
- 内嵌 embed 其实就是 &lt;style&gt;xxx&lt;/style&gt; 块
- 内联 inline 就是 &lt;p style="xxx"&gt;&lt;/p&gt; style 样式

比如开发邮件模板的时候，html是这样子的：
```html
<!DOCTYPE html>
<html>
<head>
  <!-- styles will be inlined -->
  <link rel="stylesheet" type="text/css" href="../css/styles.css">
 
  <!-- styles will be embedded -->
  <link rel="stylesheet" type="text/css" href="../css/otherStyles.css" data-embed>
 
  <!-- link tag will be preserved and styles will not be inlined or embedded -->
  <link href='http://fonts.googleapis.com/css?family=Open+Sans' rel='stylesheet' type='text/css' data-embed-ignore>
 
  <!-- styles will be inlined -->
  <style>
    p { color: red; }
  </style> 
 
  <!-- styles will be embedded -->
  <style data-embed>
    h1 { color: black; }
  </style> 
</head>
<body>
  <h1>Heading</h1>
  <p>Body</p>
</body>
</html>
```
然后在构建的时候，就会把这些html文件，转化为css inline 或者 embed 的方式：
```javascript
var gulp = require('gulp'),
    staticfy = require("gulp-staticfy"),
    replace = require("gulp-replace"),
    emailBuilder = require('gulp-email-builder'),
    htmlbeautify = require('gulp-html-beautify');

// 编译
gulp.task('build', cb => {
    return gulp.src(fileArr.map(item => {
        return `${pageDir}${item}/${item}.html`
    }))
        // 静态模板预编译
        .pipe(staticfy({
            server_url: path.resolve(workDir)
        }))
        // 消除引入的js文件
        .pipe(replace(/<script.*<\/script>/g, ''))
        // 样式内嵌
        .pipe(emailBuilder({ encodeSpecialChars: true }).build())
        // 格式化html，让他变好看
        .pipe(htmlbeautify({
            indentSize: 4
        }))
        .pipe(gulp.dest(buildDir));
});
```
## [gulp-staticfy](https://www.npmjs.com/package/gulp-staticfy) - 模板静态化预编译
这个是自己写的gulp，主要是用来做模板的静态化预编译，我们的项目，只要是涉及到多语言的，都会用到这个组件来构建多语言的静态页面
具体看： {% post_link www-history-4 %}
## [gulp-autoprefixer](https://www.npmjs.com/package/gulp-autoprefixer) - 自动添加浏览器前缀
使用gulp-autoprefixer根据设置浏览器版本自动处理浏览器前缀。而不必考虑各浏览器兼容前缀。
```javascript
var gulp = require('gulp'),
autoprefixer = require('gulp-autoprefixer');

gulp.task('testAutoFx', function () {
    gulp.src('src/css/index.css')
        .pipe(autoprefixer({
            browsers: ['last 2 versions'],
            cascade: true, //是否美化属性值 默认：true 像这样：
            //-webkit-transform: rotate(45deg);
            // transform: rotate(45deg);
            remove:true //是否去掉不必要的前缀 默认：true
        }))
        .pipe(gulp.dest('dist/css'));
});
```
## [gulp-concat](https://www.npmjs.com/package/gulp-concat) - 合并文件
将多个css或者多个js合并成一个文件。
```javascript
var gulp = require('gulp'),
concat = require('gulp-concat');

gulp.task('scripts', function() {
    return gulp.src('./lib/*.js')
            .pipe(concat('all.js'))
            .pipe(gulp.dest('./dist/'));
});
```
## [gulp-file-split](https://www.npmjs.com/package/gulp-file-split) - 将大文件切块
这个组件的作用是把一个大文件均匀分割成若干个文件。
这个需求比较特殊，主要是因为早期做后台的时候，因为后台是用broswerfiy构建的，所以最后面会有一个很大的js。
在生成最后构建的时候，会把这个js切割成相同大小的几个文件，然后在html页面用ajax请求下来，并拼接起来。再请求，这个组件也是那时候找不到类似的，所以自己重新写了一个：
```javascript
var gulp = require('gulp'),
gsplit = require('gulp-file-split');
// 将 app.js 独立拆分为几个独立的js
gulp.task('split', ['delComment'],function() {
    return gulp.src([global.config.scripts.appSrc])
            .pipe(gsplit({
                    prefix: 'resource_',
                    ext: 'js',
                    count: global.split
                }))
            .pipe(gulp.dest(global.config.scripts.dest));
});
```
## [gulp-rev](https://www.npmjs.com/package/gulp-rev) - 为文件生成hash
这个就是用来为资源文件生成hash值的。
之前做一个项目的时候，为了避免cdn缓存和版本迭代的时候，没有修过的文件不让其重新更新。因此会在打包的时候，会对每一个资源文件生成hash值，然后替换html里面的url。
不过后面用 gulp-usemin 这个整合工具来做了，刚好这个工具也有整合这个：
```javascript
var gulp = require('gulp');
var usemin = require('gulp-usemin');
var uglify = require('gulp-uglify');
var minifyHtml = require('gulp-minify-html');
var minifyCss = require('gulp-minify-css');
var rev = require('gulp-rev');
 
 
gulp.task('usemin', function() {
  return gulp.src('./*.html')
    .pipe(usemin({
      css: [ rev() ],
      html: [ minifyHtml({ empty: true }) ],
      js: [ uglify(), rev() ],
      inlinejs: [ uglify() ],
      inlinecss: [ minifyCss(), 'concat' ]
    }))
    .pipe(gulp.dest('build/'));
});
```
## [gulp-if](https://www.npmjs.com/package/gulp-if) - if判断条件
有时候在构建的时候，会需要去判断条件，比如当前是打包正式版，还是打包测试版，就需要用到这个判断逻辑, 比如有一个是这样子的：
```javascript
var gulpif = require('gulp-if');
var replace = require("gulp-replace");
// 复制 html 文件到 build 目录
gulp.task('copy_html', function () {
    return gulp.src([
        PAGE_HTML_FILES,
        path.join(baseCfg.BUILD_DIR, '**/*.html') // simple-pages
    ])
    .pipe(flatten())
    .pipe(replaceHtmlHash({
        manifest: manifestFilename,
        clean: true
    }))
    // 针对一些html 的标签，的href链接加上缓存
    .pipe(replace(/data-no-cache (href=")(\S+)"/gi, '$1$2?' + buildTime + '"'))
    .pipe(modifyHtml())
    .pipe(addBaiduAnalytics())
    .pipe(addDebugScript())
    // 将上面的js替换成要本地缓存的方式（线上版本）
    .pipe(gulpif(isReleaseTask(), useLocalStorageCache()))
    // 如果含有css的缓存加载，那么就要把body先隐藏，免得后面css下来的时候，会发生界面变化
    .pipe(gulpif(isReleaseTask(), hiddenBodyWhenHasCss()))
    // 如果是线上版本，那么要执行这一步
    .pipe(gulpif(isReleaseTask(), replace('</html>', '<script type="text/javascript" src="js/lib/get.js?2"></script>\n</html>')))
    .pipe(gulp.dest(baseCfg.BUILD_DIR));
});
```
## [gulp-replace](https://www.npmjs.com/package/gulp-replace) - 字符串替换
有时候会在打包的时候，对某一些页面或者js的东西，进行替换或者添加某些东西，用这个就可以很容易实现。
例子直接看上面的 gulp-if 的例子
## [gulp-sequence](https://www.npmjs.com/package/gulp-sequence) - 顺序执行
这个就是让任务顺序执行，如果里面有中括号的话，那么中括号里面的任务是并行执行的
```javascript
seq = require("gulp-sequence"),
gulp.task('t', seq(['copy_html', 'copy_assets'],'set_download_to_index'));
```
即 copy_html 和 copy_assets 一起执行，全部执行完了之后，再执行 set_download_to_index
## [run-sequence](https://www.npmjs.com/package/run-sequence) - 顺序执行
这个也是顺序执行，跟 gulp-sequence 的差别就是用法问题，run-sequence 是在 task的回调函数里面的，这个函数还可以有其他的逻辑，比如：
```javascript
var gulp = require('gulp');
var runSequence = require('run-sequence');
gulp.task('dev', ['clean'], function(cb){
    cb = cb || function(){};
    if(global.isPublish){
        runSequence(['images', 'views','script', 'css','browserify'], cb);
    }else{
        runSequence(['images', 'views','script', 'css','browserify'], 'watch', cb);
    }
});
```
事实上，gulp-sequence 就是基于 run-sequence 这个组件的。
## 启动一个server
有时候在开发的时候，有时候启动的时候，会创建一个server，这时候就可以这样做：
```javascript
var http    = require('http');
var express = require('express');
var gulp    = require('gulp');
var gutil   = require('gulp-util');
var morgan  = require('morgan');

gulp.task('server', function() {

  var server = express();

  // log all requests to the console
  server.use(morgan('dev'));
  server.use(express.static(global.config.dist.root));

  // Serve index.html for all routes to leave routing up to Angular
  server.all('/*', function(req, res) {
      res.sendFile('index.html', { root: 'build' });
  });

  // Start webserver if not already running
  var s = http.createServer(server);
  s.on('error', function(err){
    if(err.code === 'EADDRINUSE'){
      gutil.log('Development server is already started at port ' + global.config.serverport);
    }
    else {
      throw err;
    }
  });
  s.listen(global.config.serverport);
});
```
其实就是通过 node 创建一个 server






