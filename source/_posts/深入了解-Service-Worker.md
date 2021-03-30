---
title: 深入了解 Service Worker
date: 2020-03-15 23:05:23
tags:
---

前言
在笔者的项目中，加快页面加载速度一直是提高用户体验的重点，优化页面加载速度的手段也有很多：

- 使用 CDN 网络缓存代码资源
- 使用 webpack 等开发工具合并细小资源，减少 HTTP 请求数量
- 使用 UglifyJS 等工具压缩 Javascript 代码，减少网络传输数据
- 使用 SVG 资源代替 PNG 等图片资源
等等…

 然而上述的优化手段都是通过减少网络请求开销从而达到优化加载页面速度的目的，但对于类似在线文档这类用户较为频繁使用的产品，更加需要关注用户缓存的方向。在此之前，前端页面一直依赖浏览器的缓存管理，因此我们需要更精细化地管理用户的缓存，为了达到这个目的，我们需要使用到 **Service Worker**。

## 什么是 Service Worker
Service Worker 属于其中一个 Web Worker 技术，部分中文文档会将其翻译为 `服务工作线程` ，它是脱离于网页 Javascript 环境的一个独立的线程。因此 Service Worker 的生命周期完全独立于网页，在页面刷新时，依然能存活并且接受页面刷新所发送的事件。Service Worker 主要有三个作用：

1. 拦截网页请求，并返回指定响应
2. 后台同步功能
3. 消息推送功能

后台同步功能是得益于 Service Worker 独特的生命周期，使得网页在用户关闭后，Service Worker 线程依然能存活一定时间，将未成功的网络请求发送到服务器中。

消息推送是 Service Worker 另外一个重要的场景，它使得用户在没有打开网页的情况下，能够收到服务器推送过来的提醒消息。这种功能类似于如今 iOS 上的推送消息。消息推送依赖于浏览器厂商提供的推送服务，不幸的是，Chrome 浏览器的厂商 Google，被隔绝于 GFW 之外，使得 Chrome 的推送服务无法正常使用。

拦截网络请求功能是 Service Worker 使用最为广泛的功能。在 Service Worker 中，它能收到管辖区域内的所有资源发起的网络请求，同时能对这部分的网络请求进行自定义的处理。并且，在 Service Worker 环境中，浏览器提供了一套完备的 Cache API，能够对网络的 `Request` 以及 `Response` 进行保存处理。正式因为这两种技术，使得 Service Worker 能够为网页提供一个离线运行的环境，成为 Progress Web Application（PWA）的核心部分。

## Service Worker 的优势
本地缓存并非什么新的技术，早在 Service Worker 技术出现以前，各种浏览器对于网络请求都有进行缓存处理，为的就是提高网络的响应速度、加快页面加载时间，从而吸引更多的用户使用。而在此之后，为了方面开发者对于缓存的管理，提出了 LocalStorage、Web SQL Database、indexdDB 等技术。对比这些技术，Service Worker 的优势在哪里呢？

