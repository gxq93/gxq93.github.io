---
title: WKWebview的坑
date: 2016-05-13 14:56:05
tags:
toc: true
---

本文纯属个人学习资料

<!--more-->

# WKWebView 特性
* ``WKWebView``是现代``WebKit API``在``iOS8``和``OS X Yosemite``应用中的核心部分。它代替了``UIKit``中的``UIWebView``和``AppKit``中的``WebView``，提供了统一的跨双平台API。
* 自诩拥有60fps滚动刷新率、内置手势、高效的app和web信息交换通道、和Safari相同的JavaScript引擎
* 将``UIWebViewDelegate``与``UIWebView``重构成了14类与3个协议

# WKWebKit Framework
## Classes
* ``WKBackForwardList``: 之前访问过的web页面的列表，可以通过后退和前进动作来访问到。
* ``WKBackForwardListItem``: ``webview``中后退列表里的某一个网页。
* ``WKFrameInfo``: 包含一个网页的布局信息。
* ``WKNavigation``: 包含一个网页的加载进度信息。
* ``WKNavigationAction``: 包含可能让网页导航变化的信息，用于判断是否做出导航变化。
* ``WKNavigationResponse``: 包含可能让网页导航变化的返回内容信息，用于判断是否做出导航变化。
* ``WKPreferences``: 概括一个``webview``的偏好设置。
* ``WKProcessPool``: 表示一个web内容加载池。
* ``WKUserContentController``: 提供使用``JavaScript post``信息和注射``script``的方法。
* ``WKScriptMessage``: 包含网页发出的信息。
* ``WKUserScript``: 表示可以被网页接受的用户脚本。
* ``WKWebViewConfiguration``: 初始化``webview``的设置。
* ``WKWindowFeatures``: 指定加载新网页时的窗口属性。
## Protocols
* ``WKNavigationDelegate``: 提供了追踪主窗口网页加载过程和判断主窗口和子窗口是否进行页面加载新页面的相关方法。
* ``WKScriptMessageHandler``: 提供从网页中收消息的回调方法。
* ``WKUIDelegate``: 提供用原生控件显示网页的方法回调。
这里有篇很好的文章介绍了``UIWebViewDelegate``与``UIWebView``的API区别和JS与Swift的对话机制[http://nshipster.cn/wkwebkit/](http://nshipster.cn/wkwebkit/)。

# WKWebView Tips

* ``file:///``无法在tmp目录中工作，只能用``file:``访问``tmp``目录。
[https://github.com/shazron/WKWebViewFIleUrlTest](https://github.com/shazron/WKWebViewFIleUrlTest)有具体例子
* 不能在``Storyboard``或者``Interface Builder``中创建。
* ``HTML <a> tag``带着``target="_blank"``不会响应。
* ``URL Scheme``和 ``AppStore links``无法使用
```
// Using [bendytree/Objective-C-RegEx-Categories](https://github.com/bendytree/Objective-C-RegEx-Categories) to check URL String
#import "RegExCategories.h"
- (void)webView:(WKWebView *)webView decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction decisionHandler:(void (^)(WKNavigationActionPolicy))decisionHandler {
    NSURL *url = navigationAction.request.URL;
    NSString *urlString = (url) ? url.absoluteString : @"";

    // iTunes: App Store link
    if ([urlString isMatch:RX(@"\\/\\/itunes\\.apple\\.com\\/")]) {
        [[UIApplication sharedApplication] openURL:url];
        decisionHandler(WKNavigationActionPolicyCancel);
        return;
    }
    // Protocol/URL-Scheme without http(s)
    else if (![urlString isMatch:[@"^https?:\\/\\/." toRxIgnoreCase:YES]]) {
        [[UIApplication sharedApplication] openURL:url];
        decisionHandler(WKNavigationActionPolicyCancel);
        return;
    }
    decisionHandler(WKNavigationActionPolicyAllow);
}
```

* JS的``alert``, ``confirm``, ``prompt``需要调用``WKUIDelegate``方法
如果你想要展示对话框，你需要执行以下方法

```
webView:runJavaScriptAlertPanelWithMessage:initiatedByFrame:completionHandler:
webView:runJavaScriptConfirmPanelWithMessage:initiatedByFrame:completionHandler:
webView:runJavaScriptTextInputPanelWithPrompt:defaultText:initiatedByFrame:completionHandler:
```

[这里有设置方法](http://qiita.com/ShingoFukuyama/items/5d97e6c62e3813d1ae98)
* ``Basic/Digest/etc``验证输入对话框需要调用``WKNavigationDelegate``方法``webView:didReceiveAuthenticationChallenge:completionHandler:``

例子：

```
- (void)webView:(WKWebView *)webView didReceiveAuthenticationChallenge:(NSURLAuthenticationChallenge *)challenge completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition, NSURLCredential *))completionHandler {
    NSString *hostName = webView.URL.host;
    NSString *authenticationMethod = [[challenge protectionSpace] authenticationMethod];

    if ([authenticationMethod isEqualToString:NSURLAuthenticationMethodDefault]
    || [authenticationMethod isEqualToString:NSURLAuthenticationMethodHTTPBasic]
    || [authenticationMethod isEqualToString:NSURLAuthenticationMethodHTTPDigest]) {

        NSString *title = @"Authentication Challenge";
        NSString *message = [NSString stringWithFormat:@"%@ requires user name and password", hostName];
        UIAlertController *alertController = [UIAlertController alertControllerWithTitle:title message:message preferredStyle:UIAlertControllerStyleAlert];
        [alertController addTextFieldWithConfigurationHandler:^(UITextField *textField) {
            textField.placeholder = @"User";
            //textField.secureTextEntry = YES;
        }];
        [alertController addTextFieldWithConfigurationHandler:^(UITextField *textField) {
            textField.placeholder = @"Password";
            textField.secureTextEntry = YES;
        }];
        [alertController addAction:[UIAlertAction actionWithTitle:@"OK" style:UIAlertActionStyleDefault handler:^(UIAlertAction *action) {
            NSString *userName = ((UITextField *)alertController.textFields[0]).text;
            NSString *password = ((UITextField *)alertController.textFields[1]).text;
            NSURLCredential *credential = [[NSURLCredential alloc] initWithUser:userName password:password persistence:NSURLCredentialPersistenceNone];
            completionHandler(NSURLSessionAuthChallengeUseCredential, credential);
        }]];
        [alertController addAction:[UIAlertAction actionWithTitle:@"Cancel" style:UIAlertActionStyleCancel handler:^(UIAlertAction *action) {
            completionHandler(NSURLSessionAuthChallengeCancelAuthenticationChallenge, nil);
        }]];
        dispatch_async(dispatch_get_main_queue(), ^{
            [self presentViewController:alertController animated:YES completion:^{}];
        });
    }
    else if ([authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust]) {
        // needs this handling on iOS 9
        completionHandler(NSURLSessionAuthChallengePerformDefaultHandling, nil);
        // or, see also http://qiita.com/niwatako/items/9ae602cb173625b4530a#%E3%82%B5%E3%83%B3%E3%83%97%E3%83%AB%E3%82%B3%E3%83%BC%E3%83%89
    }
    else {
        completionHandler(NSURLSessionAuthChallengeCancelAuthenticationChallenge, nil);
    }
}
```
* 多个``WKWebView``之间的``cookie``传递
使用``WKProcessPool``在``webviews``之间进行``cookie``传递

```
self.processPool = [[WKProcessPool alloc] init];
WKWebViewConfiguration *configuration1 = [[WKWebViewConfiguration alloc] init];
configuration1.processPool = self.processPool;
WKWebView *webView1 = [[WKWebView alloc] initWithFrame:CGRectZero configuration:configuration1];
...
WKWebViewConfiguration *configuration2 = [[WKWebViewConfiguration alloc] init];
configuration2.processPool = self.processPool;
WKWebView *webView2 = [[WKWebView alloc] initWithFrame:CGRectZero configuration:configuration2];
...

```
* 无法使用``NSURLProtocol``, ``NSCachedURLResponse``,``NSURLProtocol``
``UIWebView``可以通过``NSURLProtocol``, ``NSCachedURLResponse``,``NSURLProtocol``过滤广告网站和缓存和离线浏览，但是``WKWebView``不能。
* ``Cookie``, ``Cache``, ``Credential``, ``WebKit data``不容易被清除
iOS8
1.和``UIWebView``一样的方法用使用``NSURLCache``和``NSHTTPCookie``来删除``cookies``和``caches``。
2.如果你使用``WKProccessPool``对它重新初始化。
3.在``Library``目录中删除``Cookies``, ``Caches``及``WebKit``的子目录。
4.删除所有``WKWebViews``。
iOS9
```
// Optional data
NSSet *websiteDataTypes = [NSSet setWithArray:@[
    WKWebsiteDataTypeDiskCache,
    WKWebsiteDataTypeOfflineWebApplicationCache,
    WKWebsiteDataTypeMemoryCache,
    WKWebsiteDataTypeLocalStorage,
    WKWebsiteDataTypeCookies,
    WKWebsiteDataTypeSessionStorage,
    WKWebsiteDataTypeIndexedDBDatabases,
    WKWebsiteDataTypeWebSQLDatabases
]];
// All kinds of data
NSSet *websiteDataTypes = [WKWebsiteDataStore allWebsiteDataTypes];
// Date from
NSDate *dateFrom = [NSDate dateWithTimeIntervalSince1970:0];
// Execute
[[WKWebsiteDataStore defaultDataStore] removeDataOfTypes:websiteDataTypes modifiedSince:dateFrom completionHandler:^{
// Done
}];
```

* iOS9上滚动速度bug
在iOS 8以下代码没问题,它可以用更多的惯性滚动。
```
webView.scrollView.decelerationRate = UIScrollViewDecelerationRateNormal;
```

至于iOS 9，没有在``UIScrollView``代理中设置滚动速度，这段代码是没有意义的。``scrollViewWillBeginDragging``

```
- (void)scrollViewWillBeginDragging:(UIScrollView *)scrollView {
    scrollView.decelerationRate = UIScrollViewDecelerationRateNormal;
}
```

* 不能禁用长按链接菜单
CSS: ``-webkit-touch-callout: none; ``和JavaScript: ``document.documentElement.style.webkitTouchCallout='none'``;无法使用。
iOS8.2之后修复了这个bug。
* 有时候捕捉``WKWebView``失败
有时捕捉截图``WKWebView``本身失败,试图捕捉``WKWebView``的``scrollView``属性。不然使用私有API，使用[https://github.com/lemonmojo/WKWebView-Screenshot](https://github.com/lemonmojo/WKWebView-Screenshot)。
* Xcode6.1及以上不能精确的说明``WKWebView``使用的内存
* ``window.webkit.messageHandlers``在有些站点无法使用。
一些网站一定程度上重载了JS的``window.webkit``，为了避免这个问题,你应该在一个网站的内容开始加载之前缓存这个变量。``WKUserScriptInjectionTimeAtDocumentStart``可以帮助你。
* ``cookie``有时候无法保存
``WKWebView``初始化时,它可以设置``cookie``管理地区而不等待这个区域被同步。
* ``WKWebView``的``backForwardList``属性是只读的。
* 很难和``UIWebView``在iOS7及以下共存。
参考翻译自[https://github.com/ShingoFukuyama/WKWebViewTips](https://github.com/ShingoFukuyama/WKWebViewTips)
