---
title: Objective-C内部实现浅谈-Block
date: 2015-08-22 11:05:55
tags: [Objective-C,Runtime]
categories: 技术
---
# Block
在 OC 中闭包的语法看上去特别别扭，但他实际上是作为最普通的 C 语言源代码来处理的，在实际编译时无法转换成我们能够理解的源代码，因此通过 clang (LLVM 编译器)先将代码进行转化成 C++ 代码。在这里我们将能看到我们想要的信息。接下去本文主要介绍一下 block 的实现原理。

<!--more-->

如创建一个``block.m``文件，加入内容

``` c
int main()
{
    void (^block)(void) = ^{printf("Block\n");};
    block();
    return 0;
}
```
使用 clang 命令进行转换
```
clang -rewrite-objc block.m
```
转换后将形成一个约520行左右的 cpp 文件，

转换后有关 block 的内容大致如下：
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

static void __main_block_func_0(struct __main_block_impl_0 *__cself)
{
    printf("Block\n");
}

static struct __main_block_desc_0
{
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
block 的定义大致是这样的
``` objc
struct __block_impl {
    void *isa;
    int Flags;
    int Reserved;
    void *FuncPtr;
};
```
可见 block 结构体中有 isa 指针，因此在 OC 中 block 也是当做对象来使用的。而这个 isa 指针指向的有三中类型：
* _NSConcreteStackBlock
* _NSConcreteGlobalBlock
* _NSConcreteMallocBlock

而从其名字变可知其三种类对象内存分配的区域是不同的。下表说明了三种类的存储域：

| 类                      | 设置对象的存储域      |
| :--------------------- | :------------ |
| _NSConcreteStackBlock  | 栈             |
| _NSConcreteGlobalBlock | 数据存储区（.data区） |
| _NSConcreteMallocBlock | 堆             |

如果 block 在记述全局变量的地方被设置或者 block 没有捕获外部变量，那就生成一个 _NSConcreteGlobalBlock 实例。其它情况都会生成一个 _NSConcreteStackBlock 实例，也就是说，它是在栈上的，所以一旦它所属的变量超出了变量作用域，该 block 就被废弃了。而当发生以下任一情况时：
* 手动调用 block 的实例方法 copy
* block 作为函数返回值返回
* 将 block 赋值给附有 __strong 修饰符的成员变量
* 在方法名中含有 usingBlock 的 Cocoa 框架方法或 GCD 的 API 中传递 block

如果此时 block 在栈上，那就复制一份到堆上，并将复制得到的 block 实例的 isa 指针设为 _NSConcreteMallocBlock：
``` objc
imply.isa = &__NSConcreteMallocBlock;
```
## 实现原理
通过刚刚那个小小的例子来逐步分析一下 block 的实现原理。
首先例子中 block 语句``^{printf("Block\n");};``被转换成了
``` objc
static void __main_block_func_0(struct __main_block_impl_0 *__cself)
{
    printf("Block\n");
}
```
通过 block 的匿名函数实际被作为简单的 C 语言函数来处理，而 block 所属于的函数名(main)和在该函数出现的顺序值(0)来给 clang 转换的函数命名，该函数的参数 \_\_cself 相当于 OC 中的 self，指向 block 值的变量。
### \_\_cself
\_\_cself 是 \_\_main_block_impl_0 结构体的指针，声明如下：
``` objc
struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
}
```
第一个成员变量 impl 即 block 对象的函数指针，第二个 Desc 指针：
``` objc
static struct __main_block_desc_0 {
    size_t reserved;
    size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
```
记录了今后版本升级所需的区域和 block 的大小。
而 \_\_main_block_impl\_0 结构体实际还包含了其构造函数``__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0)``
``` objc
__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
}
```
### 构造函数
接下来我们看一下 \_\_main_block_impl\_0 结构体的构造函数，``impl.isa = &_NSConcreteStackBlock;``声明了 block 的类型为 _NSConcreteStackBlock ，当构造函数调用时，即执行``void (^block)(void) = ^{printf("Block\n");};``这步时：
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
这样整个 block 就声明完成了
### 调用
调用 block，即``block();``这一步转换后是这样的：
``` objc
((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
```
这就是简单的使用函数指针调用函数，由 block 语法转换的 \_\_main_block\_func\_0 函数的指针被赋值在成员变量 FuncPtr 中，另外也说明了 \_\_main_block\_func\_0 函数的参数 \_\_cself 指向 block 值，在源码中可以看出 block 正是作为参数进行了传递。
## 捕获外部变量
block 捕获外部变量其实可分为三种情况：
* 捕获变量的瞬时值
* 捕获 __block 变量
* 捕获对象

说到外部变量，我们要先说一下 C 语言中变量有哪几种。一般可以分为一下5种：
* 自动变量
* 函数参数( block 中先不考虑这种情况)
* 静态变量
* 静态全局变量
* 全局变量

