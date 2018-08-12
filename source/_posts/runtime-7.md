---
title: Objective-C内部实现浅谈-Load方法
date: 2015-12-28 15:46:58
tags: [Objective-C,Runtime]
categories: 技术
---
本文主要介绍了 Load方法 相关的内部实现。

<!--more-->

``+ load``作为 Objective-C 中的一个方法，与其它方法有很大的不同。它只是一个在整个文件被加载到运行时，在 main 函数调用之前被 ObjC 运行时调用的钩子方法。其中关键字有这么几个：

* 文件刚加载
* main 函数之前
* 钩子方法

当一个类的 load 方法调用时，调用栈大概是这样的：
``` objc
0  +[XXObject load]
1  call_class_loads()
2  call_load_methods
3  load_images
4  dyld::notifySingle(dyld_image_states, ImageLoader const*)
11 _dyld_start
```

## 准备 + load 方法
在加载镜像时，会执行
```c 
for (uint32_t i = 0; i < infoCount; i++) {
    if (hasLoadMethods((const headerType *)infoList[i].imageLoadAddress)) {
        found = true;
        break;
    }
}
```
就会进入 load_images_nolock 来查找 load 方法：
``` c
bool load_images_nolock(enum dyld_image_states state,uint32_t infoCount,
                   const struct dyld_image_info infoList[])
{
    bool found = NO;
    uint32_t i;
    i = infoCount;
    while (i--) {
        const headerType *mhdr = (headerType*)infoList[i].imageLoadAddress;
        if (!hasLoadMethods(mhdr)) continue;
        prepare_load_methods(mhdr);
        found = YES;
    }
    return found;
}
```
调用 prepare_load_methods 对 load 方法的调用进行准备（将需要调用 load 方法的类添加到一个列表中，后面的小节中会介绍）：
``` c
void prepare_load_methods(const headerType *mhdr)
{
    size_t count, i;
    runtimeLock.assertWriting();
    classref_t *classlist = 
        _getObjc2NonlazyClassList(mhdr, &count);
    for (i = 0; i < count; i++) {
        schedule_class_load(remapClass(classlist[i]));
    }
    category_t **categorylist = _getObjc2NonlazyCategoryList(mhdr, &count);
    for (i = 0; i < count; i++) {
        category_t *cat = categorylist[i];
        Class cls = remapClass(cat->cls);
        if (!cls) continue;  // category for ignored weak-linked class
        realizeClass(cls);
        assert(cls->ISA()->isRealized());
        add_category_to_loadable_list(cat);
    }
}
```
通过 _getObjc2NonlazyClassList 获取所有的类的列表之后，会通过 remapClass 获取类对应的指针，然后调用 schedule_class_load 递归地安排当前类和没有调用 + load 父类进入列表。
``` c
static void schedule_class_load(Class cls)
{
    if (!cls) return;
    assert(cls->isRealized());
    if (cls->data()->flags & RW_LOADED) return;
    schedule_class_load(cls->superclass);
    add_class_to_loadable_list(cls);
    cls->setInfo(RW_LOADED); 
}
```
执行 add_class_to_loadable_list(cls) 将当前类加入加载列表之前，会先把父类加入待加载的列表，保证父类在子类前调用 load 方法。

## 调用 + load 方法
在将镜像加载到运行时、对 load 方法的准备就绪之后，执行 call_load_methods，开始调用 load 方法：
``` c
void call_load_methods(void)
{
    ...
    do {
        while (loadable_classes_used > 0) {
            call_class_loads();
        }
        more_categories = call_category_loads();
    } while (loadable_classes_used > 0  ||  more_categories);
    ...
}
```
其中 call_class_loads 会从一个待加载的类列表 loadable_classes 中寻找对应的类，然后找到 @selector(load) 的实现并执行。
``` c
static void call_class_loads(void)
{
    int i;
    struct loadable_class *classes = loadable_classes;
    int used = loadable_classes_used;
    loadable_classes = nil;
    loadable_classes_allocated = 0;
    loadable_classes_used = 0;
    for (i = 0; i < used; i++) {
        Class cls = classes[i].cls;
        load_method_t load_method = (load_method_t)classes[i].method;
        if (!cls) continue;
        (*load_method)(cls, SEL_load);
    }
    if (classes) free(classes);
}
```
这行 (*load_method)(cls, SEL_load) 代码就会调用 +[XXObject load] 方法。

