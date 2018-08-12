---
title: Objective-C内部实现浅谈-通知
date: 2015-11-26 15:46:53
tags: [Objective-C,Runtime]
categories: 技术
---
# 简述
对于 NSNotification 的实现，苹果并没有公布，但是既然是基于观察者模式实现的，除去优化的问题，大致也可以用 block 来进行通信实现。本文大致模拟了通知的实现。

<!--more-->

# 观察者模式
定义：定义对象间的一种一对多的依赖关系。当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并自动更新。
应用场景：
* 有两种抽象类型相互依赖。将它们封装在各自的对象中，就可以对它们单独进行改变和复用。
* 对一个对象的改变需要同时改变其他对象，而不知道具体有多少对象有待改变。
* 一个对象必须通知其他对象，而它又不需要知道其他对象是什么。

{% asset_img NSNotification.png %}

# NSNotification的伪实现
## API
1.向通知中心注册观察者:
* (void)addObserver:(id)observer selector:(SEL)aSelector name:(nullable NSString \*)aName object:(nullable id)anObject;(观察者接收到通知后执行任务的代码在发送通知的线程中执行)
* (id )addObserverForName:(nullable NSString \*)name object:(nullable id)obj queue:(nullable NSOperationQueue \*)queue usingBlock:(void (^)(GYNotification \*note))block NS_AVAILABLE(10_6, 4_0);(观察者接收到通知后执行任务的代码在指定的操作队列中执行)

2.通知中心向所有注册的观察者发送通知
* (void)postNotification:(GYNotification \*)notification;
* (void)postNotificationName:(NSString \*)aName object:(nullable id)anObject;
* (void)postNotificationName:(NSString \*)aName object:(nullable id)anObject userInfo:(nullable NSDictionary \*)aUserInfo;

3.观察者收到通知，执行相应代码

## 实现
建两个新类 GYNotification，GYNotificationCenter，这两个类的接口和苹果提供的接口完全一样，我将根据接口提供的功能给出实现代码。 要点是通知中心是单例类，并且通知中心维护了一个包含所有注册的观察者的集合，这里我选择了动态数组来存储所有的观察者，代码如下：
``` objc
+ (GYNotificationCenter *)defaultCenter {
    static GYNotificationCenter *singleton;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        singleton = [[self alloc] initSingleton];
    });
    return singleton;
}

- (instancetype)initSingleton {
    if ([super init]) {
        _observers = [NSMutableArray array];
    }
    return self;
}
```

还定义了一个观察者模型用于保存观察者，通知消息名，观察者收到通知后执行代码所在的操作队列和执行代码的回调，模型源码如下：

``` objc
@interface GYObserverModel : NSObject

@property (nonatomic, strong) id observer;
@property (nonatomic, assign) SEL selector;
@property (nonatomic, copy) NSString *notificationName;
@property (nonatomic, strong) id object;
@property (nonatomic, strong) NSOperationQueue *operationQueue;
@property (nonatomic, copy) OperationBlock block;

@end
```

向通知中心注册观察者，代码如下：
``` objc
- (void)addObserver:(id)observer selector:(SEL)aSelector name:(NSString *)aName object:(id)anObject {
    GYObserverModel *observerModel = [[GYObserverModel alloc] init];
    observerModel.observer = observer;
    observerModel.selector = aSelector;
    observerModel.notificationName = aName;
    observerModel.object = anObject;
    [self.observers addObject:observerModel];
}
- (id <NSObject>)addObserverForName:(NSString *)name object:(id)obj queue:(NSOperationQueue *)queue usingBlock:(void (^)(GYNotification * _Nonnull))block {
    GYObserverModel *observerModel = [[GYObserverModel alloc] init];
    observerModel.notificationName = name;
    observerModel.object = obj;
    observerModel.operationQueue = queue;
    observerModel.block = block;
    [self.observers addObject:observerModel];
    return nil;
}
```
发送通知有三种方式，最终都是调用- (void)postNotification:(GYNotification *)notification，代码如下：
``` objc
- (void)postNotification:(GYNotification *)notification {
    [self.observers enumerateObjectsUsingBlock:^(id  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
        GYObserverModel *observerModel = obj;
        id observer = observerModel.observer;
        SEL selector = observerModel.selector;
        if (!observerModel.operationQueue) {
            [observer performSelector:selector withObject:notification];
        } else {
            NSBlockOperation *operation = [NSBlockOperation blockOperationWithBlock:^{
                observerModel.block(notification);
            }];
            NSOperationQueue *operationQueue = observerModel.operationQueue;
            [operationQueue addOperation:operation];
        }
    }];
}
```

