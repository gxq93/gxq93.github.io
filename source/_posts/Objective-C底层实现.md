---
title: Objective-C底层实现
date: 2015-12-22 18:18:49
tags:
---

# Runtime
OC实质是围绕类的动态配置和消息传递，通过操作函数来配置类信息，通过``msgSend``函数传递消息。作为面向对象语言，“对象”(object)就是“基本构造单元”(building block)，开发者可以通过对象来存储并传递数据，在对象之间传递数据并执行任务的过程就叫做“消息传递”。
当应用程序运行起来之后，为其提供相关支持的代码叫做“Objective-C运行期环境”，它提供了一些使得对象之间能够传递信息的重要函数，并且包含创建类实例所用的全部逻辑。
# 类和对象
## 数据结构
在OC中类和对象是这样定义的
```objc
/// An opaque type that represents an Objective-C class.
typedef struct objc_class *Class;
/// Represents an instance of a class.
struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
};
/// A pointer to an instance of a class.
typedef struct objc_object *id;
```
其中``objc_class``是一个结构体，里面包含的内容如下，
```objc
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;
    if !__OBJC2__
    Class super_class            //父类
    const char *name             //类名
    long version                 //类的版本信息，默认为0
    long info                    //类信息，供运行期使用的一些位标识
    long instance_size                      //类的实例变量大小
    struct objc_ivar_list *ivars            //类的成员变量链表
    struct objc_method_list **methodLists   //方法定义的链表
    struct objc_cache *cache                //方法缓存
    struct objc_protocol_list *protocols    //协议链表
    endif
} OBJC2_UNAVAILABLE;
```
可以看出类的本质是一个结构体，在这个结构体中有两个``Class``类型的参数``isa``和``super_class``，这两个参数是实现函数的重要映射，决定找到存放在哪个类的方法实现。（isa用于自省确定所属类，super_class确定继承关系）。

实例对象的isa指针指向类，类的isa指针指向其元类（``metaClass``）。对象就是一个含isa指针的结构体。类存储实例对象的方法列表，元类中只记录类名，类方法列表等，元类也是类对象。
```objc
/// An opaque type that represents an Objective-C class.
typedef struct objc_class *Class;
/// Represents an instance of a class.
struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
};
/// A pointer to an instance of a class.
typedef struct objc_object *id;
```
这是实例对象的结构，当创建实例对象时，分配的内存包含一个``objc_object``数据结构，然后是类到父类直到根类NSObject的实例变量的数据。NSObject类的``alloc``和``allocWithZone:``方法使用函数``class_createInstance``来创建objc_object数据结构。
向一个Objective-C对象发送消息时，运行时库会根据实例对象的isa指针找到这个实例对象所属的类。Runtime库会在类的方法列表由super_class指针找到父类的方法列表直至根类NSObject中去寻找与消息对应的``selector``指向的方法。找到后即运行这个方法。(如下图)

{% asset_img runtime1.png %}

