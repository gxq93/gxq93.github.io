---
title: 写给自己的Runtime-block
date: 2015-12-22 11:05:55
tags:
cover: http://m2.quanjing.com/2m/top003/top-612099.jpg
---
# Block
在OC中Block的语法看上去特别别扭，但他实际上是作为最普通的C语言源代码来处理的，在实际编译时无法转换成我们能够理解的源代码，因此通过clang(LLVM编译器)先讲代码进行转化成C++代码。如创建一个``block.m``文件，加入内容
``` c
#include <stdio.h>
int main()
{
    void (^block)(void) = ^{printf("Block\n");};
    block();
    return 0;
}
```
使用clang命令行进行转换
``` shell
clang -rewrite-objc block.m
```
转换后将形成一个约520行左右的cpp文件，在这里我们将能看到我们想要的信息。
转换后有关block的内容大致如下：
``` objc
struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    printf("Block\n");
}

static struct __main_block_desc_0 {
    size_t reserved;
    size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};

int main()
{
    void (*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));

    ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);

    return 0;
}
```
## 数据结构
block的定义大致是这样的
``` objc
struct __block_impl {
    void *isa;
    int Flags;
    int Reserved;
    void *FuncPtr;
};
```
可见block结构体中有isa指针，因此在OC中block也是当做对象来使用的。而这个isa指针指向的有三中类型：
* _NSConcreteStackBlock
* _NSConcreteGlobalBlock
* _NSConcreteMallocBlock

而从其名字变可知其三种类对象内存分配的区域是不同的。下表说明了三种类的存储域：

| 类 | 设置对象的存储域 |
| :-------- | :-------- |
| _NSConcreteStackBlock | 栈 |
| _NSConcreteGlobalBlock | 数据存储区（.data区） |
| _NSConcreteMallocBlock | 堆 |

如果block在记述全局变量的地方被设置或者block没有捕获外部变量，那就生成一个_NSConcreteGlobalBlock实例。其它情况都会生成一个_NSConcreteStackBlock实例，也就是说，它是在栈上的，所以一旦它所属的变量超出了变量作用域，该block就被废弃了。而当发生以下任一情况时：
* 手动调用 Block 的实例方法copy
* Block 作为函数返回值返回
* 将 Block 赋值给附有__strong修饰符的成员变量
* 在方法名中含有usingBlock的 Cocoa 框架方法或 GCD 的 API 中传递 Block

如果此时block在栈上，那就复制一份到堆上，并将复制得到的block实例的isa指针设为_NSConcreteMallocBlock：
``` objc
imply.isa = &__NSConcreteMallocBlock;
```
## 实现原理
通过刚刚那个小小的例子来逐步分析一下block的实现原理。
首先例子中block语句``^{printf("Block\n");};``被转换成了
``` objc
static void __main_block_func_0(struct __main_block_impl_0 *__cself)
{
    printf("Block\n");
}
```
通过block的匿名函数实际被作为简单的C语言函数来处理，而block所属于的函数名(main)和在该函数出现的顺序值(0)来给clang转换的函数命名，该函数的参数\_\_cself相当于OC中的self，指向block值的变量。
### \_\_cself
\_\_cself是\_\_main_block_impl_0结构体的指针，声明如下：
``` objc
struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
}
```
第一个成员变量impl即block对象的函数指针，第二个Desc指针：
``` objc
static struct __main_block_desc_0 {
    size_t reserved;
    size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
```
记录了今后版本升级所需的区域和block的大小。
而\_\_main_block_impl_0结构体实际还包含了其构造函数``__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0)``
``` objc
__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
}
```
### 构造函数
接下来我们看一下\_\_main_block_impl_0结构体的构造函数，``impl.isa = &_NSConcreteStackBlock;``声明了block的类型为_NSConcreteStackBlock，当构造函数调用时，即执行``void (^block)(void) = ^{printf("Block\n");};``这步时：
``` objc
void (*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));
```
构造函数就会进行这样的初始化
``` objc
impl.isa = &_NSConcreteStackBlock;
impl.Flags = 0;
impl.FuncPtr = __main_block_func_0;
Desc = __main_block_desc_0_DATA;
```
这样整个block就声明完成了
### 调用
调用block，即``block();``这一步转换后是这样的：
``` objc
((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
```
这就是简单的使用函数指针调用函数，由block语法转换的\_\_main_block\_func\_0函数的指针被赋值在成员变量FuncPtr中，另外也说明了\_\_main_block\_func\_0函数的参数\_\_cself指向block值，在源码中可以看出block正是作为参数进行了传递。
## 捕获外部变量
Block 捕获外部变量其实可分为三种情况：
* 捕获变量的瞬时值
* 捕获__block变量
* 捕获对象

