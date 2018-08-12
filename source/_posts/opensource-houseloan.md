---
title: RAC使用心得
date: 2016-09-08 13:46:24
tags: [Objective-C,响应式编程]
categories: 技术
---
# Intro
最近公司要求开发一个小应用“房贷计算器”来引流，由于要求需要一礼拜内完成开发上线，因此时间有点仓促所以没有仔细构思就开始开发了，所以其实很多地方设计的极其不合理，后期也懒得再修改维护了，就开源一下，有问题的地方看客看了笑笑就好。附上[github地址](https://github.com/gxq93/HouseloanCalculator)
响应式编程似乎很符合计算器的特点，因此项目导入了 ReactiveCocoa2.5 框架，接下来纪录一些自己对 RAC 的使用心得。

<!--more-->

# Tip1 RACCommand传达信号
当要做如下图的小组件时，我个人的做法是放三个 button 上去，最后暴露出一个`signal`，每当点击到其中一个 button 时，把 index 发送出来。其实如果不用 RAC 的套路做，只要写个代理方法或者 block 就能很好的完成需求，但是既然要用 RAC，就要按照响应式的思路来。

{% asset_img segment.png %}

但是如果普通的使用``[button rac_signalForControlEvents:]``的方法，不好把 index 传出来，rac_signalForControlEvents 的实现如下：
``` objc
return [[RACSignal createSignal:^(id<RACSubscriber> subscriber) {
    @strongify(self);
    [self addTarget:subscriber action:@selector(sendNext:) forControlEvents:controlEvents];
    [self.rac_deallocDisposable addDisposable:[RACDisposable disposableWithBlock:^{
        [subscriber sendCompleted];
    }]];
    return [RACDisposable disposableWithBlock:^{
        @strongify(self);
        [self removeTarget:subscriber action:@selector(sendNext:) forControlEvents:controlEvents];
    }];
}]
```
可见默认``sendNext``的值是``sender``。如果使用``RACCommand``，要指定 next 就容易了很多。我的做法是给每个 button 创建一个 command，在返回的信号中将 index 传出来
``` objc
button.rac_command = [[RACCommand alloc]initWithSignalBlock:^RACSignal *(id input) {
    @weakify(self)
    return [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        @strongify(self)
        [subscriber sendNext:@(i)];
        [subscriber sendCompleted];
        return nil;
    }];
}];
```
最后将三个 command 返回的信号``merge``在一起
``` objc
self.selectSignal = [self.selectSignal merge:button.rac_command.executionSignals.switchToLatest];
/* selectSignal初始化 */
- (RACSignal *)selectSignal {
    if (!_selectSignal) {
        _selectSignal = [RACSignal empty];
    }
    return _selectSignal;
}
```
# Tip2 rac_sequence的线程问题
对数组的 rac_sequence 进行操作，比如遍历时要注意在订阅后的操作对线程是否有要求，比如如果要对 button 字体颜色进行修改，如果直接使用
``` objc
[[self.buttonArray.rac_sequence signal]subscribeNext:^(UIButton *button) {
    [button setTitleColor:UIColorFromRGB(0x666666) forState:UIControlStateNormal];
}];
```
是达不到你需要的效果的，因为
``` objc
- (RACSignal *)signal {
    return [[self signalWithScheduler:[RACScheduler scheduler]] setNameWithFormat:@"[%@] -signal", self.name];
}
```
方法是在异步线程中完成的，也就是不在当前线程完成的。
``` objc
+ (instancetype)scheduler {
    return [self schedulerWithPriority:RACSchedulerPriorityDefault];
    /* RACSchedulerPriorityDefault = DISPATCH_QUEUE_PRIORITY_DEFAULT */
}
```
因此指定信号的线程``RACSignal *signal = [self.buttonArray.rac_sequence signalWithScheduler:[RACScheduler immediateScheduler]];``在当前线程，才能达到你想要的效果。当然``[RACScheduler mainThreadScheduler]``是指定在主线程中执行，对于我的需求也是不满足的因为我是要让三个 button 先变成一个颜色再其中一个变特别的颜色，顺序执行。
# Tip3 RACCommand代替acticon
RACCommand 在 mvvm 中是非常有用的，而在我这种小项目中，有时候也能起到小小的封装的效果，比如在下图中两个按钮和上面的蒙层点击都是触发 remove 方法。

{% asset_img pickView.png %}

其实直接用同一个``acticon``就可以解决，而在 RAC 中可以这样
``` objc
- (RACCommand *)removeCommand {
    if (!_removeCommand) {
    _removeCommand = [[RACCommand alloc]initWithSignalBlock:^RACSignal *(id input) {
        @weakify(self)
        return [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
            @strongify(self);
            [self removeFromSuperview];
            [subscriber sendCompleted];
            return nil;
        }];
    }];
}
    return _removeCommand;
}
```
定义一个 command，而可以使三种动作都执行同一个 command
```objc
_cancelButton.rac_command = self.removeCommand;
_confirmButton.rac_command = self.removeCommand;
[tap.rac_gestureSignal subscribeNext:^(id x) {
    [self.removeCommand execute:nil];
}];
```
# Tip4 协议信号注意点
在创建协议信号时应注意先创建信号如：
``` objc
[self rac_signalForSelector:@selector(pickerView:didSelectRow:inComponent:) fromProtocol:@protocol(UIPickerViewDelegate)]
```
再执行``self.pickView.delegate = self;``，因为这个方式实际调用了
``` objc
static RACSignal *NSObjectRACSignalForSelector(NSObject *self, SEL selector, Protocol *protocol)
```
这个方法就是 hook 了你要调用的方法，然后进行 method swizzing，因此你要使用 hook 后的方法，而不是先实现系统默认的方法，所以先实行 RAC 的方法，再调用协议方法。
# Tip5 重复订阅问题
其实整个项目最失败最蠢的设计就是我把每个 cell 上的信号 combine 在了一起从而进行计算。。。其实这是非常不合理的，因为不显示在屏幕上的 cell 都是懒加载的所以我就必须很蠢的把所有 cell 都加载到才能进行计算，这种设计是极不规范的，但是因为刚开始已经开始做了，并且没有时间修改了，所以只能将错就错。**其实应该让计算从一个数据源去取，而让 cell 和数据源绑定在一起**。
而因为我这不合理的设计，导致了我的信号订阅得在 tableView 创建的方法中执行，而这个方法众所周知是不停调用的，这就导致了重复订阅的问题，而我想出来的方法是定义一个全局的 disposable 对象``@property(nonatomic,strong)RACDisposable *caculateDisposable;
``，在每次调用方法开始时先取消之前的订阅再次进行订阅。
``` objc
if (self.caculateDisposable) {
    [self.caculateDisposable dispose];
}
@weakify(self)
self.caculateDisposable = [[RACSignal combineLatest:self.signalArray] subscribeNext:^(RACTuple *tuple) {
    @strongify(self)
    HCHeaderCell *headerCell = (HCHeaderCell*)tableView.visibleCells[0];
    [headerCell configCellContentWithRACTuple:tuple withType:self.loadnType];
}];
```
看着就很别扭，但是暂时就这样吧。