* 1、isa：实例对象->类->元类->（不经过父元类）直接到根元类（NSObject的元类），根元类的isa指向自己；
* 2、 superclass：类->父类->...->根类NSObject，元类->父元类->...->根元类->根类,NSObject的superclass指向nil。
## 操作函数
类和对象的操作函数名只要以类对象``class_``为前缀，实例对象``object_``为前缀
### **class_**:
**get**:类名，父类，元类；实例变量，成员变量；属性；实例方法，类方法，方法实现
```objc
// 获取类的类名
const char * class_getName ( Class cls );
// 获取类的父类
Class class_getSuperclass ( Class cls );
// 获取实例大小
size_t class_getInstanceSize ( Class cls );
// 获取类中指定名称实例成员变量的信息
Ivar class_getInstanceVariable ( Class cls, const char *name );
// 获取类成员变量的信息
Ivar class_getClassVariable ( Class cls, const char *name );
// 获取指定的属性
objc_property_t class_getProperty ( Class cls, const char *name );
//方法：具体内容下文讲//
// 获取实例方法
Method class_getInstanceMethod ( Class cls, SEL name );
// 获取类方法
Method class_getClassMethod ( Class cls, SEL name );
// 获取方法的具体实现
IMP class_getMethodImplementation ( Class cls, SEL name );
IMP class_getMethodImplementation_stret ( Class cls, SEL name );
```
**copy**:成员变量列表；属性列表；方法列表；协议列表
```objc
// 获取整个成员变量列表
Ivar * class_copyIvarList ( Class cls, unsigned int *outCount );
// 获取属性列表
objc_property_t * class_copyPropertyList ( Class cls, unsigned int *outCount );
// 获取所有方法的列表
Method * class_copyMethodList ( Class cls, unsigned int *outCount );
// 获取类实现的协议列表
Protocol * class_copyProtocolList ( Class cls, unsigned int *outCount );
```
**add**:成员变量；属性；方法；协议(添加成员变量只能在运行时创建的类，且不能为元类)
```objc
// 添加成员变量
BOOL class_addIvar ( Class cls, const char *name, size_t size, uint8_t alignment, const char *types );
// 添加属性
BOOL class_addProperty ( Class cls, const char *name, const objc_property_attribute_t *attributes, unsigned int attributeCount );
// 添加方法
BOOL class_addMethod ( Class cls, SEL name, IMP imp, const char *types );
// 添加协议
BOOL class_addProtocol ( Class cls, Protocol *protocol );
```
**replace**：属性；方法
```objc
// 替换类的属性
void class_replaceProperty ( Class cls, const char *name, const objc_property_attribute_t *attributes, unsigned int attributeCount );
// 替代方法的实现
IMP class_replaceMethod ( Class cls, SEL name, IMP imp, const char *types );
```
**respond**:响应方法判断（内省）
```objc
// 类实例是否响应指定的selector
BOOL class_respondsToSelector ( Class cls, SEL sel );
```
**isMetaClass**:元类判断（内省）
```objc
// 判断给定的Class是否是一个元类
BOOL class_isMetaClass ( Class cls );
```
**conform**：遵循协议判断（内省）
```objc
// 返回类是否实现指定的协议
BOOL class_conformsToProtocol ( Class cls, Protocol *protocol );
```
### **object_**:
**get**:实例变量；成员变量；类名；类；元类；关联对象
```objc
// 获取对象实例变量
Ivar object_getInstanceVariable ( id obj, const char *name, void **outValue );
// 获取对象中实例变量的值
id object_getIvar ( id obj, Ivar ivar );
// 获取对象的类名
const char * object_getClassName ( id obj );
// 获取对象的类
Class object_getClass ( id obj );
Class objc_getClass ( const char *name );
// 返回指定类的元类
Class objc_getMetaClass ( const char *name );
//获取关联对象
id objc_getAssociatedObject(self, &myKey);
```
**copy**:对象；类；类列表；协议列表
```objc
// 获取指定对象的一份拷贝
id object_copy ( id obj, size_t size );
// 创建并返回一个指向所有已注册类的指针列表
Class * objc_copyClassList ( unsigned int *outCount );
```
**set**:实例变量；类；类列表；协议；关联对象
```objc
// 设置类实例的实例变量的值
Ivar object_setInstanceVariable ( id obj, const char *name, void *value );
// 设置对象中实例变量的值
void object_setIvar ( id obj, Ivar ivar, id value );
//设置关联对象
void objc_setAssociatedObject(self, &myKey, anObject, OBJC_ASSOCIATION_RETAIN);
```
**dispose**:对象
```objc
// 释放指定对象占用的内存
id object_dispose ( id obj );
```
### 动态创建/销毁类、对象
动态创建/销毁类：
```objc
// 创建一个新类和元类
Class objc_allocateClassPair ( Class superclass, const char *name, size_t extraBytes );
// 销毁一个类及其相关联的类
void objc_disposeClassPair ( Class cls );
// 在应用中注册由objc_allocateClassPair创建的类
void objc_registerClassPair ( Class cls );
```
动态创建/销毁对象：
```objc
// 创建类实例
id class_createInstance ( Class cls, size_t extraBytes );
// 在指定位置创建类实例
id objc_constructInstance ( Class cls, void *bytes );
// 销毁类实例
void * objc_destructInstance ( id obj );
```
# 实例变量和属性
“属性”(property)是OC的一项特性，用于封装对象中的数据。OC对象通常会把其所需要的数据保存为各种实例变量。实例变量一般通过“存取方法”(access method)来访问。其中，“获取方法”(getter)用于读取变量值，而“设置方法”(setter)用于写入变量值。而属性可以更容易地依照类对象来访问存放于其中的数据。
## 数据结构：
### 成员变量Ivar
```objc
typedef struct objc_ivar *Ivar;
struct objc_ivar {
    char *ivar_name                 OBJC2_UNAVAILABLE;  // 变量名
    char *ivar_type                 OBJC2_UNAVAILABLE;  // 变量类型
    int ivar_offset                 OBJC2_UNAVAILABLE;  // 基地址偏移字节
    #ifdef __LP64__
    int space                       OBJC2_UNAVAILABLE;
    #endif
}
```
### 属性
```objc
typedef struct objc_property *objc_property_t;
```
``objc_property_attribute_t``（属性的特性有：返回值、是否为atomic、getter/setter名字、是否为dynamic、背后使用的ivar名字、是否为弱引用等）
```objc
typedef struct {
    const char *name;           // 特性名
    const char *value;          // 特性值
} objc_property_attribute_t;
```
## 操作函数：
### **ivar_**：
**get**:成员变量名，成员变量类型编码，成员变量的偏移量
```objc
// 获取成员变量名
const char * ivar_getName ( Ivar v );
// 获取成员变量类型编码
const char * ivar_getTypeEncoding ( Ivar v );
// 获取成员变量的偏移量
ptrdiff_t ivar_getOffset ( Ivar v );
```
### **property_**：
**get**:属性名，属性特性描述字符串，属性中指定的特性，属性的特性列表
```objc
// 获取属性名
const char * property_getName ( objc_property_t property );
// 获取属性特性描述字符串
const char * property_getAttributes ( objc_property_t property );
// 获取属性中指定的特性
char * property_copyAttributeValue ( objc_property_t property, const char *attributeName );
// 获取属性的特性列表
objc_property_attribute_t * property_copyAttributeList ( objc_property_t property, unsigned int *outCount );
```
# 属性相关的内容
## 实例变量与属性
Objective-C中不提倡使用
```objc
@interface Person:NSObject {
    NSString *_firstName;
    NSString *_secondName;
}
```
虽然这样可以定义实例变量的作用域，但是对象布局在编译期就已经固定了，只要访问``_firstName``变量的代码，编译器就把其替换为偏移量（offset），这个偏移量是硬编码，表示该变量距离存放对象的内存区域的起始地址有多远，如果又加了一个实例变量，如：
```objc
@interface Person:NSObject {
    NSString *_family;
    NSString *_firstName;
    NSString *_secondName;
}
```
``_firestName``的偏移量就指向了``_family``，那么在修改定义后就必须重新编译，否则就会出错。Objective-C做法是把实例变量当作一种存储偏移量所用的“特殊变量”，交给“类对象”来管理，偏移量会在运行期查找，如果类的定义变了，存储的偏移量也就变了。
这个问题还有一种方法，尽量不要直接访问实例变量，而通过存取方法来做，也就是属性。虽然属性最终还是通过实例变量来实现，但是它更简洁。如果使用属性的话，编译器会自动编写访问这些属性所需方法，这个过程叫做“自动合成”。这个过程在编译期执行，除了生成方法代码外，编译器还自动向类中添加适当类型的实例变量，并在属性名前加前缀``_``。
也可以通过``@synthesize``语法来定义实例变量的名字
```objc
@synthesize firesName = _myFirstName;
```
这会将生成的实例名字命名为`` _myFirstName``
## 属性特质(attribute)
属性有各种特质影响编译期所产生的存取方法
* 原子性（atomic/nonatomic）
* 读／写权限(readwrite/readonly)
* 内存管理语义(assign/strong/weak/copy/unsafe_unretained)
* 方法名
getter=< name >   如：``@property(nonatomic,getter=isOn) BOOL on;``
setter=< name >
上述特质可通过前文runtime方法得到。
## 访问实例变量
在对象之外访问实例变量应该通过属性来访问，而在对象内部则可根据情况分类讨论。
如果直接访问实例变量，由于不经过方法派发，访问速度会比访问属性更快，编译器所产生的代码会直接访问保存对象实例的那块内存，但是直接访问实例变量不会触发“设置方法”，比如如果在ARC下直接访问一个声明为copy的属性，那么并不会拷贝这个属性，只会保留新值释放旧值。因此通过getter方法定义的懒加载方法也不会被调用。也不会触发``KVO``。
因此可以在写入实例变量时通过“设置方法”来做，而在读取实例变量时直接访问（懒加载除外）。
# 方法
Objective-C最重要的一个特性就是“消息传递”，消息有名称(name)和选择器(selector)，可以接受参数并可能有返回值。
## C函数
C使用的是静态绑定，在编译期就能决定运行时所应调用的函数（不考虑内联）函数地址硬解码在指令中，而如果出现只有一个函数调用指令，不过待调用的函数地址无法硬解码在指令之中，就得使用动态绑定了，在运行期把代码读取出来：
``` c
void printHello(){
    printf("hello");
}
void printWorld(){
    printf("world");
}
void doSomething(){
    void (*func)()
    if(type == 0){
    func = printHello();
    }
    else {
    func = printWorld();
    }
    func();
}
```
## Objective-C方法
Objective-C中如果向某对象传递消息，就会使用动态绑定机制来决定需要调用的方法
过程：
```objc
id returnValue = [someObject messageName:parameter];
```
someObject叫做``receiver``，messageName叫做``selector``，selector与参数``parameter``合起来成为``message``，编译器看到message后，将其转换成一条标准的c函数``objc_msgSend``。
``objc_msgSend``其原型如下：
```objc
void objc_msgSend(id self, SEL cmd, ...)
```
这是个“参数个数可变的函数”（variadic function），能接受两个或两个以上的参数。第一个参数代表接收者，第二个参数代表选择器（``SEL``是选择器的类型），后续参数就是消息中的那些参数，其顺序不变。选择器指的就是方法的名字。
### **self**:
``self``和``cmd``是隐藏参数，在编译期被插入实现代码。
self：指向消息的接受者target的对象类型，作为一个占位参数，消息传递成功后self将指向消息的receiver。
当向一般对象发送消息时，调用objc_msgSend，每个方法的实现的第一个参数即为self。
### **super**:
``super``并不是隐藏参数，它实际上只是一个”编译器标示符”，它负责告诉编译器，当调用方法时，跳过当前类去调用父类的方法，而不是本类中的方法。实际上给super发消息时，super还是与self指向的是相同的消息接收者。
```objc
struct objc_super {
    __unsafe_unretained id receiver;
    __unsafe_unretained Class super_class;
};
```
当向super发送消息时，调用的是``objc_msgSendSuper``。如果返回值是一个结构体，则会调用``objc_msgSend_stret``或``objc_msgSendSuper_stret``。
```objc
id objc_msgSendSuper ( struct objc_super *super, SEL op, ... );
```
### **SEL**:
``SEL``是选择器的类型，是表示一个方法的selector的指针,映射方法的名字。Objective-C在编译时，会依据每一个方法的名字、参数序列，生成一个唯一的整型标识(Int类型的地址)，这个标识就是SEL。
SEL的作用是作为``IMP``的KEY，存储在NSSet中，便于hash快速查询方法。SEL不能相同，对应方法可以不同。所以在Objective-C同一个类(及类的继承体系)中，不能存在2个同名的方法，就算参数类型不同。多个方法可以有同一个SEL。
不同的类可以有相同的方法名。不同类的实例对象执行相同的selector时，会在各自的方法列表中去根据selector去寻找自己对应的IMP。
相关概念：类型编码（Type Encoding）
编译器将每个方法的返回值和参数类型编码为一个字符串，并将其与方法的selector关联在一起。可以使用``@encode``编译器指令来获取它。
```objc
typedef struct objc_selector *SEL;
```
SEL本质是一个字符串！
### **IMP**:
``IMP``是指向实现函数的指针，通过SEL取得IMP后，我们就获得了最终要找的实现函数的入口。
```objc
typedef id (*IMP)(id, SEL, ...)
```
### **Method**:
这个结构体相当于在``SEL``和``IMP``之间作了一个绑定。这样有了SEL，我们便可以找到对应的IMP，从而调用方法的实现代码。（在运行时才将SEL和IMP绑定, 动态配置方法）。
```objc
typedef struct objc_method *Method;
struct objc_method {
    SEL method_name             // 方法名
    char *method_types          // 参数类型
    IMP method_imp              // 方法实现
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
方法调用最先是在方法缓存，也就是快速映射表里找的，方法调用是懒调用，第一次调用时加载后加到快速映射表里。一个objc程序启动后，需要进行类的初始化、调用方法时的cache初始化，再发送消息的时候就直接走缓存（引申：``+load``方法和``+initialize``方法。load方法是首次加载类时调用，绝对只调用一次；initialize方法是首次给类发消息时调用，通常只调用一次，但如果它的子类初始化时未定义initialize方法，则会再调用一次它的initialize方法）。
```objc
struct objc_cache {
    // 缓存bucket的总数
    unsigned int mask /* total = mask + 1 */                 
    // 实际缓存bucket的总数
    unsigned int occupied                                    
    // 指向Method数据结构指针的数组
    Method buckets[1]                                        
};
```
### 尾调用优化
如果某函数的最后一项操作是调用另外一个函数，那么就可以运用“尾调用优化”技术。编译器会生成调转至另一函数所需的指令码，而且不会向调用堆栈中推入新的“栈帧”（frame stack）。只有当某函数的最后一个操作仅仅是调用其他函数而不会将其返回值另作他用时，才能执行“尾调用优化”。这项优化对objc_msgSend非常关键，如果不这么做的话，那么每次调用Objective-C方法之前，都需要为调用objc_msgSend函数准备“栈帧”，大家在“栈踪迹”（stack trace）中可以看到这种“栈帧”。此外，若是不优化，还会过早地发生“栈溢出”（stack overflow）现象。
## 操作函数
### **method_**：
**invoke**: 方法实现的返回值；
```objc
// 调用指定方法的实现
id method_invoke ( id receiver, Method m, ... );

