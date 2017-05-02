---
title: Objective-C内部实现剖析-协议和类目
date: 2015-09-04 15:32:45
tags: [Objective-C,Runtime]
categories: 技术
thumbnail: http://7xtg0o.com1.z0.glb.clouddn.com/1-i3SuhBo3VEC8G79ee1CQ5w.jpeg
---
本文主要介绍了 runtime 中协议和类目内部相关的实现。

<!--more-->

# 协议
## 数据结构
``` objc
typedef struct objc_object Protocol;
```
其实协议就是一个对象结构体
## 操作函数
### **objc_**:
``` objc
/* 返回指定的协议 */
Protocol * objc_getProtocol ( const char *name );
/* 获取运行时所知道的所有协议的数组 */
Protocol ** objc_copyProtocolList ( unsigned int *outCount );
/* 创建新的协议实例 */
Protocol * objc_allocateProtocol ( const char *name );
/* 在运行时中注册新创建的协议 */
void objc_registerProtocol ( Protocol *proto );
```
### **protocol_**:
**get**： 协议，属性
``` objc
/* 返回协议名 */
const char * protocol_getName ( Protocol *p );

/* 获取协议的指定属性 */
objc_property_t protocol_getProperty ( Protocol *proto, const char *name, BOOL isRequiredProperty, BOOL isInstanceProperty );
```
**copy**：协议列表，属性列表
``` objc
/* 获取协议中的属性列表 */
objc_property_t * protocol_copyPropertyList ( Protocol *proto, unsigned int *outCount );
/* 获取协议采用的协议 */
Protocol ** protocol_copyProtocolList ( Protocol *proto, unsigned int *outCount );
```
**add**：属性，方法，协议
``` objc
/* 为协议添加方法 */
void protocol_addMethodDescription ( Protocol *proto, SEL name, const char *types, BOOL isRequiredMethod, BOOL isInstanceMethod );

/* 添加一个已注册的协议到协议中 */
void protocol_addProtocol ( Protocol *proto, Protocol *addition );

/* 为协议添加属性 */
void protocol_addProperty ( Protocol *proto, const char *name, const objc_property_attribute_t *attributes, unsigned int attributeCount, BOOL isRequiredProperty, BOOL isInstanceProperty );
```
**isEqual**：判断两协议等同
``` objc
/* 测试两个协议是否相等 */
BOOL protocol_isEqual ( Protocol *proto, Protocol *other );
```
**comform**：判断是否遵循协议
``` objc
/* 查看协议是否采用了另一个协议 */
BOOL protocol_conformsToProtocol ( Protocol *proto, Protocol *other );
```
# 类目
``` objc
typedef struct objc_category *Category;

struct objc_category {
    char *category_name     OBJC2_UNAVAILABLE; /* 分类名 */
    char *class_name        OBJC2_UNAVAILABLE; /* 分类所属的类名 */
    struct objc_method_list *instance_methods    OBJC2_UNAVAILABLE; /* 实例方法列表 */
    struct objc_method_list *class_methods       OBJC2_UNAVAILABLE; /* 类方法列表 */
    struct objc_protocol_list *protocols         OBJC2_UNAVAILABLE; /* 分类所实现的协议列表 */
}  

/* objc-runtime-new.h中定义：*/

struct category_t {
    const char *name;         /* name是指class_name而不是category_name */
    classref_t cls;           /* cls是要扩展的类对象，编译期间是不会定义的，而是在Runtime阶段通过name对应到对应的类对象 */
    struct method_list_t *instanceMethods;       
    struct method_list_t *classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;  
}
```
category 就是定义方法的结构体，instance_methods 列表是 objc_class 中方法列表的一个子集，class_methods 列表是元类方法列表的一个子集。由其结构成员可知，category 为什么不能添加成员变量(可添加属性，只有 setter/getter 方法)。

给 category 添加方法后，category_list 会生成 method list。这个方法列表是倒序添加的，也就是说，新生成的category 的方法会先于旧的 category 的方法插入。category 的方法会优先于类方法执行。
## 类目为什么不能添加属性
类目并不是不能添加属性，从上面类目的结构体中可以看到他拥有一个 property_list，分类其实是不能添加 Ivar,因为他本身并不是一个类，他不拥有自己的 isa 指针，因此这些 properties 并不会自动生成 Ivar，也就是不会有 @synthesize 的作用，dyld 加载的期间，这些 categories 会被加载并 patch 到相应的类中。
