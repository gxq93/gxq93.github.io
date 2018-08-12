---
title: Objective-C内部实现浅谈-类目
date: 2015-09-04 15:32:45
tags: [Objective-C,Runtime]
categories: 技术
---
本文主要介绍了 runtime 中类目内部相关的实现。

<!--more-->

# 类目
和 Category 语法很相似的还有 Extension，二者的区别在于，Extension 在编译期就直接和原类编译在一起，而 Category 是在运行时动态添加到原类中的。

``` objc
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

## 原理
在``_read_images``函数中会执行一个循环嵌套，外部循环遍历所有类，并取出当前类对应 Category 数组。内部循环会遍历取出的 Category 数组，将每个``category_t``对象取出，最终执行``addUnattachedCategoryForClass``函数添加到 Category 哈希表中。

``` c
// 将category_t添加到list中，并通过NXMapInsert函数，更新所属类的Category列表
static void addUnattachedCategoryForClass(category_t *cat, Class cls, 
                                          header_info *catHeader)
{
    // 获取到未添加的Category哈希表
    NXMapTable *cats = unattachedCategories();
    category_list *list;

    // 获取到buckets中的value，并向value对应的数组中添加category_t
    list = (category_list *)NXMapGet(cats, cls);
    if (!list) {
        list = (category_list *)
            calloc(sizeof(*list) + sizeof(list->list[0]), 1);
    } else {
        list = (category_list *)
            realloc(list, sizeof(*list) + sizeof(list->list[0]) * (list->count + 1));
    }
    // 替换之前的list字段
    list->list[list->count++] = (locstamped_category_t){cat, catHeader};
    NXMapInsert(cats, cls, list);
}
```

Category 维护了一个名为``category_map``的哈希表，哈希表存储所有 category_t 对象。

``` c
// 获取未添加到Class中的category哈希表
static NXMapTable *unattachedCategories(void)
{
    // 未添加到Class中的category哈希表
    static NXMapTable *category_map = nil;

    if (category_map) return category_map;

    // fixme initial map size
    category_map = NXCreateMapTable(NXPtrValueMapPrototype, 16);

    return category_map;
}
```

上面只是完成了向 Category 哈希表中添加的操作，这时候哈希表中存储了所有 category_t 对象。然后需要调用``remethodizeClass``函数，向对应的 Class 中添加 Category 的信息。

在``remethodizeClass``函数中会查找传入的 Class 参数对应的 Category 数组，然后将数组传给``attachCategories``函数，执行具体的添加操作。

``` c
// 将Category的信息添加到Class，包含method、property、protocol
static void remethodizeClass(Class cls)
{
    category_list *cats;
    bool isMeta;
    isMeta = cls->isMetaClass();

    // 从Category哈希表中查找category_t对象，并将已找到的对象从哈希表中删除
    if ((cats = unattachedCategoriesForClass(cls, false/*not realizing*/))) {
        attachCategories(cls, cats, true /*flush caches*/);        
        free(cats);
    }
}
```

在``attachCategories``函数中，查找到 Category 的方法列表、属性列表、协议列表，然后通过对应的``attachLists``函数，添加到 Class 对应的``class_rw_t``结构体中。

``` c
// 获取到Category的Protocol list、Property list、Method list，然后通过attachLists函数添加到所属的类中
static void attachCategories(Class cls, category_list *cats, bool flush_caches)
{
    if (!cats) return;
    if (PrintReplacedMethods) printReplacements(cls, cats);

    bool isMeta = cls->isMetaClass();

    // 按照Category个数，分配对应的内存空间
    method_list_t **mlists = (method_list_t **)
        malloc(cats->count * sizeof(*mlists));
    property_list_t **proplists = (property_list_t **)
        malloc(cats->count * sizeof(*proplists));
    protocol_list_t **protolists = (protocol_list_t **)
        malloc(cats->count * sizeof(*protolists));

    int mcount = 0;
    int propcount = 0;
    int protocount = 0;
    int i = cats->count;
    bool fromBundle = NO;
    
    // 循环查找出Protocol list、Property list、Method list
    while (i--) {
        auto& entry = cats->list[i];

        method_list_t *mlist = entry.cat->methodsForMeta(isMeta);
        if (mlist) {
            mlists[mcount++] = mlist;
            fromBundle |= entry.hi->isBundle();
        }

        property_list_t *proplist = 
            entry.cat->propertiesForMeta(isMeta, entry.hi);
        if (proplist) {
            proplists[propcount++] = proplist;
        }

        protocol_list_t *protolist = entry.cat->protocols;
        if (protolist) {
            protolists[protocount++] = protolist;
        }
    }

    auto rw = cls->data();

    // 执行添加操作
    prepareMethodLists(cls, mlists, mcount, NO, fromBundle);
    rw->methods.attachLists(mlists, mcount);
    free(mlists);
    if (flush_caches  &&  mcount > 0) flushCaches(cls);

    rw->properties.attachLists(proplists, propcount);
    free(proplists);

    rw->protocols.attachLists(protolists, protocount);
    free(protolists);
}
```

这个过程就是将 Category 中的信息，添加到对应的 Class 中，一个类的 Category 可能不只有一个，在这个过程中会将所有 Category 的信息都合并到 Class 中。

## 方法覆盖

{% asset_img category.jpg %}

在进行方法调用的时候，会优先遍历 Category 的方法，并且后面被添加到项目里的 Category，会被优先调用。上面的例子调用顺序就是Category3 -> Category2 -> Category1 -> TestObject。如果从方法列表中找到方法后，就不会继续向后查找，这就是类方法被 Category ”覆盖”的原因。

## 类目为什么不能添加属性
类目并不是不能添加属性，从上面类目的结构体中可以看到他拥有一个 property_list，分类其实是不能添加 Ivar,因为他本身并不是一个类，他不拥有自己的 isa 指针，因此这些 properties 并不会自动生成 Ivar，也就是不会有 @synthesize 的作用，dyld 加载的期间，这些 categories 会被加载并 patch 到相应的类中。

## Load方法
附加 Category 到类的工作会先于 +load 方法的执行，因此可以在 load 方法中调用分类方法。

+load 的执行顺序是先类，后 Category，而 Category 的 +load 执行顺序是根据编译顺序决定的。