### 捕获变量的瞬间值
将全局变量、静态全局变量、静态变量在 block 中捕获 __main_block_func\_0 函数大致是这样的：
``` objc
static void __main_block_func_0(struct __main_block_impl_0 *__cself)
{
    global_i ++;
    static_global_j ++;
    (*static_k) ++;
}
```
首先全局变量和静态全局变量因为是全局的，作用域很广，所以 block 捕获了它们进去之后，在 block 里面进行操作，block 结束之后，它们的值依旧可以得以保存下来。
如果 block 外面还有很多自动变量，静态变量，等等，这些变量在 block 里面并不会被使用到。那么这些变量并不会被 block 捕获进来，也就是说并不会在构造函数里面传入它们的值，block 捕获外部变量仅仅只捕获 block 闭包里面会用到的值，其他用不到的值，它并不会去捕获。
静态变量传递给 block 是内存地址值，所以能在 block 里面直接改变值。静态变量的这种做法似乎也适用于自动变量的访问，但是为什么没有这么做呢？
实际上，在由于 blcok 语法生成的值 block 上，可以存有超过其变量作用域的被截获对象的自动变量，变量作用域结束的同时，原来的自动变量被废弃，因此 block 中超过变量作用域而存在的变量同静态变量一样，将不能通过指针访问原来的自动变量。而静态局部变量具有局部作用域，它只被初始化一次，自从第一次被初始化直到程序运行结束都一直存在，它和全局变量的区别在于全局变量对所有的函数都是可见的，而静态局部变量只对定义自己的函数体始终可见，所以简单来说就是**静态局部变量不会被销毁，所以可以通过指针来访问，自动变量会被销毁，所以不能通过指针访问**
### 捕获\_\_block变量
\_\_block 的使用就不多介绍了，他实际就是修饰用于指定将变量值设置到哪个存储域中。
使用 \_\_block 修饰自动变量转化后的代码主要的变换是这样的：
``` objc
static void __main_block_func_0(struct __main_block_impl_0 *__cself)
{
    __Block_byref_i_0 *i = __cself->i; /* bound by ref */
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
从源码我们能发现，带有 \_\_block 的变量也被转化成了一个结构体 \_\_Block\_byref\_i\_0，这个结构体有5个成员变量。第一个是 isa 指针，第二个是指向自身类型的 \_\_forwarding 指针，通过 \_\_forwarding 指针访问成员变量 i，第三个是一个标记 flag，第四个是它的大小，第五个是变量值，名字和变量名同名。
``` objc
struct __Block_byref_i_0 {
    void *__isa;
    __Block_byref_i_0 *__forwarding;
    int __flags;
    int __size;
    int i;
};
```
\_\_forwarding 指针的作用就是针对堆的 Block，把原来 \_\_forwarding 指针指向自己，换成指向 \_NSConcreteMallocBlock 上复制之后的 \_\_block 自己。然后堆上的变量的 \_\_forwarding 再指向自己。这样不管 \_\_block 怎么复制到堆上，还是在栈上，都可以通过``(i->__forwarding->i)``来访问到变量值。以下是访问 \_\_block 的图解：

{% asset_img forward.png %}

而在 block 外访问 \_\_block 修饰的变量也变成了``printf("%d",(i.__forwarding->i));``这样的形式。
### 捕获对象
将以下代码进行 clang 转换
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

static void __main_block_func_0(struct __main_block_impl_0 *__cself)
{
    id array = __cself->array; /* bound by copy */
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
可以看出 block 中的 \_\_self 指针即 \_\_main\_block\_impl\_0 结构体多了一个对象 id array，大部分 block 的实现与使用 \_\_block 相同，有一个不同就是在 block copy 和 dispose 函数中最后一个参数不同，编译器自动给参数增加了备注：

| 截获目标      | 修饰参数                  |
| :-------- | :-------------------- |
| 对象        | BLOCK_FIELD_IS_OBJECT |
| __block变量 | BLOCK_FIELD_IS_BYREF  |

而对对象的访问即通过``id array = __cself->array``获取。
## 循环引用
在 block 中引用了持有 block 的对象的时候就会引起循环引用，由上一节 block 捕获对象可知，block 捕获对象实质是在 block 结构体中拥有一个与捕获对象相同的实例变量，它会在结构体初始化时被赋值，而这个对象默认是用 \_\_strong 修饰的，这就导致了循环应用。如果使用了 \_\_weak 之后，表示 Block 对象的结构体中相对应的成员变量也将附有 \_\_weak 修饰符，\_\_weak 修饰的变量不会持有对象，它用一张 weak 表(类似于引用计数表的散列表)来管理对象和变量。赋值的时候它会以赋值对象的地址作为 key，变量的地址为 value，注册到 weak 表中。一旦该对象被废弃，就通过对象地址在 weak 表中找到变量的地址，赋值为 nil，然后将该条记录从 weak 表中删除。
在多线程情况下，可能 weakSelf 指向的对象会在 block 执行前被废弃，这在大部分情况下都是可以的，只会输出 Self is nil，但在有些情况下(譬如 weakSelf 作为 KVO 的观察者被移除，或者 block 还没执行完 self 已经销毁)就会导致 crash。这时可以在 Block 内部再持有一次 weakSelf 指向的对象``typeof(weakSelf) strongSelf = weakSelf``，延长该对象的生命周期，这就是所谓的 weak-strong dance。
每次使用 \_\_weak 变量的时候，都会取出该变量指向的对象并 retain，然后将该对象注册到 autoreleasepool 中，这是延长对象生命周期的关键，但这不会造成循环引用，当函数执行结束，变量超出作用域，过一会儿(一般一次 RunLoop 之后)，对象就被释放了。所以 weak-strong dance 的行为非常符合预期：延长捕获对象的生命周期，一旦 block 执行完，这个局部对象就被释放，而 block 也会被释放(如果被 GCD 之类的 API copy 过一次增加了引用计数，那最终也会被 GCD 释放)。
每使用一次 \_\_weak 变量就会把对象注册到 autoreleasepool 中，所以如果短时间内大量使用 \_\_weak 变量的话，会导致注册到 autoreleasepool 中的对象大量增加，占用一定内存。而 weak-strong dance 恰好无意中解决了这个隐患，在执行 block 时，把 \_\_weak 变量(weakSelf)赋值给一个临时变量(strongSelf)，之后一直都使用这个临时变量，所以 \_\_weak 变量只使用了一次，也就只有一个对象注册到 autoreleasepool 中。