// 调用返回一个数据结构的方法的实现
void method_invoke_stret ( id receiver, Method m, ... );
```
**get**:方法名；方法实现；参数与返回值相关
```objc
// 获取方法名
SEL method_getName ( Method m );
// 返回方法的实现
IMP method_getImplementation ( Method m );
// 获取描述方法参数和返回值类型的字符串
const char * method_getTypeEncoding ( Method m );
// 返回方法的参数的个数
unsigned int method_getNumberOfArguments ( Method m );
// 通过引用返回方法指定位置参数的类型字符串
void method_getArgumentType ( Method m, unsigned int index, char *dst, size_t dst_len );
```
**copy**:返回值类型，参数类型
```objc
// 获取方法的返回值类型的字符串
char * method_copyReturnType ( Method m );
// 获取方法的指定位置参数的类型字符串
char * method_copyArgumentType ( Method m, unsigned int index );
// 通过引用返回方法的返回值类型字符串
void method_getReturnType ( Method m, char *dst, size_t dst_len );
```
**set**：方法实现
```objc
// 设置方法的实现
IMP method_setImplementation ( Method m, IMP imp );
```
**exchange**：交换方法实现
```objc
// 交换两个方法的实现
void method_exchangeImplementations ( Method m1, Method m2 );
```
**description**:方法描述
```objc
// 返回指定方法的方法描述结构体
struct objc_method_description * method_getDescription ( Method m );
```
### **SEL_**:
```objc
// 返回给定选择器指定的方法的名称
const char * sel_getName ( SEL sel );
// 在Objective-C Runtime系统中注册一个方法，将方法名映射到一个选择器，并返回这个选择器
SEL sel_registerName ( const char *str );
// 在Objective-C Runtime系统中注册一个方法
SEL sel_getUid ( const char *str );
// 比较两个选择器
BOOL sel_isEqual ( SEL lhs, SEL rhs );
```
## 消息传递过程
* 检查target是否为nil。如果为nil，直接cleanup，然后return。(这就是我们可以向nil发送消息的原因。)
如果方法返回值是一个对象，那么发送给nil的消息将返回nil；如果方法返回值为指针类型，其指针大小为小于或者等于**sizeof(void*)，float，double，long double 或者long long的整型**标量，发送给nil的消息将返回0；如果方法返回值为结构体,发送给nil的消息将返回0。结构体中各个字段的值将都是0；如果方法的返回值不是上述提到的几种情况，那么发送给nil的消息的返回值将是未定义的。
* 如果target非nil，在target的``Class``中根据``SEL``去找``IMP``。（因为同一个方法可能在不同的类中有不同的实现，所以我们需要依赖于接收者的类来找到的确切的实现）。
* 首先它找到``selector``对应的方法实现:
* 在target类的方法缓存列表里检查有没有对应的方法实现，有的话，直接调用。
* 比较请求的selector和类方法列表中的selector，对应的话，直接调用。
* 比较请求的selector和父类方法列表，父类的父类，直至根类，如果有对应，则直接调用。（方法重写拦截父类方法的原理）
* 调用方法实现，并将接收者对象及方法的所有参数传给它。
* 最后，将实现函数的返回值作为自己的返回值。

## 动态方法解析与消息转发
如果以上的类中没有找到对应的selector（一般保险起见先用``respondsToSelector:``内省判断），还可以利用消息转发机制依次执行以下流程：
* Method Resolution（动态方法解析）：
用所属类的类方法``+(BOOL)resolveInstanceMethod:``(实例方法)或者``+(BOOL)resolveClassMethod:``(类方法),在此方法里添加``class_addMethod``函数。一般用于@dynamic动态属性。（当一个属性声明为@dynamic，就是向编译器保证编译时不用管setter/getter实现，一定会在运行时实现）。
* Fast Forwarding （快速消息转发）：
如果上一步无法响应消息，调用``-(id)forwardingTargetForSelector:(SEL)aSelector``方法，将消息接受者转发到另一个对象target（不能为self，否则死循环）。
* Normal Forwarding（普通消息转发）：
如果上一步无法响应消息,调用方法签名``-(NSMethodSignature )methodSignatureForSelector:(SEL)aSelector``，方法签名目的将函数的参数类型和返回值封装。如果返回非nil，则创建一个NSInvocation对象利用方法签名和selector封装未被处理的消息，作为参数传递给``-(void)forwardInvocation:(NSInvocation )anInvocation``。

如果以上步骤（消息传递和消息转发）还是不能响应消息，则调动``doesNotRecognizeSelector：``方法，抛出异常。
```objc
unrecognized selector sent to instance
```

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
如果block在记述全局变量的地方被设置或者block没有捕获外部变量，那就生成一个 _NSConcreteGlobalBlock 实例。其它情况都会生成一个 _NSConcreteStackBlock 实例，也就是说，它是在栈上的，所以一旦它所属的变量超出了变量作用域，该block就被废弃了。而当发生以下任一情况时：
* 手动调用 Block 的实例方法copy
* Block 作为函数返回值返回
* 将 Block 赋值给附有__strong修饰符的成员变量
* 在方法名中含有usingBlock的 Cocoa 框架方法或 GCD 的 API 中传递 Block

如果此时block在栈上，那就复制一份到堆上，并将复制得到的block实例的isa指针设为 _NSConcreteMallocBlock：
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
通过block的匿名函数实际被作为简单的C语言函数来处理，而block所属于的函数名(main)和在该函数出现的顺序值(0)来给clang转换的函数命名，该函数的参数__cself相当于OC中的self，指向block值的变量。
### **__cself**
**__cself**是__main_block_impl_0结构体的指针，声明如下：
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
而**__main_block_impl_0**结构体实际还包含了其构造函数``__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0)``
``` objc
__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
  impl.isa = &_NSConcreteStackBlock;
  impl.Flags = flags;
  impl.FuncPtr = fp;
  Desc = desc;
}
```
### 构造函数
接下来我们看一下**__main_block_impl_0**结构体的构造函数，``impl.isa = &_NSConcreteStackBlock;``声明了block的类型为_NSConcreteStackBlock，当构造函数调用时，即执行``void (^block)(void) = ^{printf("Block\n");};``这步时：
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
这就是简单的使用函数指针调用函数，由block语法转换的**__main_block_func_0**函数的指针被赋值在成员变量FuncPtr中，另外也说明了**__main_block_func_0**函数的参数__cself指向block值，在源码中可以看出block正是作为参数进行了传递。
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
### 捕获**__block**变量
**__block**的使用就不多介绍了，他实际就是修饰用于指定将变量值设置到哪个存储域中。
使用**__block**修饰自动变量转化后的代码主要的变换是这样的：
``` objc
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    __Block_byref_i_0 *i = __cself->i; // bound by ref
    (i->__forwarding->i)++;
    printf("%d",(i->__forwarding->i));}

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
从源码我们能发现，带有**__block**的变量也被转化成了一个结构体**__Block_byref_i_0**,这个结构体有5个成员变量。第一个是isa指针，第二个是指向自身类型的**__forwarding**指针，通过**__forwarding**指针访问成员变量i，第三个是一个标记flag，第四个是它的大小，第五个是变量值，名字和变量名同名。
``` objc
struct __Block_byref_i_0 {
  void *__isa;
  __Block_byref_i_0 *__forwarding;
  int __flags;
  int __size;
  int i;
};
```
**__forwarding**指针的作用就是针对堆的Block，把原来**__forwarding**指针指向自己，换成指向_NSConcreteMallocBlock上复制之后的**__block**自己。然后堆上的变量的**__forwarding**再指向自己。这样不管**__block**怎么复制到堆上，还是在栈上，都可以通过``(i->__forwarding->i)``来访问到变量值。以下是访问**__block**的图解：

