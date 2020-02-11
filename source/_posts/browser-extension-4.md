---
title: 浏览器 extension 插件开发系列(04) -- Safari 插件的添加以及调试
date: 2020-02-11 14:41:09
tags: 
- js
- 浏览器插件
- Safari 插件
categories: 
- 前端相关
- 浏览器 extension 插件开发系列
---
## 前言
通过 {% post_link browser-extension-2 %} 和 {% post_link browser-extension-3 %} 可以看到`Chrome`和`Firefox`的安装和调试方式。接下来就是`Safari`了, 本节看下 `Safari` 怎么创建插件。

## 在Safari 扩展创建器导入一个新的扩展
首先要在 `Safari 扩展创建器` 导入一个新的扩展,必须要把这个文件夹，命名为 `safariextensiton` 后缀名的才行，比如`airdroid.safariextension`, 这样扩展器才能识别，并载入。
<!--more-->
即要把 `resources\airdroid\data` 的`data` 目录 改成 `safariextension` 后缀的，比如 `airdroid.safariextension`, 有了这个文件之后，就要导入这个插件包了, 操作是`Safari -> 开发 --> 显示扩展创建器`：

![png](1.png)

点击添加按钮，然后选择添加扩展。选择 `airdroid.safariextension` 这个目录

![png](2.png)

这时候就添加进去了。就可以在导航栏看到了:

![png](3.png)

但是这边可以看到，会有一个警告:

![png](4.png)

显示你没有`Safari 扩展证书`。如果没有这个证书的话，那么就不能上`Safari`的`store商店`。 我们可以参照这个教程 {% post_link browser-extension-5 %} 来操作。

## 扩展器参数
接下来就详细了解一下。扩展创建器的一些参数。
我们知道 , 在 Safari 扩展中 , 是由如下几个部分来提供扩展功能 :
1. 工具栏按钮
2. 扩展工具条
3. 上下文关联菜单
4. 注入 (JavaScript) 脚本
5. 注入样式表单

而 Safari 扩展框架中其实是存在了一条分割线 , 将整个体系结构分为了两个部分 :
1. Safari 应用层 (背景页) : 包含工具条按钮 , 扩展工具条 , 页面标签 , 页面窗口 , 上下文菜单等
2. 页面内容层 : 包含修改页面内容的 JavaScript 和 CSS

而这两部分不能直接调用对方的代码 , 需要通过给对方发送消息 , 再由对方的消息处理方法来调用所需的代码。 当然 , 不是每一个 Safari 扩展都需要做这样的消息调用 , 例如简单的”关闭网页”按钮扩展就不需要和页面内容层交互 , 也就不需要消息调用及代理等处理逻辑了。

而以上这些的配置都是在 Info.plist 文件读取的。是一行key，一行value的方式，所以完整的 `Info.plist` 如下:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>Author</key> ---------- 对应作者
    <string>AirDroid Inc</string>
    <key>Builder Version</key> ---------- 对应版本
    <string>10600.1.25</string>
    <key>CFBundleDisplayName</key> ---------- 对应显示名称
    <string>AirDroid</string>
    <key>CFBundleIdentifier</key> ---------- 对应包标识符
    <string>com.airdroid.safari</string>
    <key>CFBundleInfoDictionaryVersion</key> ---------- 扩展器版本
    <string>6.0</string>
    <key>CFBundleShortVersionString</key> ---------- 显示版本
    <string>1.0</string>
    <key>CFBundleVersion</key> ---------- 包版本
    <string>1</string>
    <key>Chrome</key> ---------- 扩展chrome选项
    <dict>
      <key>Global Page</key> ---------- 扩展全局页面
      <string>global.html</string>
      <key>Popovers</key> ---------- 弹出窗口配置
      <array>
        <dict>
          <key>Filename</key> ---------- 弹出窗口的文件名
          <string>popup.html</string>
          <key>Height</key> ---------- 弹出窗口的高
          <string></string>
          <key>Identifier</key> ---------- 弹出窗口的标识符
          <string>toolbar-popover</string>
          <key>Width</key> ---------- 弹出窗口的宽
          <string></string>
        </dict>
      </array>
      <key>Toolbar Items</key> ---------- 工具栏按钮配置
      <array>
        <dict>
          <key>Identifier</key> ----------- 工具栏按钮标识符
          <string>toolbar-button</string>
          <key>Image</key> ----------- 按钮图标
          <string>images/safari_logo.png</string>
          <key>Label</key> ---------- 工具栏的标签名
          <string>Pushbullet</string>
          <key>Popover</key> ------- 对应弹出窗口的标识符
          <string>toolbar-popover</string>
        </dict>
      </array>
    </dict>
    <key>Content</key> --------- 扩展内容配置，即在页面中注入的脚本
    <dict>
      <key>Scripts</key>
      <dict>
        <key>End</key> ---------- 结束脚本文件名
        <array>
          <string>js/sys/safari-end-script.js</string>
        </array>
      </dict>
      <key>Whitelist</key> ---------- 白名单配置
      <array>
        <string>http://*/*</string>
        <string>https://*/*</string>
      </array>
    </dict>
    <key>Description</key> ----------- 应用的描述
    <string>Access Android phone/tablet from computer remotely and securely. </string>
    <key>DeveloperIdentifier</key> ----------- 开发者id (这个要改成我们自己的)
    <string>J294WP5P2Y</string>
    <key>ExtensionInfoDictionaryVersion</key>
    <string>1.0</string>
    <key>Permissions</key> ------------ 权限配置
    <dict>
      <key>Website Access</key> ------------ 访问权限配置
      <dict>
        <key>Include Secure Pages</key> ---------- 包括安全页面
        <true/>
        <key>Level</key> ----------- 访问级别
        <string>All</string>
      </dict>
    </dict>
    <key>Update Manifest URL</key> ----------- 更新清单地址
    <string>https://update.pushbullet.com/Safari.plist</string>
    <key>Website</key> ----------- 应用对应站点
    <string>https://www.airdroid.com</string>
  </dict>
