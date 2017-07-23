---
title: ReactNative与原生手势的冲突
date: 2017-07-02 15:54:53
tags: [Objective-C, React Native]
categories: 技术
---

一直以为 ReactNative 在 iOS 端的手势是直接导出原生的手势供 RN 端来使用，直到踩到了坑看了源码才发现原来 RN 的手势是这样实现的。

<!--more-->

# 场景

介绍一下产生问题的场景，看如下 gif。

![](http://7xtg0o.com1.z0.glb.clouddn.com/gesture-conflict.gif)

首先底部的容器是一个iOS原生的 UIScrollView，由 UIScrollView 驱动上层页面的滑动，而每一个页面是一个单独的 RCTRootView 来展示 RN 的内容，在 RN 页面中有一个类似 Banner 的组件，由 RN 的``PanResponder``驱动，代码大致是这样的：

``` jsx
constructor() {
        super();
        this.state = {
            left: 0,
        }
    }

    componentWillMount() {
        this._panResponder = PanResponder.create({
            onStartShouldSetPanResponder: () => true,
            onMoveShouldSetPanResponder: ()=> true,
            onPanResponderGrant: ()=> {
                this._left = this.state.left
            },
            onPanResponderReject: ()=> {
            },
            onPanResponderMove: (evt, gs)=> {
                this.setState({
                    left: this._left + gs.dx
                })
            },
            onPanResponderRelease: (evt, gs)=> {
                this.setState({
                    left: this._left + gs.dx
                })
            },
        })
    }

    render() {
        return (
            <View style={styles.container}>
                <View
                    {...this._panResponder.panHandlers}
                    style={[styles.rect, {"left": this.state.left, overflow: 'hidden'}]}>
                    <Image style={{width: screenWidth, height: 200}}
                           source={{uri: 'https://images.unsplash.com/photo-1441742917377-57f78ee0e582?h=1024'}}/>
                    <Image style={{width: screenWidth, height: 200}}
                           source={{url: 'https://images.unsplash.com/photo-1441716844725-09cedc13a4e7?h=1024'}}/>
                </View>
                <View style={styles.placeholderView}>
                    <Text style = {styles.placeholderText}>这里是第一个页面</Text>
                </View>
            </View>
        );
    }
}
```

如果把 RN 的 PanResponder 当作原生的 PanGestureRecognizer 来看，那么应该是当手势在图片上滑动时，应该触发图片上的手势识别器，从而忽略 ScrollView 上的手势，但是真实的情形确是图片上的手势先响应，但过一会儿后 UIScrollView 上的手势也开始响应，从而导致在这个 Banner 组件滑动过程中 UIScrollView 也跟着滑动。

# RN手势识别器原理

既然有这么明显的手势冲突，第一反应肯定就是去看看 RN 上的手势识别究竟是怎么实现的。在阅读了 RN 所用的手势识别原生模块``RCTTouchHandler``的代码后，终于发现 RN 所使用的手势识别器与之前所设想的大相径庭。

{% asset_img RCTTouchHandler1.png %}

RCTTouchHandler 添加在 ``RCTRootContentView``之上，而这个 RCTRootContentView 对应于 RCTRootView，也就是说对于每一个 RN 页面，其实有且只有一个手势识别器 RCTTouchHandler，而这个手势识别器的范围是整个 RN 页面。

那么这个自定义的手势识别器是怎么让原生与 RN 通信的呢？

```objectivec
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
  [super touchesBegan:touches withEvent:event];

  _coalescingKey++;
  // "start" has to record new touches before extracting the event.
  // "end"/"cancel" needs to remove the touch *after* extracting the event.
  [self _recordNewTouches:touches];
  if (_dispatchedInitialTouches) {
    [self _updateAndDispatchTouches:touches eventName:@"touchStart" originatingTime:event.timestamp];
    self.state = UIGestureRecognizerStateChanged;
  } else {
    self.state = UIGestureRecognizerStateBegan;
  }
}

- (void)touchesMoved:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
  [super touchesMoved:touches withEvent:event];

  if (_dispatchedInitialTouches) {
    [self _updateAndDispatchTouches:touches eventName:@"touchMove" originatingTime:event.timestamp];
    self.state = UIGestureRecognizerStateChanged;
  }
}

- (void)touchesEnded:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
  [super touchesEnded:touches withEvent:event];

  _coalescingKey++;
  if (_dispatchedInitialTouches) {
    [self _updateAndDispatchTouches:touches eventName:@"touchEnd" originatingTime:event.timestamp];

    if (RCTAllTouchesAreCancelledOrEnded(event.allTouches)) {
      self.state = UIGestureRecognizerStateEnded;
    } else if (RCTAnyTouchesChanged(event.allTouches)) {
      self.state = UIGestureRecognizerStateChanged;
    }
  }
  [self _recordRemovedTouches:touches];
}

- (void)touchesCancelled:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
  [super touchesCancelled:touches withEvent:event];

  _coalescingKey++;
  if (_dispatchedInitialTouches) {
    [self _updateAndDispatchTouches:touches eventName:@"touchCancel" originatingTime:event.timestamp];

    if (RCTAllTouchesAreCancelledOrEnded(event.allTouches)) {
      self.state = UIGestureRecognizerStateCancelled;
    } else if (RCTAnyTouchesChanged(event.allTouches)) {
      self.state = UIGestureRecognizerStateChanged;
    }
  }
  [self _recordRemovedTouches:touches];
}
```

上面的代码是 RCTTouchHandler 对手势变化状态时的相应操作，可以看到其实 RCTTouchHandler 并没有对哪种状态进行特殊的判断或者处理，而是记录每次状态，最后把他包装成``RCTTouchEvent``发送给 RN 端对应的组件：

``` objectivec
- (void)_updateAndDispatchTouches:(NSSet<UITouch *> *)touches
                        eventName:(NSString *)eventName
                  originatingTime:(__unused CFTimeInterval)originatingTime
{
  // Update touches
  NSMutableArray<NSNumber *> *changedIndexes = [NSMutableArray new];
  for (UITouch *touch in touches) {
    NSInteger index = [_nativeTouches indexOfObject:touch];
    if (index == NSNotFound) {
      continue;
    }

    [self _updateReactTouchAtIndex:index];
    [changedIndexes addObject:@(index)];
  }

  if (changedIndexes.count == 0) {
    return;
  }

  // Deep copy the touches because they will be accessed from another thread
  // TODO: would it be safer to do this in the bridge or executor, rather than trusting caller?
  NSMutableArray<NSDictionary *> *reactTouches =
    [[NSMutableArray alloc] initWithCapacity:_reactTouches.count];
  for (NSDictionary *touch in _reactTouches) {
    [reactTouches addObject:[touch copy]];
  }

  RCTTouchEvent *event = [[RCTTouchEvent alloc] initWithEventName:eventName
                                                         reactTag:self.view.reactTag
                                                     reactTouches:reactTouches
                                                   changedIndexes:changedIndexes
                                                    coalescingKey:_coalescingKey];
  [_eventDispatcher sendEvent:event];
}
```

中间的过程由于要介绍的话需要太多篇幅就暂时不讲了，流程图大致是这样的：

{% asset_img RCTTouchHandler2.png %}

从手势触摸开始直到 RN 端在 JS 线程上执行代码整个过程的调用栈如图：

{% asset_img RCTTouchHandler3.png %}

# 冲突原因

了解了 RN 手势识别器实现的大致原理，再回到我们起冲突的地方，我们可以大致思考一下，既然 RN 用的是自定义的原生手势识别器，那么为什么会出现手势冲突的原因呢。最关键的代码在这里：

``` objectivec
- (BOOL)canPreventGestureRecognizer:(__unused UIGestureRecognizer *)preventedGestureRecognizer
{
  return NO;
}

- (BOOL)canBePreventedByGestureRecognizer:(__unused UIGestureRecognizer *)preventingGestureRecognizer
{
  return NO;
}
```

这里 RCTTouchHandler 实现了两个原生手势的代理，即他设置了当其他手势被识别时他既不去阻止其他手势，也不关心其他手势，就因为他什么都不做，这就导致了要是其他手势被识别后，他并不能去阻止其他手势被识别，这就造成了我们之前出现的情景。

我在 Hook 了 ``UIGestureRecognizerState``这一属性后打印了所有手势的状态，有关 UIScrollView 及 RN 手势识别的状态变化大致是这样的：

{% asset_img RCTTouchHandler4.png %}

这就能大致找到了手势冲突的根源。

# 解决方案

既然有了问题就必须有解决方案，但是其实当目前为止我也还没有想出来最佳的解决方案，要是有我都能跟 RN 提个 PR 了🌚

以下是几种避免冲突的方法：

1.在 RN 端操作：在 PanResponder onGrant 和 onRelease 时设置 UIScrollView 的滚动状态

2.在原生端操作：可以创建一个自定义的 UIScrollView 子类，实现

``` objectivec
-(BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldRequireFailureOfGestureRecognizer:(UIGestureRecognizer *)otherGestureRecognizer
```

代理方法，当接收到 RCTTouchHandler 手势时，让 UIScrollView 的手势失败。

3.最简单的方法—使用 ScrollView 代替 PanResponder（只针对指定场景，比如这个 Banner 组件我就是这么做的。。。）
