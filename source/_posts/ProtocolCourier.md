---
title: 基于NSProxy的中间件-ProtocolCourier
date: 2017-05-14 21:33:10
tags: [Objective-C]
categories: 技术
---

接着上篇文章我们现在把``NSProxy``应用到一个更实际的例子中，首先介绍一下``ProtocolCourier``是一款设计初衷为了简化一些遵守了很多协议而造成代码量大的类的中间件，他的目的是为了减少一些文件的代码量，把单独每个协议的代码抽离出来。接下来我们大致介绍一下如何实现这个"协议快递员"。

<!--more-->

# 设计

首先第一步我们需要设计一下到底怎么才能做到协议转发，他需要提供哪些接口，需要哪些参数。这里我的设计思路是每个 NSProxy 对象只遵守一个协议，他可以包装所有遵守这个协议或者这个协议所包含的其他协议(譬如 UITableViewDelegate 包含 UIScrollViewDelegate)的类，在实例化每个代理对象后作为例如 TabView 的 delegate  对象，从而起到转发这些代理方法的作用。因此我的头文件是这样定义的：

``` objective-c
#define CreateProtocolCourier(__protocol__, ...) ((ProtocolCourier<__protocol__>*)[ProtocolCourier packageForProtocol:@protocol(__protocol__) withObjects:((NSArray *)[NSArray arrayWithObjects:__VA_ARGS__,nil])])

@interface ProtocolCourier : NSProxy
@property (nonatomic,strong, readonly) Protocol *protocol;
@property (nonatomic,strong, readonly) NSArray *attachedObjects;

+ (instancetype)packageForProtocol:(Protocol*)protocol withObjects:(NSArray*)objects;

@end

```

最外层只需提供一个宏作为初始化方法，参数为这个转发对象所遵守的协议以及真正执行这些代理方法的对象。对外提供两个获取协议以及添加包装的对象的接口。

# 实现

## 初始化

接下来就来着手实现这个功能。首先初始化过程必然需要判断添加的真正需要执行代码的对象是否是包含这个协议。

``` objective-c
- (BOOL)object:(id)object conformsProtocolOrAdoptedByProtocol:(Protocol*)protocol {
    if ([object conformsToProtocol:protocol]) {
        return YES;
    }

    BOOL conforms = NO;

    unsigned int count = 0;
    Protocol * __unsafe_unretained * list = protocol_copyProtocolList(protocol, &count);
    for (NSUInteger i = 0; i < count; i++) {
        Protocol * aProtocol = list[i];

        // NSObject 协议因为所有类都包含了所以直接跳过
        if ([NSStringFromProtocol(aProtocol) isEqualToString:@"NSObject"]) continue;

        if ([self object:object conformsProtocolOrAdoptedByProtocol:aProtocol]) {
            conforms = YES;
            break;
        }
    }
    free(list);

    return conforms;
}
```



这个首先判断这个对象是否响应这个协议，如果不是这个协议则通过``protocol_copyProtocolList``获取这个协议所包含的协议列表进行递归查询时候有包含任何这个"协议链"中任何一个协议，如果一个都不符合则抛异常。

```objective-c
+ (instancetype)packageForProtocol:(Protocol *)protocol withObjects:(NSArray *)objects {
    ProtocolCourier *courier = [[super alloc] initWithProtocol:protocol objects:objects];
    return courier;
}

- (instancetype)initWithProtocol:(Protocol*)protocol objects:(NSArray*)objects {
    _protocol = protocol;

    NSMutableArray * validObjects = [NSMutableArray array];

    NSAssert(objects.count > 0, @"必须添加需要转发的对象");

    for (id object in objects) {
        if ([self object:object conformsProtocolOrAdoptedByProtocol:protocol]) {
            [validObjects addObject:object];
        }
    }

    NSAssert(validObjects.count > 0, @"添加的类并没有遵守%@这个协议或者这个协议所采用的协议", NSStringFromProtocol(protocol));

    _objects = [NSOrderedSet orderedSetWithArray:validObjects];

    return self;
}
```

注意这里``_objects``使用的数据结构是``NSOrderedSet``防止对象重复。

## 转发消息

接着就要实现 NSProxy 最关键的消息转发过程了，自然就是继承那几个满足需求的方法。

* respondsToSelector:(SEL)selector
* (void)forwardInvocation:(NSInvocation *)anInvocation
* (NSMethodSignature *)methodSignatureForSelector:(SEL)selector

首先是找到满足我们需要转发的协议或者代理方法是我们的代理类能够响应。

``` objective-c
- (BOOL)respondsToSelector:(SEL)selector {
    BOOL responds = NO;
    BOOL isRequired = NO;

    struct objc_method_description methodDescription = [self methodDescriptionForSelector:selector isRequired:&isRequired];

    //设计初衷：如果是是 required 方法就必须通过 Courier 来响应
    if (isRequired) {
        responds = YES;
    }
    else if (methodDescription.name != NULL) {
        responds = [self checkIfAttachedObjectsRespondToSelector:selector];
    }

    return responds;
}

- (BOOL)checkIfAttachedObjectsRespondToSelector:(SEL)selector {
    for (id object in self.objects) {
        if ([object respondsToSelector:selector]) {
            return YES;
        }
    }
    return NO;
}
```

这里有一个很关键需要实现的也是找到这个方法是否是在这个"协议链"中的，并且我特意将``required``的协议方法特殊处理，必须要经过代理类并且代理类必须要实现这些 required 方法，否则会抛异常。