说到外部变量，我们要先说一下C语言中变量有哪几种。一般可以分为一下5种：
* 自动变量
* 函数参数(block中先不考虑这种情况)
* 静态变量
* 静态全局变量
* 全局变量

### 捕获变量的瞬间值
将全局变量、静态全局变量、静态变量在block中捕获__main_block_func_0函数大致是这样的：
``` objc
static void __main_block_func_0(struct __main_block_impl_0 *__cself)
{
    global_i ++;
    static_global_j ++;
    (*static_k) ++;
}
```
首先全局变量和静态全局变量因为是全局的，作用域很广，所以block捕获了它们进去之后，在block里面进行操作，block结束之后，它们的值依旧可以得以保存下来。
如果block外面还有很多自动变量，静态变量，等等，这些变量在block里面并不会被使用到。那么这些变量并不会被block捕获进来，也就是说并不会在构造函数里面传入它们的值，block捕获外部变量仅仅只捕获block闭包里面会用到的值，其他用不到的值，它并不会去捕获。
静态变量传递给block是内存地址值，所以能在block里面直接改变值。静态变量的这种做法似乎也适用于自动变量的访问，但是为什么没有这么做呢？
实际上，在由于blcok语法生成的值block上，可以存有超过其变量作用域的被截获对象的自动变量，变量作用域结束的同时，原来的自动变量被废弃，因此block中超过变量作用域而存在的变量同静态变量一样，将不能通过指针访问原来的自动变量。而静态局部变量具有局部作用域，它只被初始化一次，自从第一次被初始化直到程序运行结束都一直存在，它和全局变量的区别在于全局变量对所有的函数都是可见的，而静态局部变量只对定义自己的函数体始终可见，所以简单来说就是**静态局部变量不会被销毁，所以可以通过指针来访问，自动变量会被销毁，所以不能通过指针访问**
### 捕获\_\_block变量
\_\_block的使用就不多介绍了，他实际就是修饰用于指定将变量值设置到哪个存储域中。
使用\_\_block修饰自动变量转化后的代码主要的变换是这样的：
``` objc
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    __Block_byref_i_0 *i = __cself->i; // bound by ref
    (i->__forwarding->i)++;
    printf("%d",(i->__forwarding->i));
}

static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->i, (void*)src->i, 8/*BLOCK_FIELD_IS_BYREF*/);}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->i, 8/*BLOCK_FIELD_IS_BYREF*/);}

static struct __main_block_desc_0 {
    size_t reserved;
    size_t Block_size;
    void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
    void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};

int main()
{
    __attribute__((__blocks__(byref))) __Block_byref_i_0 i = {(void*)0,(__Block_byref_i_0 *)&i, 0, sizeof(__Block_byref_i_0), 1};

    void (*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, (__Block_byref_i_0 *)&i, 570425344));

    ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);

    printf("%d",(i.__forwarding->i));

    return 0;
}
```
从源码我们能发现，带有\_\_block的变量也被转化成了一个结构体\_\_Block\_byref\_i\_0,这个结构体有5个成员变量。第一个是isa指针，第二个是指向自身类型的\_\_forwarding指针，通过\_\_forwarding指针访问成员变量i，第三个是一个标记flag，第四个是它的大小，第五个是变量值，名字和变量名同名。
``` objc
struct __Block_byref_i_0 {
    void *__isa;
    __Block_byref_i_0 *__forwarding;
    int __flags;
    int __size;
    int i;
};
```
\_\_forwarding指针的作用就是针对堆的Block，把原来\_\_forwarding指针指向自己，换成指向\_NSConcreteMallocBlock上复制之后的\_\_block自己。然后堆上的变量的\_\_forwarding再指向自己。这样不管\_\_block怎么复制到堆上，还是在栈上，都可以通过``(i->__forwarding->i)``来访问到变量值。以下是访问\_\_block的图解：