### Service Worker 能保存更多的数据
根据 [Wikipedia](https://en.wikipedia.org/wiki/Web_Storage) 上的描述 ，开发者通过 LocalStorage 使用的缓存限制大小为 `5MB-10MB` ，而通过 indexedDB 缓存的大小限制最大为 `50MB` 。
这容量限制对于保存普通用户数据来说已经足够，可是对于缓存脚本文件、样式文件却显得捉襟见肘。
根据 [Google 开发文档](https://developers.google.com/web/fundamentals/instant-and-offline/web-storage/offline-for-pwa)中显示，Service Worker 所能使用的容量大小不再做统一限制，而是由当前电脑的磁盘空间所限制。

![ServiceWorker容量限制.png](https://cdn.iguan7u.cn/image/ServiceWorker容量限制.png)

> 在 iOS Safari 中，Service Worker 保存的数据依然限制 50MB。

### Service Worker 与业务逻辑的耦合更低
不同于 LocalStoage 、indexDB 等需要对不同的模块代码手动处理缓存逻辑，Service Worker 在独立的线程中，统一处理网页所发送的网络请求，能够做到对业务逻辑代码零侵入。

### Service Worker 能在无网络状态下返回页面
Service Worker 处理一个独立的线程，生命周期也独立网页，因此在页面关闭或者是刷新时，Service Worker 能持续地处于存活状态。如果 Service Worker  缓存 HTML 数据，那在其存活的时间内，能够在无网络的环境下返回缓存的页面数据，让网页离线使用成为了可能。

## Service Worker 的生命周期
![LifeCycle.png](https://cdn.iguan7u.cn/image/ServiceWorkerLifeCycle.png)

Service Worker 的生命周期主要在 4 个阶段：

1. 安装
2. 等待
3. 活跃
4. 废弃

在浏览器下载完成 Service Worker 脚本资源后，浏览器会初始化 Worker 线程并加载脚本资源。在完成上述步骤后，浏览器会分发一个 `install` 事件给 Service Worker 线程，等待线程完成 install 事件的相关操作，通常在这个阶段，我们能预加载一下网页所需要的资源。而这个阶段，我们称之为**安装阶段**。

在安装完成后，Service Worker 并没有马上被激活，如果在此之前已经有正在处于激活态 Service Worker 线程，那么新安装的线程会一直处于**等待阶段**，等待前一个工作线程废弃。

如果在此之前没有另外的 Service Worker，那么它将马上进入**活跃状态**。在这个状态下，Service Worker 将会收到页面发起请求所触发的 `fetch` 事件，以及 `push` 和 `sync` 的功能性事件。

如果 Service Worker 在安装阶段预加载网络资源失败了，那个它将会进入**废弃阶段**，这个阶段 Service Worker 并不会生效，也无法监听到网页触发的所有事件。

## Service Worker 的使用流程
Service Worker 的使用方法较为简单，需要经过以下几个步骤：

### 注册 Service Worker 线程
```javascript
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/sw.js').then(function(registration) {
    /* 注册 Service Worker 成功 */
    console.log('ServiceWorker registration successful with scope: ', registration.scope);
  }).catch(function(err) {
    /* 注册 Service Worker 失败 */
    console.log('ServiceWorker registration failed: ', err);
  });
}
```

`navigator.serviceWorker.register` 方法会返回一个 Promise，注册成功后该 Promise 会触发 then 方法，如果不成功会抛出对应异常。

注册成功后，我们可以在 Chrome 中打开 `chrome://inspect/#service-workers` 查看注册成功的 Service Worker。

![ServiceWorkerChrome.png](https://cdn.iguan7u.cn/image/ServiceWorkerChrome.png)

### 在 Service Worker 中监听 Install 事件
```javascript
self.addEventListener('install', function(event) {
  // 收到 install 事件，处理资源预加载的逻辑
});
```

在 install 回调的内部，我们需要执行以下步骤：
1. `caches.open()` 打开缓存仓库。
2. `cache.addAll()` 缓存所需的资源文件。
3. 确认所有需要的资源是否已缓存。

```javascript
// serviceWorker.js
var CACHE_NAME = 'my-site-cache-v1';
var urlsToCache = [
  '/',
  '/styles/main.css',
  '/script/main.js'
];

self.addEventListener('install', function(event) {
  // Perform install steps
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(function(cache) {
        console.log('Opened cache');
        return cache.addAll(urlsToCache);
      })
  );
});
```

需要注意的是，如果 `cache.addAll()` 方法缓存的其中一个资源失败了，会导致 Service Worker 进入废弃模式，直到页面刷新后，Service Worker 重新 install 再次尝试缓存。

### 在 Service Worker 中缓存以及返回请求
在 Service Worker 激活后，它将开始接收网页所发送的 fetch 事件，并且通过 `event.responseWith()` 方法，返回自定义的响应。结合 Cache API 中的 `cache.put()` 、`caches.match()` 方法，我们能够将请求的响应保存起来，在再次收到相同请求时，返回上一次的缓存响应结果。

```javascript
// serviceWorker.js
self.addEventListener('fetch', function(event) {
  event.respondWith(
    // 根据请求 URL，查找缓存是否对应的缓存响应
    caches.match(event.request)
      .then(function(response) {
        if (response) {
			// 判断存在缓存响应，直接返回
          return response;
        }
        return fetch(event.request);
      }
    )
  );
});
```

## Service Worker 使用细节
### Service Worker 必须在 HTTPS 协议下使用
因为 Service Worker 能捕捉其作用域下的所有网络请求，为了保证其不会因为中间人篡改了 serviceWorker 脚本而导致隐私泄漏，W3C 协会规定 Service Worker 必须要在 HTTPS 协议下使用。

### Service Worker 脚本必须与网页处于同一个 Origin  下
脚本资源必须与网页资源处于同一个 Origin 下，导致了脚本资源无法部署到 CDN 独立域名上。
这可能与 Service Worker 的机制有关系：在浏览器访问网页的时候，它会根据网页的 Domain Origin 匹配之前是否已经有注册成功的 Service Worker，如果匹配存在的话，会在发起请求之前先将 Service Worker 线程安装并启动。因此这里严格地要求了 Service Worker 的脚本资源路径。

### Service Worker 的作用域限制在其脚本存放路径上
根绝文档描述，Service Worker 所能捕捉的网络请求跟脚本所在的路径有关。若脚本路径在 `/sw.js` 上，Service Worker 默认能够捕捉到 `/` 范围内的所有网络请求。当前，`register()` 方法能够进一步通过 `scope` 参数限制 Service Worker 的作用域范围：

```javascript
navigator.serviceWorker.register('/sw.js', { scope: '/subdomain/' }).then(function(registration) {
  // Registration was successful
  console.log('ServiceWorker registration successful with scope: ', registration.scope);
}, function(err) {
  // registration failed :(
  console.log('ServiceWorker registration failed: ', err);
});
```

通过上述指定 `scope` 参数的方法，Service Worker 就只会收到 `/subdomain` 下的网络 fetch 事件。

但如果我们将 Service Worker 脚本存放在 `/subdomain/sw.js` 上，那么 Service Worker 默认只会收到 `/subdomain/` 作用域下的网络请求，并不会再收到 `/` 域名下的网络请求。
如果我们强制将 `scope` 参数配置为 `/` 上，就会收到 `register()` 方法的失败响应：
```
ServiceWorker registration failed:  DOMException: Failed to register a ServiceWorker for scope ('https://www.qq.com/subdomain') with script ('https://www.qq.com/subdomain/sw.js'): The path of the provided scope ('/') is not under the max scope allowed ('/subdomain/'). Adjust the scope, move the Service Worker script, or use the Service-Worker-Allowed HTTP header to allow the scope.
```
我们需要在 `sw.js` 脚本的返回头部加入 `Service-Worker-Allowed: '/‘` 才能正确接受 `/` 作用域的网络请求事件。

> 但实际上，笔者在 Chrome/78.0.3904.113 版本上测试，`/subdomain/sw.js` 脚本注册的 Service Worker 不仅无法接收到 `/` 作用域的网络请求，甚至连其合法的 `/subdomain/` 作用域下的网络请求都无法接收。这里并不确定是 Service Worker 规范变更了， 还是 Chrome 浏览器的问题。所以还是建议 Service Worker 脚本**最好配置在网页资源的同级目录下**。