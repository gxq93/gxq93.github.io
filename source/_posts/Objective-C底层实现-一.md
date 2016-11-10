---
title: 写给自己的Runtime-类和对象
date: 2015-12-10 10:32:00
tags:
cover: http://img.tuku.cn/file_big/201501/93c19becdf65480cbf4c1eb72117d988.jpg
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
类和对象的操作函数名只要以类对象``class_``为前缀，实例对象``object_``为前缀。
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