{% asset_img forward.png %}

而在block外访问\_\_block修饰的变量也变成了``printf("%d",(i.__forwarding->i));``这样的形式。
### 捕获对象
将以下代码进行clang转换
``` objc
int main()
{
    id array = [[NSMutableArray alloc]init];    
    void (^block)(void) = ^{        
        [array addObject:@1];        
        NSLog(@"array count = %ld",[array count]);       
    };   
    block();    
    NSLog(@"array count = %ld",[array count]);   
    return 0;
}
```

得到大致的代码：
``` objc
struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    id array;

    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, id _array, int flags=0) : array(_array) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    id array = __cself->array; // bound by copy

    ((void (*)(id, SEL, ObjectType))(void *)objc_msgSend)((id)array, sel_registerName("addObject:"), (id)((NSNumber *(*)(Class, SEL, int))(void *)objc_msgSend)(objc_getClass("NSNumber"), sel_registerName("numberWithInt:"), 1));

    NSLog((NSString *)&__NSConstantStringImpl__var_folders_c3_9y1j0l292kqf2y4tcv868b4w0000gn_T_block_06bc3c_mi_0,((NSUInteger (*)(id, SEL))(void *)objc_msgSend)((id)array, sel_registerName("count")));
}

static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->array, (void*)src->array, 3/*BLOCK_FIELD_IS_OBJECT*/);}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->array, 3/*BLOCK_FIELD_IS_OBJECT*/);}

static struct __main_block_desc_0 {
    size_t reserved;
    size_t Block_size;
    void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
    void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};

int main()
{
    id array = ((NSMutableArray *(*)(id, SEL))(void *)objc_msgSend)((id)((NSMutableArray *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSMutableArray"), sel_registerName("alloc")), sel_registerName("init"));

    void (*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, array, 570425344));

    ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);

    NSLog((NSString *)&__NSConstantStringImpl__var_folders_c3_9y1j0l292kqf2y4tcv868b4w0000gn_T_block_06bc3c_mi_1,((NSUInteger (*)(id, SEL))(void *)objc_msgSend)((id)array, sel_registerName("count")));

    return 0;
}
```
可以看出block中的\_\_self指针即\_\_main\_block\_impl\_0结构体多了一个对象id array，大部分block的实现与使用\_\_block相同，有一个不同就是在block copy和dispose函数中最后一个参数不同，编译器自动给参数增加了备注：

| 截获目标 | 修饰参数 |
| :-------- | :-------- |
| 对象 | BLOCK_FIELD_IS_OBJECT |
| __block变量 | BLOCK_FIELD_IS_BYREF |

