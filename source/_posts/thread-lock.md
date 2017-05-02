---
title: 线程锁在iOS中的应用
date: 2016-3-23 13:18:04
tags: [Objective-C]
categories: 技术
thumbnail: http://7xtg0o.com1.z0.glb.clouddn.com/1-34ona0OcnJhWaw53CXmb7w.png
---
在项目中做到线程安全是十分重要的事情，有时候简单的上个线程锁就能很好的完成需求，而给线程上锁的方法有很多，本文大致介绍了几种线程锁的使用。

<!--more-->

# GCD
因为用到线程锁肯定会用到GCD，就简单先写一点GCD相关的一些东西。简单暴力介绍，上代码和图。
界面布局：
```objc
static NSString * const urlStr = @"http://cdn.nshipster.com/images/the-nshipster-fake-book-cover@2x.png";
static NSInteger const imageCount = 15;
static NSInteger const limitCount = 7;
#define height [UIScreen mainScreen].bounds.size.height/5
#define width [UIScreen mainScreen].bounds.size.width/3
/* 界面部署 */
- (void)layoutUI {
    imageView = [NSMutableArray arrayWithCapacity:imageCount];
    for (int y=0; y<5; y++) {
        for (int x=0; x<3; x++) {
            UIImageView *image = [[UIImageView alloc]initWithFrame:CGRectMake(x*width, y*height, width, height)];
            [self.view addSubview:image];
            [imageView addObject:image];
        }
    }
    imageName = [NSMutableArray arrayWithCapacity:imageCount];
    for (int i=0; i<imageCount; i++) {
        [imageName addObject:urlStr];
    }
}
```

图片加载:

```objc
- (void)loadImageAtIndex:(NSInteger)index {
/* 异步请求数据 */
    NSData *data = [self requestDataAtIndex:index];
    /* 切回主线程更新UI */
    dispatch_async(dispatch_get_main_queue(), ^{
        [self updateWithData:data atIndex:index];
    });
}

- (NSData*)requestDataAtIndex:(NSInteger)index {
    NSData *data = [NSData dataWithContentsOfURL:[NSURL URLWithString:imageName[index]]];
    return data;
}

- (void)updateWithData:(NSData*)data atIndex:(NSInteger)index {
    UIImageView *image = imageView[index];
    image.image = [UIImage imageWithData:data];
}

```
首先是串行队列异步任务:

```objc
- (void)serialTap {
    dispatch_queue_t serial = dispatch_queue_create("serialQueue", DISPATCH_QUEUE_SERIAL);
    for (NSInteger i=0; i<imageCount; i++) {
        dispatch_async(serial, ^{
            [self loadImageAtIndex:i];
        });
    }
}
```

运行结果如图，因为是串行队列图片按顺序一张一张加载。

