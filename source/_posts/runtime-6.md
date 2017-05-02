---
title: Objective-C内部实现剖析-KVC和KVO
date: 2015-12-28 15:46:58
tags: [Objective-C,Runtime]
categories: 技术
thumbnail: http://7xtg0o.com1.z0.glb.clouddn.com/1-yAQxTFJwEKXnAr2JrOFs7w.jpeg
---
本文主要介绍了KVC和KVO相关的内部实现。

<!--more-->

# KVC
## KVC在iOS中的定义
无论是Swift还是Objective-C，KVC的定义都是对NSObject的扩展来实现的(Objective-C中有个显式的NSKeyValueCoding类别名，而Swift没有，也不需要)所以对于所有继承了NSObject在类型，都能使用KVC(一些纯Swift类和结构体是不支持KVC的)，下面是KVC最为重要的四个方法:
``` objc
- (nullable id)valueForKey:(NSString *)key;                          /* 直接通过Key来取值 */
- (void)setValue:(nullable id)value forKey:(NSString *)key;          /* 通过Key来设值 */
- (nullable id)valueForKeyPath:(NSString *)keyPath;                  /* 通过KeyPath来取值 */
- (void)setValue:(nullable id)value forKeyPath:(NSString *)keyPath;  /* 通过KeyPath来设值 */
```
NSKeyValueCoding类别中还有其他的一些方法，下面列举一些:
``` objc
+ (BOOL)accessInstanceVariablesDirectly;
/* 默认返回YES，表示如果没有找到Set<Key>方法的话，会按照_key，_iskey，key，iskey的顺序搜索成员，设置成NO就不这样搜索 */
- (BOOL)validateValue:(inout id __nullable * __nonnull)ioValue forKey:(NSString *)inKey error:(out NSError **)outError;
/* KVC提供属性值确认的API，它可以用来检查set的值是否正确、为不正确的值做一个替换值或者拒绝设置新值并返回错误原因。*/
- (NSMutableArray *)mutableArrayValueForKey:(NSString *)key;
/* 这是集合操作的API，里面还有一系列这样的API，如果属性是一个NSMutableArray，那么可以用这个方法来返回 */
- (nullable id)valueForUndefinedKey:(NSString *)key;
/* 如果Key不存在，且没有KVC无法搜索到任何和Key有关的字段或者属性，则会调用这个方法，默认是抛出异常 */
- (void)setValue:(nullable id)value forUndefinedKey:(NSString *)key;
/* 和上一个方法一样，只不过是设值。*/
- (void)setNilValueForKey:(NSString *)key;
/* 如果你在SetValue方法时面给Value传nil，则会调用这个方法 */
- (NSDictionary<NSString *, id> *)dictionaryWithValuesForKeys:(NSArray<NSString *> *)keys;
/* 输入一组key,返回该组key对应的Value，再转成字典返回，用于将Model转到字典。*/
```
## KVC是怎么寻找Key的
当调用setValue:属性值forKey:@"name"的代码时，底层的执行机制如下：
* 程序优先调用set&lt;Key&gt;:属性值方法，代码通过setter方法完成设置。注意，这里的&lt;key&gt;是指成员变量名，首字母大清写要符合KVC的全名规则，下同
* 如果没有找到setName:方法，KVC机制会检查``+(BOOL)accessInstanceVariablesDirectly``方法有没有返回YES，默认该方法会返回YES，如果你重写了该方法让其返回NO的话，那么在这一步KVC会执行``setValue:forUNdefinedKey:``方法，不过一般开发者不会这么做。所以KVC机制会搜索该类里面有没有名为&lt;key&gt;的成员变量，无论该变量是在类接口部分定义，还是在类实现部分定义，也无论用了什么样的访问修饰符，只在存在以&lt;key&gt;命名的变量，KVC都可以对该成员变量赋值。
* 如果该类即没有set&lt;Key&gt;:方法，也没有\_&lt;key&gt;成员变量，KVC机制会搜索\_is&lt;Key&gt;的成员变量，
* 和上面一样，如果该类即没有set&lt;Key&gt;:方法，也没有\_&lt;key&gt;和\_is&lt;Key&gt;成员变量，KVC机制再会继续搜索&lt;key&gt;和is&lt;Key&gt;的成员变量。再给它们赋值。
* 如果上面列出的方法或者成员变量都不存在，系统将会执行该对象的``setValue:forUNdefinedKey:``方法，默认是抛出异常。

如果开发者想让这个类禁用KVC里，那么重写``+(BOOL)accessInstanceVariablesDirectly``方法让其返回NO即可，这样的话如果KVC没有找到set&lt;Key&gt;:属性名时，会直接用``setValue:forUNdefinedKey:``方法。

