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
**get**:
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
**copy**: 
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
**add**:(添加成员变量只能在运行时创建的类，且不能为元类)
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
**replace**：
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
**get**:
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
**copy**:
```objc
// 获取指定对象的一份拷贝
id object_copy ( id obj, size_t size );
// 创建并返回一个指向所有已注册类的指针列表
Class * objc_copyClassList ( unsigned int *outCount );
```
**set**: 
```objc
// 设置类实例的实例变量的值
Ivar object_setInstanceVariable ( id obj, const char *name, void *value );
// 设置对象中实例变量的值
void object_setIvar ( id obj, Ivar ivar, id value );
//设置关联对象
void objc_setAssociatedObject(self, &myKey, anObject, OBJC_ASSOCIATION_RETAIN);
```
**dispose**: 
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
**get**:
```objc
// 获取成员变量名
const char * ivar_getName ( Ivar v );
// 获取成员变量类型编码
const char * ivar_getTypeEncoding ( Ivar v );
// 获取成员变量的偏移量
ptrdiff_t ivar_getOffset ( Ivar v );
```
### **property_**：
**get**:
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
**get**:
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
**copy**:
```objc
// 获取方法的返回值类型的字符串
char * method_copyReturnType ( Method m );
// 获取方法的指定位置参数的类型字符串
char * method_copyArgumentType ( Method m, unsigned int index );
// 通过引用返回方法的返回值类型字符串
void method_getReturnType ( Method m, char *dst, size_t dst_len );
```
**set**：
```objc
// 设置方法的实现
IMP method_setImplementation ( Method m, IMP imp );
```
**exchange**：
```objc
// 交换两个方法的实现
void method_exchangeImplementations ( Method m1, Method m2 );
```
**description**:
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
如果方法返回值是一个对象，那么发送给nil的消息将返回nil；如果方法返回值为指针类型，其指针大小为小于或者等于``sizeof(void*)，float，double，long double 或者long long的整型``标量，发送给nil的消息将返回0；如果方法返回值为结构体,发送给nil的消息将返回0。结构体中各个字段的值将都是0；如果方法的返回值不是上述提到的几种情况，那么发送给nil的消息的返回值将是未定义的。
* 如果target非nil，在target的``Class``中根据``SEL``去找``IMP``。（因为同一个方法可能在不同的类中有不同的实现，所以我们需要依赖于接收者的类来找到的确切的实现）。
* 首先它找到``selector``对应的方法实现:
* 在target类的方法缓存列表里检查有没有对应的方法实现，有的话，直接调用。
* 比较请求的``selector``和类方法列表中的``selector``，对应的话，直接调用。
* 比较请求的``selector``和父类方法列表，父类的父类，直至根类，如果有对应，则直接调用。（方法重写拦截父类方法的原理）
* 调用方法实现，并将接收者对象及方法的所有参数传给它。
* 最后，将实现函数的返回值作为自己的返回值。

## 动态方法解析与消息转发
如果以上的类中没有找到对应的``selector``（一般保险起见先用``respondsToSelector:``内省判断），还可以利用消息转发机制依次执行以下流程：
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

