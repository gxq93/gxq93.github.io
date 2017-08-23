---
title: WebView缓存策略(NSURLProtocol)
date: 2016-11-09 18:13:40
tags: [Objective-C,WebView]
categories: 技术
---
**首先说明一下这个方案只能应用于UI Webview，因为 WKWebView 不支持 NSURLProtocol。**
本文主要介绍通过 NSURLProtocol 和 NSURLSession 对 Web 页面进行缓存，其中还有一些地方还没完善，如读取缓存的条件，清理缓存时间，网页异步加载问题和跳转问题，之后有空再研究。
方案的流程图：

{% asset_img flow.png %}

<!--more-->

# NSURLProtocol
首先介绍一下 NSURLProtocol，他是一个抽象类。**Objective-C 其实抽象类并不是一等公民，只是定义了一个类这个类中都是抽象方法需要子类重载，其实作用同 protocol 差不多**。而自定义 NSURLProtocol 个人理解就相当于重新对 iOS 系统默认的加载 URL 方式进行规范，这个 URL 加载包括 NSURL，NSURLRequest，NSURLConnection 和 NSURLSession 等。既然他是一个抽象类，就需要子类重写他所定义规范的抽象方法，针对初始化的是这三个方法：
``` objectivec
/**
@abstract 拦截处理对应的request，如果不打算处理，返回NO，URL Loading System会使用系统默认的行为去处理；如果打算处理，返回YES，需要处理该请求的所有东西，包括获取请求数据并返回给URL Loading System。
*/
+ (BOOL)canInitWithRequest:(NSURLRequest *)request
{
    return YES;
}
/**
@abstract 返回加载的request，可以在这里修改request，比如添加header，修改host等，并返回一个新的request
*/
+ (NSURLRequest *)canonicalRequestForRequest:(NSURLRequest *)request
{
    return request;
}
/**
@abstract 主要判断两个request是否相同，如果相同的话可以使用缓存数据，通常只需要调用父类的实现。
*/
+ (BOOL)requestIsCacheEquivalent:(NSURLRequest *)a toRequest:(NSURLRequest *)b
{
    return [super requestIsCacheEquivalent:a toRequest:b];
}
```
另外要重写``- (void)startLoading``和``- (void)stopLoading``这两个方法，这两个方法分别在开始和取消 load URL 时调用。
这里有一个需要注意的地方，当加载一个 URL 资源的时候，URL Loading System 会询问你自定义的 URLProtocol 是否能处理该请求，因为你返回 YES，所以 URL Loading System 会创建一个自定义的 URLProtocol 实例，然后调用 NSURLSession 去获取数据，这里也会调用 URL Loading System，而你在``+canInitWithRequest:``中又总是返回 YES，这样 URL Loading System 又会创建一个自定义 URLProtocol 实例导致无限循环。
解决的办法就是标记每一个 request，如果是同一个 request 则返回 no，而系统也提供了标记 request 的方法：
``` objectivec
+ (nullable id)propertyForKey:(NSString *)key inRequest:(NSURLRequest *)request;
+ (void)setProperty:(id)value forKey:(NSString *)key inRequest:(NSMutableURLRequest *)request;
+ (void)removePropertyForKey:(NSString *)key inRequest:(NSMutableURLRequest *)request;
```
因此只要你在``- (void)startLoading``中给 request 标记，同时在``+canInitWithRequest:``中判断 request 是否被标记过，如果标记过就返回 no，这样就避免了反复创建 protocol 的问题。
# NSURLSession请求
既然已经完成了自定义 NSURLProtocol，接着就是考虑在什么时候去缓存请求到的数据，很明显在``- (void)startLoading``和``- (void)stopLoading``已经提供给我们了很好的时机。我们可以在 startLoading 时发起一个 NSURLSession 来请求而在 NSURLSession 的协议方法中我们可以对请求到的数据做自己想做的一些事。
我们可以在 protocol 中定义全局变量 cacheData 和 response，在 NSURLSession 请求时获取到请求的数据及响应，并保存到本地。这里我的缓存是归档解档到指定的路径下，并以 request 的 URL 为文件名进行 md5 签名(tips:很多 URL 带“/”是无法生成文件的)。
``` objectivec
- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask didReceiveResponse:(NSURLResponse *)response completionHandler:(void (^)(NSURLSessionResponseDisposition))completionHandler
{
    [self.client URLProtocol:self didReceiveResponse:response cacheStoragePolicy:NSURLCacheStorageNotAllowed];
    completionHandler(NSURLSessionResponseAllow);
    self.cacheData = [NSMutableData data];
    self.response = response;
}

-  (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask didReceiveData:(NSData *)data
{
    /* 下载过程中 */
    [self.client URLProtocol:self didLoadData:data];
    [self.cacheData appendData:data];
}

- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didCompleteWithError:(NSError *)error
{
    /* 下载完成后触发 */
    if (error) {
        [self.client URLProtocol:self didFailWithError:error];
    } else {
    /* 将数据的缓存归档存入到本地文件中 */
    GYWebCacheData *cache = [[GYWebCacheData alloc] init];
    cache.data = [self.cacheData copy];
    cache.timeDate = [NSDate date];
    cache.response = self.response;
    NSLog(@"write in cache = %@ filepath = %@",cache,self.request.URL.absoluteString);
    [NSKeyedArchiver archiveRootObject:cache toFile:[self filePathWithUrlString:self.request.URL.absoluteString]];
    [self.client URLProtocolDidFinishLoading:self];
    }
}
```
这里在每个 URLSession 方法中必须对 client 进行相应的操作，因为这是你自定义的 URLProtocol，因为你必须实现 client 的请求加载。
因为需要有一个缓存时间，我保留了请求的 data，response 以及缓存时间：
``` objectivec
@interface GYWebCacheData : NSObject<NSCoding>
@property (nonatomic, strong) NSDate *timeDate;                /* 缓存时间 */
@property (nonatomic, strong) NSData *data;                    /* 缓存数据 */
@property (nonatomic, strong) NSURLResponse *response;         /* 缓存请求 */
@end
```
在 stratLoading 和 stopLoading 方法中需要对请求进行判断，如果有缓存就直接加载缓存的数据：
``` objectivec
- (void)startLoading
{
    NSString *url = self.request.URL.absoluteString;
    NSLog(@"requst url = %@",url);
    GYWebCacheData *cache = (GYWebCacheData*)[NSKeyedUnarchiver unarchiveObjectWithFile:[self filePathWithUrlString:url]];

    /* 判断是否有缓存，如果有并且在缓存时间内则读缓存 */
    if ([self isUseCahceWithCache:cache]) {
        NSLog(@"catch from cache = %@",cache.response.URL.absoluteString);
        [self.client URLProtocol:self didReceiveResponse:cache.response cacheStoragePolicy:NSURLCacheStorageNotAllowed];
        [self.client URLProtocol:self didLoadData:cache.data];
        [self.client URLProtocolDidFinishLoading:self];
    }
    else {
        NSMutableURLRequest *newRequest = [self setupRequest];
        NSLog(@"catch from sever = %@",newRequest.URL.absoluteString);
        /* 给新的请求加标示防止循环创建protocol */
        [NSURLProtocol setProperty:@YES forKey:GYURLProtocolHandledKey inRequest:newRequest];
        [self setupTask];
    }
}
- (void)stopLoading
{
    [self.downloadTask cancel];
    self.cacheData = nil;
    self.downloadTask = nil;
    self.response = nil;
}
```
在判断是否加载缓存中你可以加上对缓存时间的判断，从而指定加载缓存的时间段。
使用自定义的 URLProtocol 十分简单，只要在你想使用的地方注册这个自定义的 URLProtocol
``` objectivec
[NSURLProtocol registerClass:[GYWebCacheURLProtocol class]];
```
# 讨论
基于这种方案的 web 离线缓存能够满足基本的需求，当然在要求缓存的时机上你可以加上针对无网情况才加载缓存的策略，还有就是这样子的缓存是十分消耗内存的，可以针对性的定时清理缓存，毕竟用户的网也不是一直会断的🌚还有一个问题就是现在web页面的复杂性，很多页面都是异步渲染的，你只有对能加载到的DOM进行缓存，而特别长的页面分页加载的因为没有显示的地方并不会发起请求自然也就缓存不到，这是不可避免的，也是需要取舍的地方。还有就是类似电商秒杀的页面是不断在刷新的，因为如果要页面跳转可能同一个跳转的页面内容都是不一样的，所以也会存在明明之前点进去的页面缓存过了，但是出来再进去并不是这些内容，笔者的前端知识有限，还没去了解过这些前端的机制，有空了再去研究优化一下。
最后贴出[demo](https://github.com/gxq93/GYWebCache)
