---
title: WKWebView使用Tips
date: 2016-02-04 13:00:14
tags:
cover: http://desk.fd.zol-img.com.cn/t_s960x600c5/g5/M00/03/03/ChMkJ1bVPASIRLpXABShkzlZnEgAAMLkAKgToEAFKGr133.jpg
---
* WKWebView是现代WebKit API在iOS8和OS X Yosemite应用中的核心部分。它代替了UIKit中的UIWebView和AppKit中的WebView，提供了统一的跨双平台API。
* 自诩拥有60fps滚动刷新率、内置手势、高效的app和web信息交换通道、和Safari相同的JavaScript引擎
* 将UIWebViewDelegate与UIWebView重构成了14类与3个协议

本文记录了一些相关的注意事项

# WKWebKit Framework
## Classes
* WKBackForwardList: 之前访问过的web页面的列表，可以通过后退和前进动作来访问到。
* WKBackForwardListItem: webview中后退列表里的某一个网页。
* WKFrameInfo: 包含一个网页的布局信息。
* WKNavigation: 包含一个网页的加载进度信息。
* WKNavigationAction: 包含可能让网页导航变化的信息，用于判断是否做出导航变化。
* WKNavigationResponse: 包含可能让网页导航变化的返回内容信息，用于判断是否做出导航变化。
* WKPreferences: 概括一个webview的偏好设置。
* WKProcessPool: 表示一个web内容加载池。
* WKUserContentController: 提供使用JavaScript post信息和注射script的方法。
* WKScriptMessage: 包含网页发出的信息。
* WKUserScript: 表示可以被网页接受的用户脚本。
* WKWebViewConfiguration: 初始化webview的设置。
* WKWindowFeatures: 指定加载新网页时的窗口属性。

## Protocols
* WKNavigationDelegate: 提供了追踪主窗口网页加载过程和判断主窗口和子窗口是否进行页面加载新页面的相关方法。
* WKScriptMessageHandler: 提供从网页中收消息的回调方法。
* WKUIDelegate: 提供用原生控件显示网页的方法回调。
这里有篇很好的文章介绍了UIWebViewDelegate与UIWebView的API区别和JS与Swift的对话机制[http://nshipster.cn/wkwebkit/](http://nshipster.cn/wkwebkit/)。

# WKWebView Tips

* ``file:///``无法在tmp目录中工作，只能用``file:``访问tmp目录。
[https://github.com/shazron/WKWebViewFIleUrlTest](https://github.com/shazron/WKWebViewFIleUrlTest)有具体例子
* 不能在Storyboard或者Interface Builder中创建。
* ``HTML <a> tag``带着``target="_blank"``不会响应。
* URL Scheme和 AppStore links无法使用
```objc
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

* JS的alert, confirm, prompt需要调用WKUIDelegate方法
如果你想要展示对话框，你需要执行以下方法

```objc
webView:runJavaScriptAlertPanelWithMessage:initiatedByFrame:completionHandler:
webView:runJavaScriptConfirmPanelWithMessage:initiatedByFrame:completionHandler:
webView:runJavaScriptTextInputPanelWithPrompt:defaultText:initiatedByFrame:completionHandler:
```

[这里有设置方法](http://qiita.com/ShingoFukuyama/items/5d97e6c62e3813d1ae98)
* ``Basic/Digest/etc``验证输入对话框需要调用WKNavigationDelegate方法``webView:didReceiveAuthenticationChallenge:completionHandler:``

例子：

```objc
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
* 多个WKWebView之间的cookie传递
使用WKProcessPool在webviews之间进行cookie传递

```objc
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
* 无法使用NSURLProtocol, NSCachedURLResponse,NSURLProtocol
UIWebView可以通过NSURLProtocol, NSCachedURLResponse,NSURLProtocol过滤广告网站和缓存和离线浏览，但是WKWebView不能。
* Cookie, Cache, Credential, WebKit data不容易被清除
iOS8
1.和UIWebView一样的方法用使用NSURLCache和NSHTTPCookie来删除cookies和caches。
2.如果你使用WKProccessPool对它重新初始化。
3.在Library目录中删除Cookies, Caches及WebKit的子目录。
4.删除所有WKWebViews。
iOS9
```objc
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
```objc
webView.scrollView.decelerationRate = UIScrollViewDecelerationRateNormal;
```
至于iOS 9，没有在UIScrollView代理中设置滚动速度，这段代码是没有意义的。
```objc
- (void)scrollViewWillBeginDragging:(UIScrollView *)scrollView {
    scrollView.decelerationRate = UIScrollViewDecelerationRateNormal;
}
```
* 不能禁用长按链接菜单
CSS: ``-webkit-touch-callout: none; ``和JavaScript: ``document.documentElement.style.webkitTouchCallout='none';``无法使用。
iOS8.2之后修复了这个bug。
* 有时候捕捉WKWebView失败
有时捕捉截图WKWebView本身失败,试图捕捉WKWebView的scrollView属性。不然使用私有API，使用[https://github.com/lemonmojo/WKWebView-Screenshot](https://github.com/lemonmojo/WKWebView-Screenshot)。
* Xcode6.1及以上不能精确的说明WKWebView使用的内存
* window.webkit.messageHandlers在有些站点无法使用
一些网站一定程度上重载了JS的window.webkit，为了避免这个问题,你应该在一个网站的内容开始加载之前缓存这个变量。``WKUserScriptInjectionTimeAtDocumentStart``可以帮助你。
* cookie有时候无法保存
WKWebView初始化时,它可以设置cookie管理地区而不等待这个区域被同步。
* WKWebView的backForwardList属性是只读的。
* 很难和UIWebView在iOS7及以下共存。
参考翻译自[https://github.com/ShingoFukuyama/WKWebViewTips](https://github.com/ShingoFukuyama/WKWebViewTips)