``` objective-c
- (struct objc_method_description)methodDescriptionForSelector:(SEL)selector isRequired:(BOOL *)isRequired {
    struct objc_method_description method = {NULL, NULL};

    method = [self methodDescriptionInProtocol:self.protocol selector:selector isRequired:isRequired];

    if (method.name == NULL) {
        unsigned int count = 0;
        Protocol * __unsafe_unretained * list = protocol_copyProtocolList(self.protocol, &count);
        for (NSUInteger i = 0; i < count; i++) {
            Protocol * aProtocol = list[i];

            if ([NSStringFromProtocol(aProtocol) isEqualToString:@"NSObject"]) continue;

            method = [self methodDescriptionInProtocol:aProtocol selector:selector isRequired:isRequired];
            if (method.name != NULL) {
                break;
            }
        }
        free(list);
    }

    return method;
}

- (struct objc_method_description)methodDescriptionInProtocol:(Protocol *)protocol selector:(SEL)selector isRequired:(BOOL *)isRequired {
    struct objc_method_description method = {NULL, NULL};

    //返回一个指定的方法的方法描述结构给定的协议
    method = protocol_getMethodDescription(protocol, selector, YES, YES);
    if (method.name != NULL) {
        *isRequired = YES;
        return method;
    }

    method = protocol_getMethodDescription(protocol, selector, NO, YES);
    if (method.name != NULL) {
        *isRequired = NO;
    }

    return method;
}
```

可以看到获取方法描述的方式与上面判断时候在当前"协议链"的方式一样，都是通过递归来遍历整个"协议链"，最后通过方法描述的名字是否为空来判断实际实现的对象遵守这个协议或者这个协议继承的协议。

能正确判断实现对象包含这个协议并且能拿到方法描述后我们就能准确的将这些方法转发给实际实现的对象了。

``` objective-c
- (void)forwardInvocation:(NSInvocation *)anInvocation {
    SEL selector = [anInvocation selector];

    BOOL isRequired = NO;

    struct objc_method_description methodDescription = [self methodDescriptionForSelector:selector isRequired:&isRequired];

    if (methodDescription.name == NULL) {
        [super forwardInvocation:anInvocation];
        return;
    }

    BOOL someoneResponded = NO;
    for (id object in self.objects) {
        if ([object respondsToSelector:selector]) {
            [anInvocation invokeWithTarget:object];
            someoneResponded = YES;
        }
    }

    NSAssert(!(isRequired && !someoneResponded), @"%@方法未实现", NSStringFromSelector(selector));
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)selector {
    NSMethodSignature * methodSignature;

    BOOL isRequired = NO;
    struct objc_method_description methodDescription = [self methodDescriptionForSelector:selector isRequired:&isRequired];

    if (methodDescription.name == NULL) {
        return [super methodSignatureForSelector:selector];
    }

    methodSignature = [NSMethodSignature signatureWithObjCTypes:methodDescription.types];

    return methodSignature;
}
```

# 使用

虽然这个中间件只有200多行代码，但是使用起来还是挺有意思的，比如一个 ViewController 上有一个 UITableView，我们需要实现 UIScrollViewDelegate，UITableViewDelegate，UITableViewDataSource，我们就可以这样使用:

``` objective-c
@interface ViewController ()
@property (nonatomic, strong) IBOutlet UILabel *offsetLabel;
@property (nonatomic, strong) IBOutlet UITableView *tableView;

@property (nonatomic, strong) GYTableViewDelegate *tableViewDelegate;
@property (nonatomic, strong) GYTableViewDataSource *tableviewDataSource;
@property (nonatomic, strong) GYScrollViewDelegate *scrollViewDelegate;

@property (nonatomic, strong) ProtocolCourier<UITableViewDelegate> *tableViewDelegateCourier;
@property (nonatomic, strong) ProtocolCourier<UITableViewDataSource> *tableViewDataSourceCourier;

@end
- (void)viewDidLoad {
    [super viewDidLoad];

    self.tableViewDelegate = [[GYTableViewDelegate alloc] init];
    self.tableviewDataSource = [[GYTableViewDataSource alloc] init];
    self.scrollViewDelegate = [[GYScrollViewDelegate alloc] init];
    self.scrollViewDelegate.label = self.offsetLabel;

    self.tableViewDelegateCourier = CreateProtocolCourier(UITableViewDelegate, self.tableViewDelegate, self.scrollViewDelegate);
    self.tableViewDataSourceCourier = CreateProtocolCourier(UITableViewDataSource, self.tableviewDataSource);

    self.tableView.delegate = self.tableViewDelegateCourier;
    self.tableView.dataSource = self.tableViewDataSourceCourier;
}
```

这三个实现协议的对象是这样的:

``` objective-c
@interface GYTableViewDelegate : NSObject<UITableViewDelegate>
@end
@interface GYTableViewDataSource : NSObject<UITableViewDataSource>
@end
@interface GYScrollViewDelegate : NSObject<UIScrollViewDelegate>
@property (nonatomic, weak) UILabel * label;
@end
```

我们只要在 .m 文件中实现各自遵守的协议，我们的"协议快递员"就能准确的把"协议快递"发送过来啦。

这里有个问题就是虽然我们这样确实是抽离了很多的 ViewController 的代码，但是很多情况下我们可能需要一些状态是 ViewController 和代理对象共享的，比如上述例子中的 offsetLabel，如果把这些需要共享的控件或者状态也通过 ProtocolCourier 传递过来我觉得有点多余了，所以我就是做了简单的对象赋值，可能不算优美，之后再考虑有没有别的好方法。另外还有一个问题不可避免的就是过度使用的话可能会造成一个"类爆炸"的情况，类太多了有时候可能阅读起来会比较费力，因此需要适当使用。当然了这只是个简单的实现，可能还不能具体应用到工程中，不过就当作学习理解消息转发这个过程吧。

最后这个中间件的代码在[这里](https://github.com/gxq93/ProtocolCourier)。