# NSNotification的线程问题
>In a multithreaded application, notifications are always delivered in the thread in which the notification was posted, which may not be the same thread in which an observer registered itself.

在多线程应用中，Notification 在哪个线程中 post，就在哪个线程中被转发，而不一定是在注册观察者的那个线程中。

示例：
``` objectivec
- (void)viewDidLoad {
    [super viewDidLoad];
    NSLog(@"current thread = %@", [NSThread currentThread]);
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(handleNotification:) name:POST_NOTIFICATION object:nil];
}

- (void)viewDidAppear:(BOOL)animated {
    [self postNotificationInBackground];
}

- (void)postNotificationInBackground {
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [[NSNotificationCenter defaultCenter] postNotificationName:POST_NOTIFICATION object:nil userInfo:nil];
    });
}

- (void)handleNotification:(NSNotification *)notification {
    NSLog(@"current thread = %@", [NSThread currentThread]);
}
```
Log:
``` objectivec
current thread = <NSThread: 0x7f8548405250>{number = 1, name = main}
current thread = <NSThread: 0x7f854845b790>{number = 2, name = (null)}
```
>For example, if an object running in a background thread is listening for notifications from the user interface, such as a window closing, you would like to receive the notifications in the background thread instead of the main thread. In these cases, you must capture the notifications as they are delivered on the default thread and redirect them to the appropriate thread.

我们在 Notification 所在的默认线程中捕获这些分发的通知，然后将其重定向到指定的线程中。如果我们希望一个 Notification 的 post 线程与转发线程不是同一个线程，就需要重定向。

一种重定向的实现思路是自定义一个通知队列(注意，不是 NSNotificationQueue 对象，而是一个数组)，让这个队列去维护那些我们需要重定向的 Notification。我们仍然是像平常一样去注册一个通知的观察者，当 Notification 来了时，先看看 post 这个 Notification 的线程是不是我们所期望的线程，如果不是，则将这个 Notification 存储到我们的队列中，并发送一个信号(signal)到期望的线程中，来告诉这个线程需要处理一个 Notification。指定的线程在收到信号后，将 Notification 从队列中移除，并进行处理。

