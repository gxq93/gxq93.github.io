---
title: link programming in Objective-C
date: 2016-4-13 13:34:02
tags:
cover: http://img.tuku.cn/file_big/201501/b54c1abfa8174e138eead1252b98db24.jpg
---
在oc中方法的调用大部分都是通过``[ ]``，这与其他很多语言有很大的差异，而有些开发框架如``Masonry``则采用了链式编程的思想，这大大增加了代码的可读性和简洁性。今天有空就对这种思想简单的实践一下。

## 链式编程思想
链式编程思想即将多个操作（多行代码）用过.语法链接在一起，形成单行代码，如``objc.method1().method2()``，跟c,java,swift等其他语言一样。

在oc中链式编程主要的方式是方法都是返回自身，而实现的功能块都在这个方法中，而接下去返回的对象因为是调用者自身，因此可以不断的调用方法，从而形成链式的结构，如果想在方法中添加参数，一般都以block的形式传参，block返回方法调用者本身。

## 通过block构造一个简单的计算器

接下来通过构造一个简单的加减法计算器来对这种思想进行简单的实践。

首先构造一个NSObject的子类计算器``CaculateMaker``，之后这个计算器将用来作为链式调用的对象。

```objc
@interface CaculateMaker : NSObject
@property (nonatomic,assign)int result;
- (CaculateMaker *(^)(int))add;
- (CaculateMaker *(^)(int))substrct;
- (CaculateMaker *(^)(int))muilt;
- (CaculateMaker *(^)(int))divide;
- (void(^)(void))showResult;
```

属性``result``用来记录每一次计算后的结果，声明的五个方法分别是加减乘除四种运算和打印结果。可以看到前四种方法都是返回一个带参数int返回``CaculateMaker``对象的block，这个参数就是用来在链式结构中传入的参数，最后一个方法不需要参数，我只是需要打印最后的结果，因此也不需要再进行链式调用，链式至此结束，之后也不能再调用``CaculateMaker``的其他方法，因为这个block没有返回值。

方法的实践：
```objc
- (CaculateMaker *(^)(int))add {
    return ^(int num){
        _result = _result + num;
        return self;
    };
}
- (CaculateMaker *(^)(int))substrct {
    return ^(int num){
        _result = _result - num;
        return self;
    };
}
- (CaculateMaker *(^)(int))muilt {
    return ^(int num){
        _result = _result * num;
        return self;
    };
}
- (CaculateMaker *(^)(int))divide {
    return ^(int num){
        _result = _result / num;
        return self;
    };
}
- (void (^)(void))showResult {
    return ^(){        
        NSLog(@"result is %d",_result);
    };
}
```
可以看到这些方法的实现都是大同小异，即返回一个block，计算的方法的block中都返回``CaculateMaker``自身，并在这个blog中将这个计算器保存的结果``result``进行相应的运算。
调用这个计算器是这样的
```objc
CaculateMaker *maker = [[CaculateMaker alloc]init];
maker.result = 1;
maker.add(1).add(3).substrct(1).muilt(3).divide(3).showResult();
```
在每一次.语法调用之后都会返回``maker``自身，这就是链式编程的主要思想，当然如果不想传参的话直接用
```objc
- (CaculateMaker*)method {
    //do something//
    return self;
}
```
即可，只要方法最后返回是对象自身就好。

## 更符合逻辑的设计
如果将这个计算器像上述方法调用，看上去逻辑还是不那么舒服。可以将这个计算器放到一个方法中对其进行调度，就像``Masonry``这么做的，我们想得到的只是最后的结果，把这条链式代码放到一个代码块中处理，返回结果，这样代码的意思就更明确了。

创建一个NSObject的类目，创建一个方法将这个计算器以block方式传入并返回最终结果。

```objc
+ (int)makeCaculate:(void (^)(CaculateMaker *))caculateMaker {
    CaculateMaker *maker = [[CaculateMaker alloc]init];
    caculateMaker(maker);
    return maker.result;
}
```
然后对一个数字进行处理
```objc
int result = [NSObject makeCaculate:^(CaculateMaker *make) {
    make.result = 1;
    make.add(1).add(3).substrct(1).muilt(3).divide(3);
}];
NSLog(@"result is %d",result);
```
这样的逻辑看上去就更清晰了。

## 为什么能用点语法
在实践工程中，突然想到为什么这些方法能用.语法调用呢，我并没有声明属性或者变量，点语法一般用来代替存储器方法setter和getter方法，我这里只是普通的方法，然后调用了一下``self.viewDidLoad``居然真的能够调用，也就是不管返回值是否为空，调用不带参数的方法均能用点语法，但是如果调用无返回值的方法会报警告``"property access result unused - getters should not be used for side effects"``。因此思考其实点语法其实只是一个语法糖，用来代替存取getter和setter方法，其实隐式得调用了oc得方括号方法``[something setter/getter]``，而当你声明无参数方法后系统默认将其当作了读取方法getter，因此可以使用点语法，这也使oc最终能够成功实现链式结构。
