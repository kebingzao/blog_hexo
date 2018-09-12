---
title: 官网构建优化流程(3) - grunt静态页面预编译插件 grunt-staticfy
date: 2018-09-12 19:40:11
tags: 
- js
- grunt
categories: 
- 前端相关
- 官网构建优化流程
---
## 前言
通过 {% post_link www-history-2 %} 可以知道整个的grunt打包流程。其中最大的难点就是 静态页面预编译 这一块有用到了一个插件。
这个插件是我们团队那时候自己写的一个grunt插件，专门用来静态页面预编译。 
[github 传送门 grunt-staticfy](https://github.com/AxesTeam/grunt-staticfy)
[npm 组件传送门 grunt-staticfy](https://www.npmjs.com/package/grunt-staticfy)
其原理就是先启用一个web server，然后将要静态化的页面，用phantomJS 跑起来，最后再把 phantomJS 跑完新的页面保存起来。
<!--more-->
## 代码分析
**staticfy.js**:
```javascript
"use strict";

module.exports = function (grunt) {
    var SimpleServer = require('./lib/simpleServer.js');
    var exec = require('child_process').exec;
    var path = require('path');
    var _ = require('underscore');

    grunt.registerMultiTask('staticfy', 'Staticfy your website', function () {
        var injectScript, options, gruntDone, fakeFiles;

        gruntDone = grunt.task.current.async();
        // Merge task-specific and/or target-specific options with these defaults.
        options = this.options({
            query_string: '',
            cwd: '',
            inject_script: '',
            onfinish: function (str) {
                return str;
            },
            wait_request: ' '
        });

        // Convert to string, it would be used by phantomjs later.
        if (grunt.util.kindOf(options.inject_script) === 'function') {
            injectScript = options.inject_script
                .toString()
                .replace(/(function \(\) \{([\w\W]*?)\})/, "$2")
                .trim();
        } else {
            injectScript = options.inject_script;
        }

        fakeFiles = _.map(this.files, function (file) {
            var src, wwwDir, basePath;

            src = file.src[0];
            wwwDir = options.cwd || path.dirname(src);
            basePath = src.replace(wwwDir, '').replace(/(^\/|\/$)/g, '');

            grunt.log.writeln('File "' + basePath + '" staticfying.');

            if (!grunt.file.exists(src)) {
                // File not exist
                grunt.log.warn('Source file "' + src + '" not found.');
                return;
            }

            var url, queryString;
            queryString = options.query_string;
            url = 'http://localhost:{{port}}/' + basePath;
            if (queryString) url += '?' + queryString;

            return {
                src: src,
                dest: file.dest,
                wwwDir: wwwDir,
                url: url
            };
        });

        var fileGroups = _.groupBy(fakeFiles, 'wwwDir');
        var restGroupCount = _.size(fileGroups);
        _.each(fileGroups, function (files, wwwDir) {

            // Run a server to serve html files, we need a static server so we
            // wouldn't got a crossdomain error if the page use ajax or etc.
            SimpleServer.start(wwwDir, function (server) {
                // Replace {{port}} with server.port
                files = _.each(files, function (opt) {
                    opt.url = opt.url.replace('{{port}}', server.port);
                });

                // Call phantom/savePage.js
                phantom(files, injectScript, options.wait_request, function () {
                    _.each(files, function (file) {
                        // After phantom, read the dest html file then normalizelf() and apply onfinish() callback.
                        var str = grunt.file.read(file.dest);
                        str = grunt.util.normalizelf(str);
                        str = options.onfinish(str);
                        grunt.file.write(file.dest, str);
                        grunt.log.writeln('File "' + file.dest + '" created.');
                    });
                    restGroupCount--;
                    if (restGroupCount === 0) {
                        gruntDone();
                    }
                });
            });
        });
    });

    function phantom(files, injectScript, waitRequest, callback) {
        var phantomProgram, cmd;

        phantomProgram = path.join(__dirname, '/lib/phantom/savePage.js');

        cmd = 'phantomjs "'
        + phantomProgram + '" "'
        + escape(JSON.stringify(files)) + '" "'
        + injectScript + '" '
        + waitRequest;
        exec(cmd, callback);
        // grunt.log.writeln(cmd);
    }
};
```
当用 phantomJS 跑出来之后，再用 **lib/phantom/savePage.js** 去保存这个新的html页面：
```javascript
var page = require('webpage').create();
var fs = require('fs');
var system = require('system');
var args = system.args;

var file, url, dest;
var files = unescape(args[1]);
var inject_script = args[2] || 'no';
var wait_request = args[3] || 'no';

console.log('files:', files);
console.log('eval:', inject_script);
console.log('wait_request:', wait_request);

files = JSON.parse(files);

run();

function run() {
    file = files.pop();
    url = file.url;
    dest = file.dest;

    page.open(url, function () {
        if (inject_script !== 'no') {
            page.evaluate(function (evalStr) {
                eval(evalStr);
            }, inject_script);
        }
        if (wait_request == 'no') writeFile();
    });

    // Show page console
    page.onConsoleMessage = function (msg) {
        console.log(msg);
    };

    page.onResourceReceived = function (response) {
        if (response.stage === 'end' && wait_request != 'no' && response.url.indexOf(wait_request) > -1) {
            // Timeout 0 for execute page script.
            console.log(page.content);
            setTimeout(function () {
                writeFile();
            }, 0);
        }
    };
}

function writeFile() {
    console.log('file created:', dest);
    fs.write(dest, page.content, 'w');
    if (files.length === 0) {
        console.log('phantom.exit');
        phantom.exit();
    } else {
        run();
    }
}
```
而要类似于静态化的页面就类似于 index.html：
{% codeblock lang:html %}
<!DOCTYPE html>
<!--[if lt IE 7]>
<html class="lt-ie9 lt-ie8 lt-ie7"> <![endif]--><!--[if IE 7]>
<html class="lt-ie9 lt-ie8"> <![endif]--><!--[if IE 8]>
<html class="lt-ie9"> <![endif]--><!--[if gt IE 8]><!-->
<html> <!--<![endif]-->
<head>
    <meta charset="UTF-8">
    <title></title>
    <link rel="shortcut icon" type="image/x-icon" href="/workspace/img/favicon.ico"/>
    <!-- build:css ./css/main.min.css -->
    <link rel="stylesheet" href="/workspace/css/base.css"/>
    <link rel="stylesheet" href="/workspace/css/header.css"/>
    <link rel="stylesheet" href="/workspace/css/footer.css"/>
    <!-- endbuild -->

    <!-- build:css ./css/home.min.css -->
    <link rel="stylesheet" href="/workspace/css/home.css"/>
    <!-- endbuild -->
    
    <!-- build:js ./js/framework.min.js -->
    <script src="/bower_components/es5-shim/es5-shim.js"></script>
    <script src="/bower_components/base64/base64.js"></script>
    <script src="/bower_components/console-js/console.js"></script>
    <script src="/bower_components/underscore/underscore.js"></script>
    <script src="/bower_components/jquery/jquery.js"></script>
    <script src="/bower_components/jquery-cookie/jquery.cookie.js"></script>
    <script src="/bower_components/Placeholders.js/lib/utils.js"></script>
    <script src="/bower_components/Placeholders.js/lib/main.js"></script>
    <script src="/bower_components/Placeholders.js/lib/adapters/placeholders.jquery.js"></script>
    <script src="/framework/js/lib/lib.jquery.jsonp.js"></script>
    <script src="/framework/js/lib/lib.md5.js"></script>
    <script src="/framework/js/util/util.js"></script>
    <script src="/framework/js/util/valid/isValidEmail.js"></script>
    <script src="/framework/js/util/url/getUrlParam.js"></script>
    <script src="/framework/js/util/url/toUrlParam.js"></script>
    <script src="/framework/js/util/tpl/getTpl.js"></script>
    <script src="/framework/js/util/string/capitalize.js"></script>
    <script src="/framework/js/util/browser/browser_os.js"></script>
    <script src="/framework/js/util/analytics/analytics.js"></script>
    <script src="/workspace/js/sys/server.js"></script>
    <script src="/workspace/js/sys/i18n.js"></script>
    <script src="/workspace/js/sys/util.js"></script>
    <script src="/workspace/js/module/base.js"></script>
    <!-- endbuild -->
</head>
<body>
<script id="index-header">
    document.getElementById("index-header").outerHTML = util.getTemplate('common/header');
</script>

<script id="index-body">
    document.getElementById("index-body").outerHTML = util.getTemplate('home/index');
</script>

<script id="index-footer">
    document.getElementById("index-footer").outerHTML = util.getTemplate('common/footer');
</script>

<!-- 这里是该页面独有的 js 文件 -->
<!-- build:js ./js/home.min.js -->
<script src="/workspace/js/module/home.js"></script>
<!-- endbuild -->
<script>
    if (navigator.userAgent.indexOf("PhantomJS") < 0) {
        var at = document.createElement('script');
        at.src = ('https:' == document.location.protocol ? 'https' : 'http') +
                '://s7.addthis.com/js/300/addthis_widget.js#pubid=ra-xxx';
        at.type = 'text/javascript';
        at.async = 'true';
        var s = document.getElementsByTagName('script')[0];
        s.parentNode.insertBefore(at, s);
    }
</script>
<!-- build:remove -->
<script src="//localhost:35729/livereload.js"></script>
<script src="/workspace/js/debug.js"></script>
<!-- endbuild -->
</body>
</html>
{% endcodeblock %}
可以看到 body 里面都是用 ajax 去加载模板的。而且要注意一点的是，有些资源是不用在phantom 阶段请求的，比如上述请求第三方分享的组件，这个在静态化编译的时候，不会用到。所以要去判断当前是否是phantom打包，如果是的话，就跳过。
```
navigator.userAgent.indexOf("PhantomJS")
```
同时在进行静态化编译的时候，也会生成一些不在预期的代码，这些都要在 finish 的时候去掉, 所以后面grunt的任务就会这么写：
循环对html文件进行一一预编译，并在最后结束的时候，删掉一些不用的代码，或者是状态变化的代码，并且填上一些代码，比如对应的语言包js文件
{% codeblock lang:js %}
staticfy: (function () {
    var cfg = {};
    var files;
    _.each(supportLangs, function (lang) {
        files = {};
        _.each(htmlFiles, function (htmlFile) {
            var file = htmlFile.replace('html/', '');
            files['<%= dist_no_version %>/' + lang + '/' + file] = '<%= dist_no_version %>/' + file;
        });
        cfg[lang] = {
            options: {
                cwd: '<%= dist_no_version %>',
                query_string: 'lang=' + lang,
                onfinish: function (str) {
                    return str
                        // 预编译要跟进对应的语言，对 a 标签的相对路径进行替换
                        .replace(/(<a.*?href=['"])(\/.*?)(['"])/gi, '$1/' + lang + '$2$3')
                        // 消除 jsonp 请求产生的 script 标签
                        .replace(/<script.*async="".*?<\/script>/g, '')
                        // 消除 首页加载Google字体请求产生的link标签
                        .replace(/<link.*http:\/\/fonts.googleapis.com.*?>/, '')
                        // 清理首页的图片，重设为隐藏
                        .replace(/(<img class="item-preload.*)style="display:\s*inline.*"(>)/g, '$1$2')
                        // 清理邮件验证页，重设为隐藏
                        .replace(/"(item-verify-send)"/, '"$1 i-hide"')
                        // signin 重置链接地址
                        .replace(/(href=".*signin\/)\?.*(")/, '$1$2')
                        // 重置 header 右侧的 item-actions 和 item-profile 的状态
                        .replace(/"(item-actions|item-profile)"/, '"$1 i-hide"')
                        // 去掉指定的要删除的脚本
                        .replace(/<script.*id="removeScript".*?<\/script>/g, '')
                        // 去掉自动下载的iframe中的url
                        .replace(/(<iframe id="downloadFrame".*?)(src=")(\S+)(".*><\/iframe>)/g, '$1$2$4')
                        // 去掉首页 Facebook twitter g+ 的 addthis widget 中间代码
                        .replace(/<div.*id="_atssh".*?<\/iframe><\/div>/g, '')
                        // 这条规则要特别注意(用于首页清除addthis的样式)，很容易请到别的页面，所以尽量不要在页面内加上style标签
                        .replace(/<style.*type="text\/css".*?<\/style><\/head>/g, '</head>')
                        // 最后把对应的语言文件再加到里面去
                        .replace(/<\/head>/, '<script src="/' + version + '/lang/' + supportLangObj[lang].code + '.js"></script></head>');
                }
            },
            files: files
        };
    });
    return cfg;
})(),
{% endcodeblock %}
## 缺陷
通过这种方式，虽然可以对模板进行静态预编译，但是还有几个缺陷。
1. 要去下载 phantomJs 的exe 程序
2. 加载编译的速度比较慢，而且有时候会卡死
3. 功能太过复杂了，其实可以简单一点。

后面会通过用gulp进行这个组件的重写，来解决这些问题。






