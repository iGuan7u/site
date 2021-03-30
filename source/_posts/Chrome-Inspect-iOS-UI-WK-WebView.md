---
title: Chrome Inspect iOS (UI/WK)WebView
date: 2020-01-03 15:46:03
tags:
---

## 被 Safari 支配的恐惧
![Safari_Icoon](https://cdn.iguan7u.cn/image/safari_icon_240.png)
曾几何时，Safari 也是一个相当优秀的浏览器：启动迅速，资源占用少，能耗影响低，同时能跟 macOS 中其他的服务无缝的配合着：Keychain,  TouchID, handOff。即便全世界都高呼 Chrome 宇宙第一浏览器，笔者一直都是 Safari 的忠实用户，不离不弃。
可是不知道什么时候开始，Safari 也主打性能了，渲染速度、JavaScript 运行速度，都出现在了发布会 Keynote 上。普通用户对性能提升当然欢呼雀跃，可是对于经常不插电源使用 macBook 的笔者，我也慢慢留意到 Safari 开始频繁出现在**使用大量能耗**的应用列表上了。
当然，只是使用大量能耗，也谈不上恐惧。真正恐惧的，是作为一名 iOS 开发者，Safari 基本上是你离不开的工具，在调试 iOS 的 UIWebView、WKWebView、MobileSafari，Safari **曾经**是你唯一可靠的工具，而如今，添加 CSS 属性异常，Source 无法展示部分较大的 js 文件，甚至于 WebInspector 直接 Crash，弄得民怨沸腾的。

## ios-webkit-debug-proxy
[ios-webkit-debug-proxy](https://github.com/google/ios-webkit-debug-proxy) 是我在无意间划水 Github 时发现的一个代理工具，它借助于 [usbmuxd](https://github.com/libimobiledevice/usbmuxd) 工具所提供的功能，将调试命令发送到 iOS 设备的 com.apple.webinspector 服务中，从而达到不使用 Safari 调试 iOS WebView 的目的。

借用项目中的示意图：

![](https://cdn.iguan7u.cn/image/ios-webkit-debug-proxy.png)

### 安装依赖：

- [homebrew](http://brew.sh/)

### 安装方法：

> 为了避免出现 `Could not connect to lockdownd, error code -<number>. Exiting.` 错误，建议不要直接 `brew install ios-webkit-debug-proxy` 安装。  

```shell
$: brew update
$: brew uninstall --force libimobiledevice ios-webkit-debug-proxy usbmuxd
$: brew install --HEAD usbmuxd
$: brew install --HEAD libimobiledevice
$: brew install -s ios-webkit-debug-proxy
```

### 使用方法：

```shell
$: ios-webkit-debug-proxy
```

> ios-webkit-debug-proxy 默认使用 9221 端口，然后从 9222-9322 会递增地分配给每一台连接成功的 iOS 设备。  

连接电脑后，确认已经通过 iTunes 信任设备后，Terminal 会显示出设备名以及对应的端口号，或者直接打开 [localhost:9221](h) 查看连接设备列表。

```shell
$: ios-webkit-debug-proxy
Listing devices on :9221
Connected :9222 to “iGuan7u” iPad (aaaaaaaaaaaaaaaaaaaaaaaaaab7e05)
Disconnected :9222 from “iGuan7u”  iPad (aaaaaaaaaaaaaaaaaaaaaaaaaab7e05)
```

## Chrome 拯救世人
当然，我们的目的不是启动 ios-webkit-debug-proxy 来查看 iOS 的设备名以及 UUID 的，而是希望脱离 Safari 的魔爪，奔向自由的 Chrome DevTool。

1. 我们就需要启动 Chrome
2. 打开 [Inspector](chrome://inspect) 页面
3. 点击 `Discover network targets` 后的 `configure` 按钮
4. 输入上面展示的设备端口地址：`localhost:9222` ，点击 `done` 按钮
5. 在页面的 Remote Target 中即可看到你连入的设备了，开始你的调试吧～

不过 Inspect 页面可能没有直接展示 WebView 的截屏，这里可以使用另外一个开源工具 [RemoteDebug iOS WebKit Adapter](https://github.com/RemoteDebug/remotedebug-ios-webkit-adapter)，这个使用 ios-webkit-debug-proxy 做底层的一个工具。

## Bug
笔者自测，在 iOS13 中，inspect 页面似乎没有正确展示 Element 等信息，Console Tab 也没有打出任何日志信息，估计是工具的问题，期待后期 Google 方（没错，这个工具是 Google 的～）的解决吧。