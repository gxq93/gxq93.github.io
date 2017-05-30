---
title: Objective-C内部实现浅谈-KVC和KVO
date: 2015-12-28 15:46:58
tags: [Objective-C,Runtime]
categories: 技术
---
本文主要介绍了 KVC 和 KVO 相关的内部实现。

<!--more-->

# KVC
## KVC 在 iOS 中的定义
无论是 Swift 还是 Objective-C，KVC 的定义都是对 NSObject 的扩展来实现的(Objective-C 中有个显式的 NSKeyValueCoding 类别名，而 Swift 没有，也不需要)，所以对于所有继承了 NSObject 在类型，都能使用 KVC (一些纯 Swift 类和结构体是不支持 KVC 的)，下面是 KVC 最为重要的四个方法:
``` objc
- (nullable id)valueForKey:(NSString *)key;                          
- (void)setValue:(nullable id)value forKey:(NSString *)key;          
- (nullable id)valueForKeyPath:(NSString *)keyPath;                 
- (void)setValue:(nullable id)value forKeyPath:(NSString *)keyPath;  
```
## KVC 是怎么寻找 Key 的
当调用``setValue:@"" forKey:@""``的代码时，底层的执行机制如下：
* 程序优先调用``set<Key>:``属性值方法，代码通过 setter 方法完成设置。注意，这里的``<key>``是指成员变量名，首字母大清写要符合 KVC 的全名规则，下同。
* 如果没有找到``set<Key>:``方法，KVC 机制会检查``+(BOOL)accessInstanceVariablesDirectly``方法有没有返回 YES，默认该方法会返回 YES，如果你重写了该方法让其返回 NO 的话，那么在这一步 KVC 会执行``setValue:forUNdefinedKey:``方法，不过一般开发者不会这么做。所以 KVC 机制会搜索该类里面有没有名为 ``_<key> ``的成员变量，无论该变量是在类接口部分定义，还是在类实现部分定义，也无论用了什么样的访问修饰符，只在存在以``<key>``命名的变量，KVC 都可以对该成员变量赋值。
* 如果该类即没有``set<Key>:``方法，也没有``_<key>``成员变量，KVC 机制会搜索``_is<Key>``的成员变量，
* 和上面一样，如果该类即没有``set<Key>:``方法，也没有``_<key>``和``_is<Key>``成员变量，KVC 机制再会继续搜索名为``<key>``和``is<Key>``的成员变量。再给它们赋值。
* 如果上面列出的方法或者成员变量都不存在，系统将会执行该对象的``setValue:forUNdefinedKey:``方法，默认是抛出异常。

如果想让这个类禁用 KVC ，那么重写``+(BOOL)accessInstanceVariablesDirectly``方法让其返回 NO 即可，这样的话如果 KVC 没有找到``set<Key>:``属性名时，会直接用``setValue:forUNdefinedKey:``方法。

当调用``ValueforKey:@""``的代码时，KVC 对 key 的搜索方式不同于``setValue:``属性值``forKey:@"name"``，其搜索方式如下：
* 首先按``get<Key>``，``<key>``，``is<Key>``的顺序方法查找 getter 方法，找到的话会直接调用。如果是 BOOL 或者 int 等值类型， 会做 NSNumber 转换
* 如果上面的 getter 没有找到，KVC 则会查找``countOf<Key>``，``objectIn<Key>AtIndex``，``<Key>AtIndex``三个方法。如果三个方法都找到，那么就会返回一个可以响应 NSArray 的方法的代理集合(它是 NSKeyValueArray，是 NSArray 的子类)，调用这个代理集合的方法，或者说给这个代理集合发送 NSArray 的方法，就会以``countOf<Key>``，``objectIn<Key>AtIndex``，``<Key>AtIndex``这几个方法组合的形式调用。还有一个可选的``get<Key>:range:``方法。所以你想重新定义 KVC 的一些功能，你可以添加这些方法，需要注意的是你的方法名要符合 KVC 的标准命名方法，包括方法签名。
* 如果上面的方法没有找到，那么会查找``countOf<Key>``，``enumeratorOf<Key>``，``memberOf<Key>``三个方法。如果这三个方法都找到，那么就返回一个可以响应 NSSet 所的方法的代理集合，以送给这个代理集合消息方法，就会以``countOf<Key>``，``enumeratorOf<Key>``，``memberOf<Key>``组合的形式调用。
* 如果还没有找到，再检查类方法``+(BOOL)accessInstanceVariablesDirectly``，如果返回 YES (默认行为)，那么和先前的设值一样，会按``_<key>``，``_is<Key>``，``<key>``，``is<Key>``的顺序搜索成员变量名，这里不推荐这么做，因为这样直接访问实例变量破坏了封装性，使代码更脆弱。如果重写了类方法``+(BOOL)accessInstanceVariablesDirectly``返回 NO 的话，那么会直接调用``valueForUndefinedKey:``
* 还没有找到的话，调用``valueForUndefinedKey:``

## 在 KVC 中使用 KeyPath
在开发过程中，一个类的成员变量有可能是其他的自定义类，你可以先用 KVC 获取出来再该属性，然后再次用 KVC 来获取这个自定义类的属性，但这样是比较繁琐的，对此，KVC 提供了一个解决方案，那就是键路径 KeyPath。KVC 对于 KeyPath 是搜索机制第一步就是分离 key，用小数点.来分割 key，然后再像普通 key 一样按照先前介绍的顺序搜索下去。