Q：load 方法是如何被调用的？

A：当 Objective-C 运行时初始化的时候，会通过 dyld_register_image_state_change_handler 在每次有新的镜像加入运行时的时候，进行回调。执行 load_images 将所有包含 load 方法的文件加入列表 loadable_classes ，然后从这个列表中找到对应的 load 方法的实现，调用 load 方法。

## 生成 loadable_class
在调用 load_images -> load_images_nolock -> prepare_load_methods -> schedule_class_load -> add_class_to_loadable_list 的时候会将未加载的类添加到 loadable_classes 数组中：
``` c
void add_class_to_loadable_list(Class cls)
{
    IMP method;
    loadMethodLock.assertLocked();
    method = cls->getLoadMethod();
    if (!method) return;
    if (loadable_classes_used == loadable_classes_allocated) {
        loadable_classes_allocated = loadable_classes_allocated*2 + 16;
        loadable_classes = (struct loadable_class *)
            realloc(loadable_classes,
                              loadable_classes_allocated *
                              sizeof(struct loadable_class));
    }
    loadable_classes[loadable_classes_used].cls = cls;
    loadable_classes[loadable_classes_used].method = method;
    loadable_classes_used++;
}
```
方法刚被调用时：
* 会从 class 中获取 load 方法： method = cls->getLoadMethod();
* 判断当前 loadable_classes 这个数组是否已经被全部占用了：loadable_classes_used == loadable_classes_allocated
* 在当前数组的基础上扩大数组的大小：realloc
* 把传入的 class 以及对应的方法的实现加到列表中

另外一个用于保存分类的列表 loadable_categories 也有一个类似的方法 add_category_to_loadable_list。
``` c
void add_category_to_loadable_list(Category cat)
{
    IMP method;
    loadMethodLock.assertLocked();
    method = _category_getLoadMethod(cat);
    if (!method) return;
    if (loadable_categories_used == loadable_categories_allocated) {
        loadable_categories_allocated = loadable_categories_allocated*2 + 16;
        loadable_categories = (struct loadable_category *)
            realloc(loadable_categories,
                              loadable_categories_allocated *
                              sizeof(struct loadable_category));
    }
    loadable_categories[loadable_categories_used].cat = cat;
    loadable_categories[loadable_categories_used].method = method;
    loadable_categories_used++;
}
```
实现几乎与 add_class_to_loadable_list 完全相同。

## 调用 loadable_class
调用 load 方法的过程：load_images -> call_load_methods -> call_class_loads 会从 loadable_classes 中取出对应类和方法，执行 load。
``` c
void call_load_methods(void)
{
    static bool loading = NO;
    bool more_categories;
    loadMethodLock.assertLocked();
    if (loading) return;
    loading = YES;
    void *pool = objc_autoreleasePoolPush();
    do {
        while (loadable_classes_used > 0) {
            call_class_loads();
        }
        more_categories = call_category_loads();
    } while (loadable_classes_used > 0  ||  more_categories);
    objc_autoreleasePoolPop(pool);
    loading = NO;
}
```
上述方法对所有在 loadable_classes 以及 loadable_categories 中的类以及分类执行 load 方法。
``` c
do {
    while (loadable_classes_used > 0) {
        call_class_loads();
    }
    more_categories = call_category_loads();
} while (loadable_classes_used > 0  ||  more_categories);
```
调用顺序如下：
* 不停调用类的 + load 方法，直到 loadable_classes 为空
* 调用一次 call_category_loads 加载分类
* 如果有 loadable_classes 或者更多的分类，继续调用 load 方法

