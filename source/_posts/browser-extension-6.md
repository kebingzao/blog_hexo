---
title: 浏览器 extension 插件开发系列(06) -- 各浏览器导航栏按钮的配置的点击出现的panel
date: 2020-02-11 15:54:41
tags: 
- js
- 浏览器插件
- Chrome 插件
- Firefox 插件
- Safari 插件
categories: 
- 前端相关
- 浏览器 extension 插件开发系列
---
## 前言
这一节讲的就是导航栏按钮的那个`icon`，以及点击`icon` 所出来的界面是怎么出来的:

![png](1.png)

<!--more-->
## 1. Chrome
`Chrome` 是在`manifest.json`设置的。有一个选项 `browser_action`. 可以设定`icon`和点击所出现的 `popup页面`
```javascript
"browser_action": {
    "default_icon": "images/icon.png",
    "default_popup": "popup.html"
}
```
## 2. Firefox
是在main.js 里面指定, `lib/main.js`:
```javascript
// 显示菜单栏图标
var setUpToolbarButton = function() {
    var icon = {
        '16': self.data.url('images/icon_16.png'),
        '32': self.data.url('images/icon_32.png')
    };

    if (firefoxVersion >= 30) {
        if (!widget) {
            widget = require('sdk/ui/button/toggle').ToggleButton({
                id: 'airdroid-widget',
                label: 'AirDroid',
                icon: icon,
                onChange: function(state) {
                    if (state.checked) {
                        var panel = onToolbarButtonClicked(function() {
                            widget.state('window', { checked: false });
                            panel.destroy();
                        });
                        panel.show({
                            position: widget
                        });
                    }
                }
            });
        } else {
            widget.icon = icon;
        }
    } else 
    if (firefoxVersion == 29) {
        if (!widget) {
            widget = require('sdk/widget').Widget({
                id: 'airdroid-widget',
                label: 'AirDroid',
                contentURL: icon['16'],
                onClick: function(view) {
                    view.panel = onToolbarButtonClicked();
                }
            });
        } else {
            widget.contentURL = icon['16'];
        }
    } else {
        // Set up the toolbar buttons on existing windows
        var enumerator = mediator.getEnumerator('navigator:browser');
        while (enumerator.hasMoreElements()) {
            compatToolbarButton.add(self, enumerator.getNext(), onToolbarButtonClicked, icon['16']);
        }

        // Set up the toolbar button on any new windows that get opened
        windows.browserWindows.on('open', function(window) {
            compatToolbarButton.add(self, mediator.getMostRecentWindow('navigator:browser'), onToolbarButtonClicked, icon['16']);
        });
    }
};

var onToolbarButtonClicked = function(onHide) {
    var options = {
        //'width': 355,
        //'height': 315,
        'contentURL': self.data.url('popup.html'),
        'contentScriptFile': [
            self.data.url('js/lib/jquery/jquery.js'),
            self.data.url('js/lib/jquery/jquery.toast.js'),
            self.data.url('js/lib/jquery/jquery.clipboard.js'),
            self.data.url('js/lib/underscore/underscore.js'),
            self.data.url('js/util/tplHelper.js'),
            self.data.url('js/model/device.js'),
            self.data.url('js/web/base.js'),
            self.data.url('js/web/popup.js'),
            self.data.url('js/lib/bootstrap/js/bootstrap.js')
        ]
    };

    if (onHide) {
        options.onHide = onHide;
    }

    var panel = Panel(options);

    attachListeners(panel);

    if (firefoxVersion >= 29) {
        return panel;
    } else {
        panel.show(mediator.getMostRecentWindow('navigator:browser').document.getElementById('airdroid'));
    }
};

setUpToolbarButton();
```
而这个文件 `main.js` 是在 `bootstrap.js` 调用的， `bootstrap.js` 部分代码:
```javascript
let main = options.mainPath;

require('sdk/addon/runner').startup(reason, {
      loader: loader,
      main: main,
      prefsURI: rootURI + 'defaults/preferences/prefs.js'
    });
```
而这个配置是在 `harness-options` 配置的, `harness-options.json` 部分配置:
```javascript
 "main": "main", 
 "mainPath": "airdroid/main",
 "manifest": {
  "airdroid/main": {
   "docsSHA256": null, 
   "jsSHA256": "777fe06b9d647b297bc014460274c484b5802694c107f2c79031a2e15cddc5bd", 
   "moduleName": "main", 
   "packageName": "airdroid",
   "requirements": {
    "./toolbar-button": "airdroid/toolbar-button",
    "chrome": "chrome", 
    "sdk/clipboard": "sdk/clipboard", 
    "sdk/context-menu": "sdk/context-menu", 
    "sdk/notifications": "sdk/notifications", 
    "sdk/page-mod": "sdk/page-mod", 
    "sdk/page-worker": "sdk/page-worker", 
    "sdk/panel": "sdk/panel", 
    "sdk/request": "sdk/request", 
    "sdk/self": "sdk/self", 
    "sdk/simple-prefs": "sdk/simple-prefs", 
    "sdk/simple-storage": "sdk/simple-storage", 
    "sdk/system": "sdk/system", 
    "sdk/tabs": "sdk/tabs", 
    "sdk/timers": "sdk/timers", 
    "sdk/ui/button/toggle": "sdk/ui/button/toggle", 
    "sdk/widget": "sdk/widget", 
    "sdk/window/utils": "sdk/window/utils", 
    "sdk/windows": "sdk/windows"
   }, 
   "sectionName": "lib"
  }, 
```
你会发现，这样其实很麻烦，互相调用。后面有做了一些简化。

## 3. Safari
`Safari` 跟`Chrome`差不多，是直接在 `Info.plist`里面配置的, 部分代码如下:
```javascript
<key>Popovers</key>
<array>
  <dict>
    <key>Filename</key>
    <string>popup.html</string>
    <key>Height</key>
    <string></string>
    <key>Identifier</key>
    <string>toolbar-popover</string>
    <key>Width</key>
    <string></string>
  </dict>
</array>
<key>Toolbar Items</key>
<array>
  <dict>
    <key>Identifier</key>
    <string>toolbar-button</string>
    <key>Image</key>
    <string>images/safari_logo.png</string>
    <key>Label</key>
    <string>AirDroid</string>
    <key>Popover</key>
    <string>toolbar-popover</string>
  </dict>
</array>
```

