</plist>
```
完整的截图如下：

![png](5.png)
![png](6.png)
![png](7.png)
![png](8.png)
![png](9.png)

这边有几个细节:
1. `popup.html` 的高宽度，可以不用在plist里面指定
2. 有一个全局文件。叫 `global.html` 这个其实就是`chrome`的`manifest.json`的`background`选项。只不过`chrome`是自己生成的`global`页面。即背景页

![png](10.png)

而`Safari`是要自己建的。只不过是一模一样的: `global.html`:
```javascript
<!DOCTYPE html>
<html>
    <head>
        <meta charset="utf-8">
    </head>
    <title>AirDroid</title>
    <body>
    <script src="js/lib/underscore/underscore.js"></script>
    <script src="js/lib/jquery/jquery.js"></script>
    <script src="js/lib/md5/md5.js"></script>
    <script src="js/lib/des/tripledes.js"></script>
    <script src="js/lib/des/mode.ecb.js"></script>
    <script src="js/lib/e2ee/System.js"></script>
    <script src="js/lib/e2ee/System.IO.js"></script>
    <script src="js/lib/e2ee/System.Text.js"></script>
    <script src="js/lib/e2ee/System.Convert.js"></script>
    <script src="js/lib/e2ee/System.BitConverter.js"></script>
    <script src="js/lib/e2ee/System.BigInt.js"></script>
    <script src="js/lib/e2ee/System.Security.Cryptography.SHA1.js"></script>
    <script src="js/lib/e2ee/System.Security.Cryptography.js"></script>
    <script src="js/lib/e2ee/System.Security.Cryptography.RSA.js"></script>
    <script src="js/lib/e2ee/System.Security.Cryptography.HMACSHA1.js"></script>
    <script src="js/lib/e2ee/System.Security.Cryptography.RijndaelManaged.js"></script>
    <script src="js/util/util.js"></script>
    <script src="js/util/tabs.js"></script>
    <script src="js/util/e2ee.js"></script>
    <script src="js/util/des.js"></script>
    <script src="js/util/localstorage.js"></script>
    <script src="js/util/browser_os.js"></script>
    <script src="js/model/account.js"></script>
    <script src="js/model/file.js"></script>
    <script src="js/model/device.js"></script>
    <script src="js/model/contextmenus.js"></script>
    <script src="js/model/notification.js"></script>
    <script src="js/sys/cache.js"></script>
    <script src="js/sys/events.js"></script>
    <script src="js/sys/baseSocket.js"></script>
    <script src="js/sys/subSocket.js"></script>
    <script src="js/sys/pushManage.js"></script>
    <script src="js/sys/notificationManage.js"></script>
    <script src="js/sys/server.js"></script>
    <script src="js/sys/background.js"></script>
    </body>
</html>
```
## 调试
`Safari` 跟`chrome`一样。一个是跟`chrome`一样的在后台运行的界面, 即`global页面`,一个是普通页面。

而调试也是差不多。对于背景页面。`chrome` 是在 插件页面点击弹出的。

![png](11.png)

而`Safari` 也是在扩展创建器点击弹出的。

![png](12.png)

![png](13.png)

对于普通页面的话。就直接右击选择就行了。

![png](14.png)

![png](15.png)

如果文件有改变的话。直接点击重新载入即可:

![png](16.png)

背景页重新载入。是在扩展器处理的。

![png](17.png)

## 总结
这样子`Safari`插件的添加和调试就差不多这样了。接下来我们就要开始接入具体的业务了。