当调用ValueforKey：@"name"的代码时，KVC对key的搜索方式不同于setValue:属性值forKey:@"name"，其搜索方式如下：
* 首先按get&lt;Key&gt;，&lt;key&gt;，is&lt;Key&gt;的顺序方法查找getter方法，找到的话会直接调用。如果是BOOL或者int等值类型， 会做NSNumber转换
* 如果上面的getter没有找到，KVC则会查找countOf&lt;Key&gt;，objectIn&lt;Key&gt;AtIndex，&lt;Key&gt;AtIndex格式的方法。如果countOf&lt;Key&gt;和另外两个方法中的要个被找到，那么就会返回一个可以响应NSArray所的方法的代理集合(它是NSKeyValueArray，是NSArray的子类)，调用这个代理集合的方法，或者说给这个代理集合发送NSArray的方法，就会以countOf&lt;Key&gt;，objectIn&lt;Key&gt;AtIndex，&lt;Key&gt;AtIndex这几个方法组合的形式调用。还有一个可选的``get<Ket>:range:``方法。所以你想重新定义KVC的一些功能，你可以添加这些方法，需要注意的是你的方法名要符合KVC的标准命名方法，包括方法签名。
* 如果上面的方法没有找到，那么会查找countOf&lt;Key&gt;，enumeratorOf&lt;Key&gt;,memberOf&lt;Key&gt;格式的方法。如果这三个方法都找到，那么就返回一个可以响应NSSet所的方法的代理集合，以送给这个代理集合消息方法，就会以countOf&lt;Key&gt;，enumeratorOf&lt;Key&gt;,memberOf&lt;Key&gt;组合的形式调用。
* 如果还没有找到，再检查类方法``+(BOOL)accessInstanceVariablesDirectly``，如果返回YES(默认行为)，那么和先前的设值一样，会按\_&lt;key&gt;，\_is&lt;Key&gt;，&lt;key&gt;，is&lt;Key&gt;的顺序搜索成员变量名，这里不推荐这么做，因为这样直接访问实例变量破坏了封装性，使代码更脆弱。如果重写了类方法``+(BOOL)accessInstanceVariablesDirectly``返回NO的话，那么会直接调用``valueForUndefinedKey:``
* 还没有找到的话，调用``valueForUndefinedKey:``

## 在KVC中使用KeyPath
在开发过程中，一个类的成员变量有可能是其他的自定义类，你可以先用KVC获取出来再该属性，然后再次用KVC来获取这个自定义类的属性，但这样是比较繁琐的，对此，KVC提供了一个解决方案，那就是键路径KeyPath。KVC对于KeyPath是搜索机制第一步就是分离key，用小数点.来分割key，然后再像普通key一样按照先前介绍的顺序搜索下去。

## KVC伪实现
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
KVO其实也是Objective-C观察者设计模式的一种实现。KVO提供一种机制，指定一个被观察对象(例如A类)，当对象某个属性(例如A中的字符串name)发生更改时，对象会获得通知，并作出相应处理，且不需要给被观察的对象添加任何额外代码，就能使用KVO机制。
## 原理
当观察某对象A时，KVO机制动态创建一个对象A当前类的子类，并为这个新的子类重写了被观察属性keyPath的setter方法。setter方法随后负责通知观察对象属性的改变状况。

Apple使用了isa混写（isa-swizzling）来实现KVO。当观察对象A时，KVO机制动态创建一个新的名为``NSKVONotifying_A``的新类，该类继承自对象A的本类，且KVO为NSKVONotifying_A重写观察属性的setter方法，setter方法会负责在调用原setter方法之前和之后，通知所有观察对象属性值的更改情况。

在这个过程，被观察对象的 isa 指针从指向原来的A类，被KVO机制修改为指向系统新创建的子类 NSKVONotifying_A类，来实现当前类属性值改变的监听，所以当我们从应用层面上看来，完全没有意识到有新的类出现，这是系统“隐瞒”了对KVO的底层实现过程，让我们误以为还是原来的类。但是此时如果我们创建一个新的名为“NSKVONotifying_A”的类()，就会发现系统运行到注册KVO的那段代码时程序就崩溃，因为系统在注册监听的时候动态创建了名为NSKVONotifying_A的中间类，并指向这个中间类了。

KVO的键值观察通知依赖于NSObject的两个方法``willChangeValueForKey:``和``didChangevlueForKey:``在存取数值的前后分别调用2个方法：

被观察属性发生改变之前，``willChangeValueForKey:``被调用，通知系统该keyPath的属性值即将变更；当改变发生后， ``didChangeValueForKey:``被调用，通知系统该keyPath的属性值已经变更。之后``observeValueForKey:ofObject:change:context:``也会被调用。且重写观察属性的setter方法这种继承方式的注入是在运行时而不是编译时实现的。大致代码是这样的：
``` objc
-(void)setName:(NSString *)newName {
  [self willChangeValueForKey:@"name"];    /* KVO在调用存取方法之前总调用 */
  [super setValue:newName forKey:@"name"]; /* 调用父类的存取方法 */
  [self didChangeValueForKey:@"name"];     /* KVO在调用存取方法之后总调用 */
}
```
观察者观察的是属性，只有遵循KVO变更属性值的方式才会执行KVO的回调方法，例如是否执行了setter方法、或者是否使用了KVC赋值(当然从上一节可知KVC内部走的也是setter方法)。如果赋值没有通过setter方法或者KVC，而是直接修改属性对应的成员变量，例如：仅调用_name = @"newName"，这时是不会触发KVO机制，更加不会调用回调方法的。所以使用KVO机制的前提是遵循KVO的属性设置方式来变更属性值。
