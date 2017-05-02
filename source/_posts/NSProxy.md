---
title: NSProxy浅谈
date: 2017-02-04 11:32:37
tags: [Objective-C]
categories: 技术
thumbnail: http://7xtg0o.com1.z0.glb.clouddn.com/1-Bnx0KZgPIBQ75F5rjLx70g.png
---
>NSProxy is an abstract superclass defining an API for objects that act as stand-ins for other objects or for objects that don’t exist yet. Typically, a message to a proxy is forwarded to the real object or causes the proxy to load (or transform itself into) the real object. Subclasses of NSProxy can be used to implement transparent distributed messaging (for example, NSDistantObject) or for lazy instantiation of objects that are expensive to create.

NSProxy是一个抽象父类定义一个API的对象充当其他对象或对象的替身。通常,一个消息代理转发给真正的对象或导致代理负载或转型为真正的对象。子类NSProxy可用于实现透明的分布式消息(例如NSDistantObject)或延迟实例化的对象。

>A concrete subclass must therefore provide an initialization or creation method and override the forwardInvocation: and methodSignatureForSelector: methods to handle messages that it doesn’t implement itself.

创建合适的NSProxy子类，通过复写forwardInvocation:方法和 methodSignatureForSelector:方法，将NSProxy不识别的调用forward给具体负责实现的对象。

<!--more-->

NSObject 是绝大多数iOS应用的根Class，通过``NS_ROOT_CLASS``宏可以自己注册一个新的根Class。除了NSObject之外，iOS SDK还内置了另外一个根Class，那就是NSProxy。二者都是Foundation框架中的基类, 并且都实现了<NSObject>这个接口。单从名字上来看，就能够大致理解NSProxy设计用于代理方法。也就是说，NSProxy本身（设计上不提供方法的实现，而是将调用转发给实际执行该方法的对象。

# 与NSObject对比
通过二者来创建代理类的最基本实现代码:

继承自NSProxy
``` objectivec
@interface ProxyA : NSProxy

@property (nonatomic, strong) id target;

@end

@implementation ProxyA

- (id)initWithObject:(id)object {
    self.target = object;
    return self;
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)selector {
    return [self.target methodSignatureForSelector:selector];
}

- (void)forwardInvocation:(NSInvocation *)invocation {
    [invocation invokeWithTarget:self.target];
}

@end
```

继承自NSObject
``` objectivec
@interface ProxyB : NSObject

@property (nonatomic, strong) id target;

@end

@implementation ProxyB

- (id)initWithObject:(id)object {
    self = [super init];
    if (self) {
        self.target = object;
    }
    return self;
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)selector {
    return [self.target methodSignatureForSelector:selector];
}

- (void)forwardInvocation:(NSInvocation *)invocation {
    [invocation invokeWithTarget:self.target];
}

@end
```
NSProxy这个基类没有定义默认的init方法。如果我们要获得一个NSProxy的实例，代码只需要这样：``NSProxy *proxyInstance = [NSProxy alloc];``

``methodSignatureForSelector:``方法签名

``forwardInvocation:``消息转发

经测试发现以下两个在<NSObject>中定义的接口, 在二者之间表现是不一致的:
``` objectivec
NSString *string = @"test";
ProxyA *proxyA = [[ProxyA alloc] initWithObject:string];
ProxyB *proxyB = [[ProxyB alloc] initWithObject:string];

NSLog(@"%d", [proxyA respondsToSelector:@selector(length)]);
NSLog(@"%d", [proxyB respondsToSelector:@selector(length)]);

NSLog(@"%d", [proxyA isKindOfClass:[NSString class]]);
NSLog(@"%d", [proxyB isKindOfClass:[NSString class]]);
```
结果会输出完成不同的结论:
``` objectivec
1 0
1 0
```
也就是说通过继承自NSObject的代理类是不会自动转发respondsToSelector:和isKindOfClass:这两个方法的, 而继承自NSProxy的代理类却是可以的. 测试<NSObject>中定义的其它接口二者表现都是一致的。