官方示例代码：
``` objectivec
@interface ViewController () <NSMachPortDelegate>

@property (nonatomic) NSMutableArray    *notifications;         /* 通知队列 */
@property (nonatomic) NSThread          *notificationThread;    /* 期望线程 */
@property (nonatomic) NSLock            *notificationLock;      /* 用于对通知队列加锁的锁对象，避免线程冲突 */
@property (nonatomic) NSMachPort        *notificationPort;      /* 用于向期望线程发送信号的通信端口 */

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    NSLog(@"current thread = %@", [NSThread currentThread]);

    /* 初始化 */
    self.notifications = [[NSMutableArray alloc] init];
    self.notificationLock = [[NSLock alloc] init];
    self.notificationThread = [NSThread currentThread];
    self.notificationPort = [[NSMachPort alloc] init];
    self.notificationPort.delegate = self;

    /* 往当前线程的run loop添加端口源 */
    /* 当Mach消息到达而接收线程的run loop没有运行时，则内核会保存这条消息，直到下一次进入run loop */
    [[NSRunLoop currentRunLoop] addPort:self.notificationPort
    forMode:(__bridge NSString *)kCFRunLoopCommonModes];

    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(processNotification:) name:@"TestNotification" object:nil];
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [[NSNotificationCenter defaultCenter] postNotificationName:TEST_NOTIFICATION object:nil userInfo:nil];
    });
}

- (void)handleMachMessage:(void *)msg {
    [self.notificationLock lock];

    while ([self.notifications count]) {
        NSNotification *notification = [self.notifications objectAtIndex:0];
        [self.notifications removeObjectAtIndex:0];
        [self.notificationLock unlock];
        [self processNotification:notification];
        [self.notificationLock lock];
    };

    [self.notificationLock unlock];
}

- (void)processNotification:(NSNotification *)notification {
    if ([NSThread currentThread] != _notificationThread) {
        /* Forward the notification to the correct thread. */
        [self.notificationLock lock];
        [self.notifications addObject:notification];
        [self.notificationLock unlock];
        [self.notificationPort sendBeforeDate:[NSDate date]
                                   components:nil
                                         from:nil
                                     reserved:0];
    }
    else {
        /* Process the notification here; */
        NSLog(@"current thread = %@", [NSThread currentThread]);
        NSLog(@"process notification");
    }
}

@end
```
Log:
``` objectivec
current thread = <NSThread: 0x7ffa4070ed50>{number = 1, name = main}
current thread = <NSThread: 0x7ffa4070ed50>{number = 1, name = main}
process notification
```
这种方案有明显额缺陷，官方文档也对其进行了说明，归结起来有两点：

* 所有的通知的处理都要经过 processNotification 函数进行处理。
* 所有的接听对象都要提供相应的 NSMachPort 对象，进行消息转发。

正是由于存在这样的缺陷，因此官方文档并不建议直接这样使用，而是鼓励开发者去继承 NSNoticationCenter 或者自己去提供一个单独的类进行线程的维护。

# NSNotificationCenter的线程安全性
之所以采取通知中心在同一个线程中 post 和转发同一消息这一策略，应该是出于线程安全的角度来考量的。官方文档告诉我们，NSNotificationCenter 是一个线程安全类，我们可以在多线程环境下使用同一个 NSNotificationCenter 对象而不需要加锁。

我们可以在任何线程中添加/删除通知的观察者，也可以在任何线程中post一个通知。
```objectivec
@interface Poster : NSObject

@end

@implementation Poster

- (instancetype)init {
    self = [super init];
    if (self) {
        [self performSelectorInBackground:@selector(postNotification) withObject:nil];
    }
    return self;
}

- (void)postNotification {
    [[NSNotificationCenter defaultCenter] postNotificationName:TEST_NOTIFICATION object:nil];
}

@end

@interface Observer : NSObject {
    Poster  *_poster;
}

@property (nonatomic, assign) NSInteger i;

@end

@implementation Observer

- (instancetype)init {
    self = [super init];
    if (self) {
        _poster = [[Poster alloc] init];
        [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(handleNotification:) name:TEST_NOTIFICATION object:nil];
    }
    return self;
}

- (void)handleNotification:(NSNotification *)notification {
    NSLog(@"handle notification begin");
    sleep(1);
    NSLog(@"handle notification end");
    self.i = 10;
}

- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
    NSLog(@"Observer dealloc");
}

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    __autoreleasing Observer *observer = [[Observer alloc] init];
}

@end
```
这段代码是在主线程添加了一个 TEST_NOTIFICATION 通知的监听者，并在主线程中将其移除，而我们的 NSNotification 是在后台线程中 post 的。在通知处理函数中，我们让回调所在的线程睡眠1秒钟，然后再去设置属性 i 值。这就会导致程序崩溃：
* 当我们注册一个观察者是，通知中心会持有观察者的一个弱引用，来确保观察者是可用的。
* 主线程调用 dealloc 操作会让 Observer 对象的引用计数减为0，这时对象会被释放掉。
* 后台线程发送一个通知，如果此时 Observer 还未被释放，则会向其转发消息，并执行回调方法。而如果在回调执行的过程中对象被释放了，就会出现上面的问题。

