---
title: Objective-C内部实现浅谈-方法
date: 2015-08-06 10:47:11
tags: [Objective-C,Runtime]
categories: 技术
thumbnail: http://7xtg0o.com1.z0.glb.clouddn.com/1-jwiNw4za2fidw6xi-oyZRQ.jpeg
---
# 方法
Objective-C 最重要的一个特性就是“消息传递”，消息有名称(name)和选择器(selector)，可以接受参数并可能有返回值。本文主要介绍了消息传递的一些原理。

<!--more-->

## C函数
C 使用的是静态绑定，在编译期就能决定运行时所应调用的函数(不考虑内联)函数地址硬解码在指令中，而如果出现只有一个函数调用指令，如果待调用的函数地址无法硬解码在指令之中，就得使用动态绑定了，在运行期把代码读取出来：
``` c
void printHello() {
    printf("hello");
}
void printWorld() {
    printf("world");
}
void doSomething() {
    void (*func)()
    if(type == 0) {
        func = printHello();
    }
    else {
        func = printWorld();
    }
    func();
}
```
## Objective-C方法
Objective-C 中如果向某对象传递消息，就会使用动态绑定机制来决定需要调用的方法
过程：
```objc
id returnValue = [someObject messageName:parameter];
```
someObject 叫做``receiver``，messageName 叫做``selector``，selector 与参数``parameter``合起来成为``message``，编译器看到 message 后，将其转换成一条标准的 c 函数``objc_msgSend``。
objc_msgSend 其原型如下：
```objc
void objc_msgSend(id self, SEL cmd, ...)
```
这是个“参数个数可变的函数”(variadic function)，能接受两个或两个以上的参数。第一个参数代表接收者，第二个参数代表选择器(``SEL``是选择器的类型)，后续参数就是消息中的那些参数，其顺序不变。选择器指的就是方法的名字。
### **self**:
``self``和``cmd``是隐藏参数，在编译期被插入实现代码。
self：指向消息的接受者 target 的对象类型，作为一个占位参数，消息传递成功后 self 将指向消息的 receiver。
当向一般对象发送消息时，调用 objc_msgSend，每个方法的实现的第一个参数即为 self。
### **super**:
``super``并不是隐藏参数，它实际上只是一个”编译器标示符”，它负责告诉编译器，当调用方法时，跳过当前类去调用父类的方法，而不是本类中的方法。实际上给 super 发消息时，super 还是与 self 指向的是相同的消息接收者。
```objc
struct objc_super {
    __unsafe_unretained id receiver;
    __unsafe_unretained Class super_class;
};
```
当向 super 发送消息时，调用的是``objc_msgSendSuper``。如果返回值是一个结构体，则会调用``objc_msgSend_stret``或``objc_msgSendSuper_stret``。
```objc
id objc_msgSendSuper ( struct objc_super *super, SEL op, ... );
```
### **SEL**:
``SEL``是选择器的类型，是表示一个方法的 selector 的指针，映射方法的名字。Objective-C 在编译时，会依据每一个方法的名字、参数序列，生成一个唯一的整型标识(Int 类型的地址)，这个标识就是 SEL。
SEL的作用是作为``IMP``的 KEY，存储在 NSSet 中，便于 hash 快速查询方法。SEL 不能相同，对应方法可以不同。所以在 Objective-C 同一个类(及类的继承体系)中，不能存在2个同名的方法，就算参数类型不同。多个方法可以有同一个 SEL。
不同的类可以有相同的方法名。不同类的实例对象执行相同的 selector 时，会在各自的方法列表中去根据 selector 去寻找自己对应的 IMP。
相关概念：类型编码(Type Encoding)
编译器将每个方法的返回值和参数类型编码为一个字符串，并将其与方法的 selector 关联在一起。可以使用``@encode``编译器指令来获取它。
```objc
typedef struct objc_selector *SEL;
```
**SEL 本质是一个字符串！**
### **IMP**:
``IMP``是指向实现函数的指针，通过 SEL 取得 IMP 后，我们就获得了最终要找的实现函数的入口。
```objc
typedef id (*IMP)(id, SEL, ...)
```
### **Method**:
这个结构体相当于在``SEL``和``IMP``之间作了一个绑定。这样有了 SEL，我们便可以找到对应的 IMP，从而调用方法的实现代码。(在运行时才将 SEL 和 IMP 绑定, 动态配置方法)
```objc
typedef struct objc_method *Method;
struct objc_method {
    SEL method_name             /* 方法名 */
    char *method_types          /* 参数类型 */
    IMP method_imp              /* 方法实现 */
}
```
``objc_method_list`` 就是用来存储当前类的方法链表，``objc_method``存储了类的某个方法的信息。
```objc
struct objc_method_list {
    struct objc_method_list *obsolete                        
    int method_count                                                 
    #ifdef __LP64__
    int space                                                              
    #endif
    /* variable length structure */
    struct objc_method method_list[1]                        
}
```
### 快速映射表
方法调用最先是在方法缓存，也就是快速映射表里找的，方法调用是懒调用，第一次调用时加载后加到快速映射表里。一个 objc 程序启动后，需要进行类的初始化、调用方法时的 cache 初始化，再发送消息的时候就直接走缓存(引申：``+load``方法和``+initialize``方法。load 方法是首次加载类时调用，绝对只调用一次；initialize 方法是首次给类发消息时调用，通常只调用一次，但如果它的子类初始化时未定义 initialize 方法，则会再调用一次它的initialize 方法)
```objc
struct objc_cache {
    /* 缓存bucket的总数 */
    unsigned int mask /* total = mask + 1 */                 
    /* 实际缓存bucket的总数 */
    unsigned int occupied                                    
    /* 指向Method数据结构指针的数组 */
    Method buckets[1]                                        
};
```
### 尾调用优化
如果某函数的最后一项操作是调用另外一个函数，那么就可以运用“尾调用优化”技术。编译器会生成调转至另一函数所需的指令码，而且不会向调用堆栈中推入新的“栈帧”(frame stack)。只有当某函数的最后一个操作仅仅是调用其他函数而不会将其返回值另作他用时，才能执行“尾调用优化”。这项优化对 objc_msgSend 非常关键，如果不这么做的话，那么每次调用 Objective-C 方法之前，都需要为调用 objc_msgSend 函数准备“栈帧”，大家在“栈踪迹”(stack trace)中可以看到这种“栈帧”。此外，若是不优化，还会过早地发生“栈溢出”(stack overflow)现象。
## 操作函数
### **method_**：
**invoke**：方法实现的返回值
```objc
/* 调用指定方法的实现 */
id method_invoke ( id receiver, Method m, ... );
/* 调用返回一个数据结构的方法的实现 */
void method_invoke_stret ( id receiver, Method m, ... );
```
**get**：方法名；方法实现；参数与返回值相关
```objc
/* 获取方法名 */
SEL method_getName ( Method m );
/* 返回方法的实现 */
IMP method_getImplementation ( Method m );
/* 获取描述方法参数和返回值类型的字符串 */
const char * method_getTypeEncoding ( Method m );
/* 返回方法的参数的个数 */
unsigned int method_getNumberOfArguments ( Method m );
/* 通过引用返回方法指定位置参数的类型字符串 */
void method_getArgumentType ( Method m, unsigned int index, char *dst, size_t dst_len );
```
**copy**：返回值类型，参数类型
```objc
/* 获取方法的返回值类型的字符串 */
char * method_copyReturnType ( Method m );
/* 获取方法的指定位置参数的类型字符串 */
char * method_copyArgumentType ( Method m, unsigned int index );
/* 通过引用返回方法的返回值类型字符串 */
void method_getReturnType ( Method m, char *dst, size_t dst_len );
```
**set**：方法实现
```objc
/* 设置方法的实现 */
IMP method_setImplementation ( Method m, IMP imp );
```
**exchange**：交换方法实现
```objc
/* 交换两个方法的实现 */
void method_exchangeImplementations ( Method m1, Method m2 );
```
**description**：方法描述
```objc
/* 返回指定方法的方法描述结构体 */
struct objc_method_description * method_getDescription ( Method m );
```
### **SEL_**:
```objc
/* 返回给定选择器指定的方法的名称 */
const char * sel_getName ( SEL sel );
/* 在Objective-C Runtime系统中注册一个方法，将方法名映射到一个选择器，并返回这个选择器 */
SEL sel_registerName ( const char *str );
/* 在Objective-C Runtime系统中注册一个方法 */
SEL sel_getUid ( const char *str );
/* 比较两个选择器 */
BOOL sel_isEqual ( SEL lhs, SEL rhs );
```
## 消息传递过程
* 检查 target 是否为 nil。如果为 nil，直接 cleanup，然后 return。(这就是我们可以向 nil 发送消息的原因)
  如果方法返回值是一个对象，那么发送给 nil 的消息将返回 nil。如果方法返回值为指针类型，其指针大小为小于或者等于**sizeof(void*)、float、double、long double 、long long**的整型标量，发送给nil的消息将返回0。如果方法返回值为结构体，发送给 nil 的消息将返回0。结构体中各个字段的值将都是0。如果方法的返回值不是上述提到的几种情况，那么发送给 nil 的消息的返回值将是未定义的。