NSObject的所有Category中定义的方法无法在ProxyB中完成转发，举一个很常见的例子, valueForKey:是定义在NSKeyValueCoding这个NSObject的Category中的方法, 尝试二者执行的表现。
``` objectivec
NSLog(@"%@",[proxyA valueForKey:@"length"]);
NSLog(@"%@",[proxyB valueForKey:@"length"]);
```
这段代码第一句能正确运行, 但第二行却会抛出异常, 分析最终原因其实很简单, 因为valueForKey:是NSObject的Category中定义的方法, 让NSObject具备了这样的接口, 而消息转发是只有当接收者无法处理时才会通过forwardInvocation:来寻求能够处理的对象。因此NSProxy更适合实现做为消息转发的代理类, 因为作为一个抽象类, NSProxy自身能够处理的方法极小(仅<NSObject>接口中定义的部分方法), 所以其它方法都能够按照设计的预期被转发到被代理的对象中。

# 懒加载：
假设说有一个单例类WhatService，调用``+shared``方法会初始化这个单例，提供一个方法``-what``。假设说现在需要拿到这个类的单例，但是有很大概率并不会调用它。那么可以用NSProxy做一层封装。
```objectivec
@interface WhatServiceProxy : NSProxy

@end

@implementation

- (void)forwardInvocation:(NSInvocation *)anInvocation {
    [anInvocation invokeWithTarget:[WhatService shared]];
}

- (NSMethodSignature*)methodSignatureForSelector:(SEL)aSelector {
    return [WhatService instanceMethodSignatureForSelector:aSelector];
}

@end
```
通过 [WhatServiceProxy alloc]可以拿到一个完美代替WhatService的对象，任何对该Proxy的调用都会转移到[WhatService shared]上。
也就是说，对WhatServiceProxy实例调用``-what``等同于调用[[WhatService shared]what]。

# 作为代理类
NSProxy还有一个最重要的用法就是作为抽象代理类，把一些调用的方法来交给NSProxy来调度，而将对象和方法解耦：
``` objectivec
@interface CustomProxy : NSProxy

- (instancetype)initWithObject:(CustomClass*)object;

@property(nonatomic, strong)CustomClass* object;

@end

@implementation CustomProxy
- (instancetype)initWithObject:(CustomClass*)object{
    _object = object;

    /* 映射target及其对应方法名 */
    [self _registerMethodsWithTarget:_object];
    return self;
}

- (void)_registerMethodsWithTarget:(id )target{

    unsigned int numberOfMethods = 0;

    /* 获取target方法列表 */
    Method *method_list = class_copyMethodList([target class], &numberOfMethods);

    for (int i = 0; i < numberOfMethods; i ++) {
    /* 获取方法名并存入字典 */
    Method temp_method = method_list[i];
    SEL temp_sel = method_getName(temp_method);
    const char *temp_method_name = sel_getName(temp_sel);
    [_methodsMap setObject:target forKey:[NSString stringWithUTF8String:temp_method_name]];
    }

    free(method_list);
}

- (void)forwardInvocation:(NSInvocation *)invocation{
    /* 获取当前选择子 */
    SEL sel = invocation.selector;

    /* 获取选择子方法名 */
    NSString *methodName = NSStringFromSelector(sel);

    /* 在字典中查找对应的target */
    id target = _methodsMap[methodName];

    /* 检查target */
    if (target && [target respondsToSelector:sel]) {
        [invocation invokeWithTarget:target];
    } else {
        [super forwardInvocation:invocation];
    }
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel{
    /* 获取选择子方法名 */
    NSString *methodName = NSStringFromSelector(sel);

    /* 在字典中查找对应的target */
    id target = _methodsMap[methodName];

    /* 检查target */
    if (target && [target respondsToSelector:sel]) {
        return [target methodSignatureForSelector:sel];
    } else {
        return [super methodSignatureForSelector:sel];
    }
}
```
而方法调用对象就可以从普通的对象变成了proxy这个代理对象
``` objectivec
CustomProxy *proxy = [[CustomProxy alloc]initWithObject:[CustomClass alloc]init];
[proxy doSomeCustomClassMethod];
```
