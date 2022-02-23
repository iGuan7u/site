---
title: WKWebView 设置 Cookie 的正确方式
date: 2021-10-10 15:54:00
tags:
---

## 1. 前言：

> WKWebView 万般好，Cookie 操作对比 UIWebView，仿佛就是处于原始时代一般。

笔者在开发过程中遇到了一个问题，在 Server 端返回的 302 响应中，响应头部的 `Set-Cookie` 字段全部没有生效。~~深究下去后，掀开了 WKWebView 英俊的外表下，及其丑恶的嘴脸。~~

## 2. 问题

笔者的场景中，在使用 WKWebView 打开的页面，Server 端需要校验 Client 的身份。因此需要在打开网页前，注入当前用户的身份信息。由于 Cookie 支持 Request 的其中一个头部字段，因此可以很简单的操作为：

```objectivec
NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
[request addValue:cookieValue forHTTPHeaderField:@"Cookie"];
[webView loadRequest:request];
```

然后，他就生效了，Server 能正确获取到 Cookie 数据，问题得以解决。

直到，其后的需求变更，其中一种场景下，Request 会触发一个 HTTP 状态码为 302 的**跨域**跳转，同时，Response 中会通过 `Set-Cookie` 字段注入目的地址所需的身份信息。

然后，跳转失败，302 后的 Request 并没有带上该有的 Cookie 数据。

## 3. 原因

究其根本，笔者认为该是 WKWebView 所处的是**独立进程**的原因。
众所周知，不同于 UIWebView，WKWebView 所在的进程是独立于 App 的进程的，因此 WKWebView 无法像 UIWebView 一样轻松获取应用的 NSHTTPCookieStorage 的单例对象，转而需要进程间通讯传递所需的参数。

在 [这才是 WKWebView Cookie 管理的正确方式](https://www.jianshu.com/p/163c03ed0b5e) 文章中说到：

> 结合两者，你也会发现一个核心的概念-**如果设置了 allHTTPHeaderFields，则不用使用 the cookie manager by default**。

通过 NSMutableURLRequest 修改 Header 字段，默认不再需要 WKWebView 自动管理 Cookie 信息，因此在笔者的场景中，302 后的 Cookie 由于没有自行管理，因此便被丢弃。

## 4. 解决方法

在 [这才是 WKWebView Cookie 管理的正确方式](https://www.jianshu.com/p/163c03ed0b5e) 文章中提到，在设置 Cookie 的时候，应该避免通过 NSMutableURLRequest 设置 Cookie 信息。然后该作者提供了一种通过 Javascript 方式注入 Cookie：

```objectivec
WKUserContentController *userContentController = [WKUserContentController new];

WKWebViewConfiguration *webViewConfig = [[WKWebViewConfiguration alloc] init];
    webViewConfig.userContentController = userContentController;

webViewConfig.processPool = [AppHostCookie sharedPoolManager];
    
NSMutableArray<NSString *> *oldCookies = [AppHostCookie cookieJavaScriptArray];
    [oldCookies enumerateObjectsUsingBlock:^(NSString *_Nonnull obj, NSUInteger idx, BOOL *_Nonnull stop) {
    NSString *setCookie = [NSString stringWithFormat:@"document.cookie='%@';", obj];
    WKUserScript *cookieScript = [[WKUserScript alloc] initWithSource:setCookie injectionTime:WKUserScriptInjectionTimeAtDocumentStart forMainFrameOnly:YES];
    [userContentController addUserScript:cookieScript];
}];

WKWebView *webview = [[WKWebView alloc] initWithFrame:CGRectMake(0, -1, SCREEN_WIDTH,ONE_PIXEL) configuration:webViewConfig];

webview.navigationDelegate = self;
webview.UIDelegate = self;

return webview;
```

这种方式其实是**不准确**的，由于 JS 注入 Cookie 的时机是 	`WKUserScriptInjectionTimeAtDocumentStart` ，这意味着需要等 WKWebView 发起了首次请求，获取到 HTML 数据的时候，Cookie 并没有注入。这种方式只适用于获取 HTML 的 Request 不校验 Cookie，只是其中的 Ajax 才需要 Cookie 信息的场景。

笔者认为，正确的应该是通过 iOS 11.0 版本引入的 **WKHTTPCookieStore** 成员去注入 Cookie 信息到 WKWebView 中

```objectivec
// 建议自行初始化 DataSource，使用 WKWebView 默认的有几率不生效
WKWebsiteDataStore *dataStore = [WKWebsiteDataStore nonPersistentDataStore];
WKWebViewConfiguration *wkWebConfig = [[WKWebViewConfiguration alloc] init];
wkWebConfig.websiteDataStore = dataStore;

WKWebView *webview = [[WKWebView alloc] initWithFrame:CGRectZero configuration:wkWebConfig];
// 获取 Cookie Store 对象
WKHTTPCookieStore *cookieStore = webView.configuration.websiteDataStore.httpCookieStore;

// 由于设置 Cookie 应该是跨进程通讯，这里需要等待设置完成后，再发起请求
dispatch_group_t cookieGroup = dispatch_group_create();
for (NSHTTPCookie *cookie in [[NSHTTPCookieStorage sharedHTTPCookieStorage] cookiesForURL:request.URL]) {
    dispatch_group_enter(cookieGroup);
    [cookieStore setCookie:cookie completionHandler:^{
        dispatch_group_leave(cookieGroup);
    }];
}

dispatch_group_notify(cookieGroup, dispatch_get_main_queue(), ^{
    [webView loadRequest:request];
});
```

另外对于从 WKWebView 同步 Cookie 到 NSHTTPCookieStorage 的场景，可以通过增加 WKHTTPCookieStore 的 `Observer` ，实现代理 `<WKHTTPCookieStoreObserver> (cookiesDidChangeInCookieStore:)`  事件，即可将变更的 Cookie 同步回 NSHTTPCookieStorage 中。