* 如果 target 非 nil，在 target 的``Class``中根据``SEL``去找``IMP``。（因为同一个方法可能在不同的类中有不同的实现，所以我们需要依赖于接收者的类来找到的确切的实现）
* 首先它找到``selector``对应的方法实现。
* 在 target 类的方法缓存列表里检查有没有对应的方法实现，有的话，直接调用。
* 比较请求的 selector 和类方法列表中的 selector，对应的话，直接调用。
* 比较请求的 selector 和父类方法列表，父类的父类，直至根类，如果有对应，则直接调用。（方法重写拦截父类方法的原理）
* 调用方法实现，并将接收者对象及方法的所有参数传给它。
* 最后，将实现函数的返回值作为自己的返回值。

## 动态方法解析与消息转发
如果以上的类中没有找到对应的 selector (一般保险起见先用``respondsToSelector:``内省判断)，还可以利用消息转发机制依次执行以下流程：
* Method Resolution(动态方法解析)：
  用所属类的类方法``+(BOOL)resolveInstanceMethod:``(实例方法)或者``+(BOOL)resolveClassMethod:``(类方法)，在此方法里添加``class_addMethod``函数。一般用于@dynamic动态属性。（当一个属性声明为@dynamic，就是向编译器保证编译时不用管 setter/getter 实现，一定会在运行时实现）
* Fast Forwarding (快速消息转发)：
  如果上一步无法响应消息，调用``-(id)forwardingTargetForSelector:(SEL)aSelector``方法，将消息接受者转发到另一个对象target。(不能为self，否则死循环)
* Normal Forwarding(普通消息转发)：
  如果上一步无法响应消息，调用方法签名``-(NSMethodSignature )methodSignatureForSelector:(SEL)aSelector``，方法签名目的将函数的参数类型和返回值封装。如果返回非 nil，则创建一个NSInvocation 对象利用方法签名和 selector 封装未被处理的消息，作为参数传递给``-(void)forwardInvocation:(NSInvocation )anInvocation``。

如果以上步骤(消息传递和消息转发)还是不能响应消息，则调动``doesNotRecognizeSelector：``方法，抛出异常。
```objc
unrecognized selector sent to instance
```