￼![](http://www.z4a.net/images/2016/05/09/1ed39ccc88b0d963.gif)

然后是并行队列:

```objc
- (void)currentTap {
    for (NSInteger i=0; i<imageCount; i++) {
        dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
            [self loadImageAtIndex:i];
    });
    }
}
```
运行结果可以看出图片是并发加载的

![](http://www.z4a.net/images/2016/05/09/fa59e219154607ea.gif)
￼
然后把加载图片的async异步任务改成sync同步任务

```objc
- (void)syncTap {
    for (NSInteger i=0; i<imageCount; i++) {
        dispatch_sync(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
            [self loadImageAtIndex:i];
        });
    }
}

```
运行结果如图，图片同时加载出来

￼![](http://www.z4a.net/images/2016/05/09/sync.gif)

# 线程锁
下面主要介绍一下iOS的线程锁
个人理解使用线程锁的目的就是为了保护资源，防止当前线程上的资源被其他线程抢占，相当于抢红包10个红包20个人抢，不进行保护的话因为速度差不多就可能发生资源分配错乱，而在iOS中线程锁的使用方式其实有很多种。
```objc
/* 如果只要显示9张图片 */
- (void)unlockTap {
    for (NSInteger i=0; i<imageCount; i++) {
        dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
            [self unlockloadImageAtIndex:i];
        });
    }
}
- (void)unlockloadImageAtIndex:(NSInteger)index {
    if (imageName.count>limitCount) {
        NSString *str = [imageName lastObject];
        NSData *data =[NSData dataWithContentsOfURL:[NSURL URLWithString:str]];
        [imageName removeLastObject];
        dispatch_async(dispatch_get_main_queue(), ^{
            UIImageView *image = imageView[index];
            image.image = [UIImage imageWithData:data];
        });
    }
}
```
以上代码原因就是没有进行线程保护，从而导致如下结果没有达到预期需求。

￼![](http://www.z4a.net/images/2016/05/09/22cf0414115ce3b8.gif)
￼
## NSLock
使用NSLock把需要加锁的代码（以后暂时称这段代码为”加锁代码“）放到NSLock的``lock``和``unlock``之间，一个线程A进入加锁代码之后由于已经加锁，另一个线程B就无法访问，只有等待前一个线程A执行完加锁代码后解锁，B线程才能访问加锁代码。另外，demo中数组imageName定义成了成员变量，这么做其实是不明智的，应该定义为“原子属性”。对于被抢占资源来说将其定义为原子属性是一个很好的习惯，因为有时候很难保证同一个资源不在别处读取和修改。``nonatomic``属性读取的是内存数据（寄存器计算好的结果），而``atomic``就保证直接读取寄存器的数据，这样一来就不会出现一个线程正在修改数据，而另一个线程读取了修改之前（存储在内存中）的数据，永远保证同时只有一个线程在访问一个属性。

初始化NSLock
```objc
NSLock *lock = [[NSLock alloc]init];
```

加锁解锁

```objc
- (void)lockloadImageAtIndex:(NSInteger)index {
    /* 加锁 */
    [lock lock];
    if (imageName.count>limitCount){
        NSString *str = [imageName lastObject];
        NSData *data =[NSData dataWithContentsOfURL:[NSURL URLWithString:str]];
        [imageName removeLastObject];
        dispatch_async(dispatch_get_main_queue(), ^{
            UIImageView *image = imageView[index];
            image.image = [UIImage imageWithData:data];
        });
    }
    /* 解锁 */
    [lock unlock];
}
```

运行结果如下图，很好的完成了需求，因为线程加了锁，所以图片是按顺序加载的。

￼![](http://www.z4a.net/images/2016/05/09/74853d81c8d3f6e0.gif)
￼
但是不可忽视的是多线程中很容易发生死锁，如递归。可以使用NSRecursiveLock替换NSLock，它实际上定义的是一个递归锁，这个锁可以被同一线程多次请求，而不会引起死锁。这主要是用在循环或递归操作中。

## GCD信号量
GCD中也已经提供了一种信号机制，使用它我们也可以来构建一把“锁”(从本质意义上讲，信号量与锁是有区别，具体差异为信号量与互斥锁之间的区别):
```objc
dispatch_semaphore_t semaphore = dispatch_semaphore_create(1);
```
```objc
- (void)lockloadImageAtIndex:(NSInteger)index {
    /* 加锁 */
    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);

    if (imageName.count>limitCount){
        NSString *str = [imageName lastObject];
        NSData *data =[NSData dataWithContentsOfURL:[NSURL URLWithString:str]];
        [imageName removeLastObject];
        dispatch_async(dispatch_get_main_queue(), ^{
            UIImageView *image = imageView[index];
            image.image = [UIImage imageWithData:data];
        });
    }
    /* 解锁 */
    dispatch_semaphore_signal(semaphore);
}
```

最后实现的结果和使用NSLock相同
## OSSpinLock

iOS中效率最高的线程锁应该是OSSpinLock，但是这篇文章中有指出它的一些问题[spinlock is unsafe in ios](http://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/?utm_source=tuicool&utm_medium=referral)，它的使用方法与普通线程锁大同小异。

```objc
OSSpinLock spinlock = OS_SPINLOCK_INIT;
```
```objc
- (void)lockloadImageAtIndex:(NSInteger)index {
    /* 加锁 */
    OSSpinLockLock(&spinlock);

    if (imageName.count>limitCount){
        NSString *str = [imageName lastObject];
        NSData *data =[NSData dataWithContentsOfURL:[NSURL URLWithString:str]];
        [imageName removeLastObject];

        dispatch_async(dispatch_get_main_queue(), ^{
            UIImageView *image = imageView[index];
            image.image = [UIImage imageWithData:data];
        });
    }
    /* 解锁 */
    OSSpinLockUnlock(&spinlock);
}
```
此外还有好几种线程锁以后有空再研究: )
文章代码在[git](https://github.com/gxq93/GCD_Lock)上。