而对对象的访问即通过``id array = __cself->array``获取。
## 循环引用
在block中引用了持有block的对象的时候就会引起循环引用，由上一节block捕获对象可知，block捕获对象实质是在block结构体中拥有一个与捕获对象相同的实例变量，它会在结构体初始化时被赋值，而这个对象默认是用\_\_strong修饰的，这就导致了循环应用。如果使用了\_\_weak之后，表示 Block 对象的结构体中相对应的成员变量也将附有\_\_weak修饰符，\_\_weak修饰的变量不会持有对象，它用一张 weak 表（类似于引用计数表的散列表）来管理对象和变量。赋值的时候它会以赋值对象的地址作为key，变量的地址为value，注册到weak表中。一旦该对象被废弃，就通过对象地址在weak表中找到变量的地址，赋值为nil，然后将该条记录从weak表中删除。
在多线程情况下，可能weakSelf指向的对象会在block执行前被废弃，这在大部分情况下都是可以的，只会输出Self is nil，但在有些情况下(譬如weakSelf作为KVO的观察者被移除，或者block还没执行完self已经销毁)就会导致crash。这时可以在Block内部再持有一次weakSelf指向的对象``typeof(weakSelf) strongSelf = weakSelf``，延长该对象的生命周期，这就是所谓的 weak-strong dance。
每次使用\_\_weak变量的时候，都会取出该变量指向的对象并retain，然后将该对象注册到autoreleasepool中，这是延长对象生命周期的关键，但这不会造成循环引用，当函数执行结束，变量超出作用域，过一会儿(一般一次RunLoop之后)，对象就被释放了。所以weak-strong dance的行为非常符合预期：延长捕获对象的生命周期，一旦block执行完，这个局部对象就被释放，而block也会被释放(如果被GCD之类的API copy过一次增加了引用计数，那最终也会被GCD释放)。
每使用一次\_\_weak变量就会把对象注册到autoreleasepool中，所以如果短时间内大量使用\_\_weak变量的话，会导致注册到autoreleasepool中的对象大量增加，占用一定内存。而weak-strong dance恰好无意中解决了这个隐患，在执行block时，把\_\_weak变量(weakSelf)赋值给一个临时变量(strongSelf)，之后一直都使用这个临时变量，所以\_\_weak变量只使用了一次，也就只有一个对象注册到autoreleasepool中。
# 协议
## 数据结构
``` objc
typedef struct objc_object Protocol;
```
其实协议就是一个对象结构体
## 操作函数
### **objc_**:
``` objc
// 返回指定的协议
Protocol * objc_getProtocol ( const char *name );
// 获取运行时所知道的所有协议的数组
Protocol ** objc_copyProtocolList ( unsigned int *outCount );
// 创建新的协议实例
Protocol * objc_allocateProtocol ( const char *name );
// 在运行时中注册新创建的协议
void objc_registerProtocol ( Protocol *proto );
```
### **protocol_**:
**get**: 协议，属性
``` objc
// 返回协议名
const char * protocol_getName ( Protocol *p );

// 获取协议的指定属性
objc_property_t protocol_getProperty ( Protocol *proto, const char *name, BOOL isRequiredProperty, BOOL isInstanceProperty );
```
**copy**：协议列表，属性列表
``` objc
// 获取协议中的属性列表
objc_property_t * protocol_copyPropertyList ( Protocol *proto, unsigned int *outCount );
// 获取协议采用的协议
Protocol ** protocol_copyProtocolList ( Protocol *proto, unsigned int *outCount );
```
**add**：属性，方法，协议
``` objc
// 为协议添加方法
void protocol_addMethodDescription ( Protocol *proto, SEL name, const char *types, BOOL isRequiredMethod, BOOL isInstanceMethod );

// 添加一个已注册的协议到协议中
void protocol_addProtocol ( Protocol *proto, Protocol *addition );

// 为协议添加属性
void protocol_addProperty ( Protocol *proto, const char *name, const objc_property_attribute_t *attributes, unsigned int attributeCount, BOOL isRequiredProperty, BOOL isInstanceProperty );
```
**isEqual**：判断两协议等同
``` objc
// 测试两个协议是否相等
BOOL protocol_isEqual ( Protocol *proto, Protocol *other );
```
**comform**：判断是否遵循协议
``` objc
// 查看协议是否采用了另一个协议
BOOL protocol_conformsToProtocol ( Protocol *proto, Protocol *other );
```
# 类目
``` objc
typedef struct objc_category *Category;

struct objc_category {
char *category_name     OBJC2_UNAVAILABLE; // 分类名
char *class_name        OBJC2_UNAVAILABLE; // 分类所属的类名
struct objc_method_list *instance_methods    OBJC2_UNAVAILABLE; // 实例方法列表
struct objc_method_list *class_methods       OBJC2_UNAVAILABLE; // 类方法列表
struct objc_protocol_list *protocols         OBJC2_UNAVAILABLE; // 分类所实现的协议列表
}  

// objc-runtime-new.h中定义：

struct category_t {
const char *name;         // name 是指 class_name 而不是 category_name
classref_t cls;           // cls是要扩展的类对象，编译期间是不会定义的，而是在Runtime阶段通过name对应到对应的类对象
struct method_list_t *instanceMethods;       
struct method_list_t *classMethods;
struct protocol_list_t *protocols;
struct property_list_t *instanceProperties;    // instanceProperties表示Category里所有的properties，(这就是我们可以通过objc_setAssociatedObject和objc_getAssociatedObject增加实例变量的原因，)不过这个和一般的实例变量是不一样的
};
```
category就是定义方法的结构体，instance_methods列表是objc_class中方法列表的一个子集，class_methods列表是元类方法列表的一个子集。由其结构成员可知，category为什么不能添加成员变量（可添加属性，只有setter/getter方法）。

给category添加方法后，category_list会生成method list。这个方法列表是倒序添加的，也就是说，新生成的category的方法会先于旧的category的方法插入。category的方法会优先于类方法执行。