{% asset_img forward.png %}

而在block外访问**__block**修饰的变量也变成了``printf("%d",(i.__forwarding->i));``这样的形式。
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
可以看出block中的**__cself**指针即**__main_block_impl_0**结构体多了一个对象id array，大部分block的实现与使用**__block**相同，有一个不同就是在block copy和dispose函数中最后一个参数不同，编译器自动给参数增加了备注：

| 截获目标 | 修饰参数 |
| :-------- | :-------- |
| 对象 | BLOCK_FIELD_IS_OBJECT |
| __block变量 | BLOCK_FIELD_IS_BYREF |

而对对象的访问即通过``id array = __cself->array``获取。
## 循环引用
在block中引用了持有block的对象的时候就会引起循环引用，由上一节block捕获对象可知，block捕获对象实质是在block结构体中拥有一个与捕获对象相同的实例变量，它会在结构体初始化时被赋值，而这个对象默认是用**__strong**修饰的，这就导致了循环应用。如果使用了**__weak**之后，表示 Block 对象的结构体中相对应的成员变量也将附有**__weak**修饰符，**__weak**修饰的变量不会持有对象，它用一张 weak 表（类似于引用计数表的散列表）来管理对象和变量。赋值的时候它会以赋值对象的地址作为 key，变量的地址为 value，注册到 weak 表中。一旦该对象被废弃，就通过对象地址在 weak 表中找到变量的地址，赋值为 nil，然后将该条记录从 weak 表中删除。
在多线程情况下，可能weakSelf指向的对象会在block执行前被废弃，这在大部分情况下都是可以的，只会输出Self is nil，但在有些情况下(譬如weakSelf作为 KVO 的观察者被移除，或者block还没执行完self已经销毁)就会导致 crash。这时可以在 Block 内部再持有一次weakSelf指向的对象``typeof(weakSelf) strongSelf = weakSelf``，延长该对象的生命周期，这就是所谓的 weak-strong dance。
每次使用**__weak**变量的时候，都会取出该变量指向的对象并retain，然后将该对象注册到autoreleasepool中，这是延长对象生命周期的关键，但这不会造成循环引用，当函数执行结束，变量超出作用域，过一会儿(一般一次RunLoop之后)，对象就被释放了。所以weak-strong dance的行为非常符合预期：延长捕获对象的生命周期，一旦block执行完，这个局部对象就被释放，而block也会被释放(如果被GCD之类的API copy过一次增加了引用计数，那最终也会被GCD释放)。
每使用一次**_weak**变量就会把对象注册到autoreleasepool中，所以如果短时间内大量使用**_weak**变量的话，会导致注册到autoreleasepool中的对象大量增加，占用一定内存。而weak-strong dance恰好无意中解决了这个隐患，在执行block时，把**__weak**变量(weakSelf)赋值给一个临时变量(strongSelf)，之后一直都使用这个临时变量，所以**__weak**变量只使用了一次，也就只有一个对象注册到autoreleasepool中。
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