## KVC 伪实现
``` objc
@interface NSObject(GYKVC)
-(void)setValue:(id)value forKey:(NSString*)key;
-(id)valueforKey:(NSString*)key;
@end

@implementation NSObject(GYKVC)
-(void)setValue:(id)value forKey:(NSString *)key{
    if (key == nil || key.length == 0) {
        return;
    }
    if ([value isKindOfClass:[NSNull class]]) {
        [self setNilValueForKey:key];
        return;
    }
    if (![value isKindOfClass:[NSObject class]]) {
        @throw @"must be s NSobject type";
        return;
    }
    NSString* funcName = [NSString stringWithFormat:@"set%@:",key.capitalizedString];
    if ([self respondsToSelector:NSSelectorFromString(funcName)]) {
        [self performSelector:NSSelectorFromString(funcName) withObject:value];
        return;
    }
    unsigned int count;
    BOOL flag = false;
    Ivar* vars = class_copyIvarList([self class], &count);
    for (NSInteger i = 0; i<count; i++) {
        Ivar var = vars[i];
        NSString* keyName = [[NSString stringWithCString:ivar_getName(var) encoding:NSUTF8StringEncoding] substringFromIndex:1];
        if ([keyName isEqualToString:[NSString stringWithFormat:@"_%@",key]]) {
            flag = true;
            object_setIvar(self, var, value);
            break;
        }
        if ([keyName isEqualToString:key]) {
            flag = true;
            object_setIvar(self, var, value);
            break;
        }
    }
    if (!flag) {
        [self setValue:value forUndefinedKey:key];
    }
}

-(id)myValueforKey:(NSString *)key{
    if (key == nil || key.length == 0) {
        return [NSNull new];
    }
    NSString* funcName = [NSString stringWithFormat:@"gett%@:",key.capitalizedString];
    if ([self respondsToSelector:NSSelectorFromString(funcName)]) {
       return [self performSelector:NSSelectorFromString(funcName)];
    }
    unsigned int count;
    BOOL flag = false;
    Ivar* vars = class_copyIvarList([self class], &count);
    for (NSInteger i = 0; i<count; i++) {
        Ivar var = vars[i];
        NSString* keyName = [[NSString stringWithCString:ivar_getName(var) encoding:NSUTF8StringEncoding] substringFromIndex:1];
        if ([keyName isEqualToString:[NSString stringWithFormat:@"_%@",key]]) {
            flag = true;
            return object_getIvar(self, var);
            break;
        }
        if ([keyName isEqualToString:key]) {
            flag = true;
            return object_getIvar(self, var);
            break;
        }
    }
    if (!flag) {
        [self valueForUndefinedKey:key];
    }
   return [NSNull new];
}
@end
```

# KVO
KVO 其实也是 Objective-C 观察者设计模式的一种实现。KVO 提供一种机制，指定一个被观察对象(例如 A 类)，当对象某个属性(例如 A 中的字符串 name)发生更改时，对象会获得通知，并作出相应处理，且不需要给被观察的对象添加任何额外代码，就能使用 KVO 机制。
## 原理
当观察某对象 A 时，KVO 机制动态创建一个对象 A 当前类的子类，并为这个新的子类重写了被观察属性 keyPath 的 setter 方法。setter 方法随后负责通知观察对象属性的改变状况。

Apple 使用了 isa 混写(isa-swizzling)来实现 KVO。当观察对象 A 时，KVO 机制动态创建一个新的名为``NSKVONotifying_A``的新类，该类继承自对象 A 的本类，且 KVO 为 NSKVONotifying_A 重写观察属性的 setter 方法，setter 方法会负责在调用原 setter 方法之前和之后，通知所有观察对象属性值的更改情况。

在这个过程，被观察对象的 isa 指针从指向原来的 A 类，被 KVO 机制修改为指向系统新创建的子类  NSKVONotifying_A 类，来实现当前类属性值改变的监听，所以当我们从应用层面上看来，完全没有意识到有新的类出现，这是系统“隐瞒”了对 KVO 的底层实现过程，让我们误以为还是原来的类。但是此时如果我们创建一个新的名为 NSKVONotifying_A 的类，就会发现系统运行到注册 KVO 的那段代码时程序就崩溃，因为系统在注册监听的时候动态创建了名为 NSKVONotifying_A 的中间类，并指向这个中间类了。

KVO 的键值观察通知依赖于 NSObject 的两个方法``willChangeValueForKey:``和``didChangevlueForKey:``在存取数值的前后分别调用2个方法：

被观察属性发生改变之前，willChangeValueForKey: 被调用，通知系统该 keyPath 的属性值即将变更；当改变发生后， didChangeValueForKey: 被调用，通知系统该 keyPath 的属性值已经变更。之后``observeValueForKey:ofObject:change:context:``也会被调用。且重写观察属性的 setter 方法这种继承方式的注入是在运行时而不是编译时实现的。大致代码是这样的：
``` objc
-(void)setName:(NSString *)newName {
  [self willChangeValueForKey:@"name"];    
  [super setValue:newName forKey:@"name"];
  [self didChangeValueForKey:@"name"];     
}
```
观察者观察的是属性，只有遵循 KVO 变更属性值的方式才会执行 KVO 的回调方法，例如是否执行了 setter 方法、或者是否使用了 KVC 赋值(当然从上一节可知 KVC 内部走的也是 setter 方法)。如果赋值没有通过 setter 方法或者 KVC，而是直接修改属性对应的成员变量，例如：仅调用``_name = @"newName"``，这时是不会触发 KVO 机制，更加不会调用回调方法的。所以使用 KVO 机制的前提是遵循 KVO 的属性设置方式来变更属性值。