相比于类 load 方法的调用，分类中 load 方法的调用就有些复杂了：
``` c
static bool call_category_loads(void)
{
    int i, shift;
    bool new_categories_added = NO;
    // 1. 获取当前可以加载的分类列表
    struct loadable_category *cats = loadable_categories;
    int used = loadable_categories_used;
    int allocated = loadable_categories_allocated;
    loadable_categories = nil;
    loadable_categories_allocated = 0;
    loadable_categories_used = 0;
    for (i = 0; i < used; i++) {
        Category cat = cats[i].cat;
        load_method_t load_method = (load_method_t)cats[i].method;
        Class cls;
        if (!cat) continue;
        cls = _category_getClass(cat);
        if (cls  &&  cls->isLoadable()) {
            // 2. 如果当前类是可加载的 `cls  &&  cls->isLoadable()` 就会调用分类的 load 方法
            (*load_method)(cls, SEL_load);
            cats[i].cat = nil;
        }
    }
    // 3. 将所有加载过的分类移除 `loadable_categories` 列表
    shift = 0;
    for (i = 0; i < used; i++) {
        if (cats[i].cat) {
            cats[i-shift] = cats[i];
        } else {
            shift++;
        }
    }
    used -= shift;
    // 4. 为 `loadable_categories` 重新分配内存，并重新设置它的值
    new_categories_added = (loadable_categories_used > 0);
    for (i = 0; i < loadable_categories_used; i++) {
        if (used == allocated) {
            allocated = allocated*2 + 16;
            cats = (struct loadable_category *)
                realloc(cats, allocated *
                                  sizeof(struct loadable_category));
        }
        cats[used++] = loadable_categories[i];
    }
    if (loadable_categories) free(loadable_categories);
    if (used) {
        loadable_categories = cats;
        loadable_categories_used = used;
        loadable_categories_allocated = allocated;
    } else {
        if (cats) free(cats);
        loadable_categories = nil;
        loadable_categories_used = 0;
        loadable_categories_allocated = 0;
    }
    return new_categories_added;
}
```
这个方法有些长，我们来分步解释方法的作用：

* 获取当前可以加载的分类列表
* 如果当前类是可加载的 cls && cls->isLoadable() 就会调用分类的 load 方法
* 将所有加载过的分类移除 loadable_categories 列表
* 为 loadable_categories 重新分配内存，并重新设置它的值

## 调用的顺序

你过去可能会听说过，对于 load 方法的调用顺序有两条规则：

* 父类先于子类调用
* 类先于分类调用

第一条规则是由于 schedule_class_load 有如下的实现：
``` c
static void schedule_class_load(Class cls)
{
    if (!cls) return;
    assert(cls->isRealized());
    if (cls->data()->flags & RW_LOADED) return;
    schedule_class_load(cls->superclass);
    add_class_to_loadable_list(cls);
    cls->setInfo(RW_LOADED); 
}
```
这里通过这行代码 schedule_class_load(cls->superclass) 总是能够保证没有调用 load 方法的父类先于子类加入 loadable_classes 数组，从而确保其调用顺序的正确性。

类与分类中 load 方法的调用顺序主要在 call_load_methods 中实现：
``` c
do {
    while (loadable_classes_used > 0) {
        call_class_loads();
    }
    more_categories = call_category_loads();
} while (loadable_classes_used > 0  ||  more_categories);
```
上面的 do while 语句能够在一定程度上确保，类的 load 方法会先于分类调用。但是这里不能完全保证调用顺序的正确。

如果分类的镜像在类的镜像之前加载到运行时，上面的代码就没法保证顺序的正确了，所以，我们还需要在 call_category_loads 中判断类是否已经加载到内存中（调用 load 方法）：
``` c
if (cls  &&  cls->isLoadable()) {
    (*load_method)(cls, SEL_load);
    cats[i].cat = nil;
}
```
这里，检查了类是否存在并且是否可以加载，如果都为真，那么就可以调用分类的 load 方法了。