那我们该怎么做呢？这里有一些好的建议：
* 尽量在一个线程中处理通知相关的操作，大部分情况下，这样做都能确保通知的正常工作。不过，我们无法确定到底会在哪个线程中调用 dealloc 方法，所以这一点还是比较困难。
* 注册监听时，使用基于 block 的API。这样我们在 block 还要继续调用 self 的属性或方法，就可以通过 weak-strong 的方式来处理。具体大家可以改造下上面的代码试试是什么效果。
* 使用带有安全生命周期的对象，这一点对象单例对象来说再合适不过了，在应用的整个生命周期都不会被释放。
* 使用代理。

# block方式的NSNotification
为了顺应语法的变化，apple 从 iOS4 之后提供了带有 block 的 NSNotification。使用方式如下：
``` objectivec
- (id<NSObject>)addObserverForName:(NSString *)name
                            object:(id)obj
                             queue:(NSOperationQueue *)queue
                        usingBlock:(void (^)(NSNotification *note))block
```
* 观察者就是当前对象
* queue 定义了 block 执行的线程，nil 则表示 block 的执行线程和发通知在同一个线程
* block 就是相应通知的处理函数

这个 API 已经能够让我们方便的控制通知的线程切换。但是，这里有个问题需要注意。就是其 remove 操作。
``` objectivec
- (void)removeObservers {
    if（_observer）{
        [[NSNotificationCenter defaultCenter]removeObserver:_observer];
    }
}
```
其中 _observer 是 addObserverForName 方式的 api 返回观察者对象。这也就意味着，你需要为每一个观察者记录一个成员对象，然后在 remove 的时候依次删除。试想一下，你如果需要10个观察者，则需要记录10个成员对象，这个想想就是很麻烦，而且它还不能够方便的指定 observer。因此，理想的做法就是自己再做一层封装，将这些细节封装起来。

**通知的移除不一定非要写**

如果在闭包中使用了 weakself，则这个通知所在 NSObject 或者 ViewController 都会释放，就不会再继续调用 weakself 调用的方法。

对于addObserver：这里要分 ViewController 和普通 NSObject 两个说起。

**ViewController**：在调用 ViewController 的 dealloc 的时候，系统会调用``[[NSNotificationCenter defaultCenter]removeObserver:self]``方法，所以如果是在 viewDidLoad 中使用 addObserver 添加监听者的话可以省掉移除。当然，如果是在 viewWillAppear 中添加的话，那么就要在 viewWillDisappear 中自己移除，而且，最好是使用``[[NSNotificationCenter defaultCenter]removeObserver:self name:@"test" object:nil]``，而非``[[NSNotificationCenter defaultCenter]removeObserver:self]``，否则很有可能你会把系统的其他通知也移除了

**普通NSObject**：在 iOS9 之后，NSObject 也会像 ViewController 一样在 dealloc 时调用``[[NSNotificationCenter defaultCenter]removeObserver:self]``方法，在 iOS9 之前的不会调用，需要自己写。

![](http://upload-images.jianshu.io/upload_images/1024259-43227bc7bf1e6a73.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

所以可以这么写：
``` objectivec
if ([[UIDevice currentDevice].systemVersion floatValue] < 9.0) {
    __weak typeof(self) weakSelf;
    [[NSNotificationCenter defaultCenter]addObserverForName:TEST_NOTIFICATION
                                                     object:nil
                                                      queue:[NSOperationQueue mainQueue]
                                                 usingBlock:^(NSNotification *note){
                                                            [weakSelf doSomeThing];
                                                 }];
} else {
    [[NSNotificationCenter defaultCenter]addObserver:self selector:@selector(doSomeThing) name:TEST_NOTIFICATION object:nil];
}
```
在使用类别的时候如果我们添加了通知，那么我们是没有办法在类别里面重写 dealloc 的，如果不移除通知就会出现野指针，这个时候我们就可以在 iOS9 以上使用 addObserver，将通知的移除交给系统，iOS9 一下使用 addObserverForName+weakSelf，虽然通知依然存在，但是不会调用 doSomeThing 方法(当然不能直接在 block 里面写处理过程，因为还是会执行)。