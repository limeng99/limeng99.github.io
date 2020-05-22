---
layout: post
title: "iOS Runtime分析(二)"
author: "李萌"
categories: learning
tags: [learning]
feature-img: "assets/img/article/runtime.jpg"
thumbnail: "assets/img/article/runtime.jpg"
typora-root-url: ../assets
---

[Runtime源码下载](https://opensource.apple.com/tarballs/objc4)

### 1、一个NSObject占用多少内存？

受限于内存分配的机制，一个 `NSObject`对象都会分配 `16byte` 的内存空间。但是实际上在 32位系统上，只使用了 `8byte`;

一个 NSObject 实例对象成员变量所占的大小，实际上是 8 字节。

```
size_t obj_size = class_getInstanceSize([NSObject class]);
NSLog(@"class_getInstanceSize ---- %zu", obj_size);
打印日志：class_getInstanceSize ---- 8
```

我们可以去`runtime`的源码里面，相关方法具体是怎么实现的。

```
// class_getInstanceSize源码实现
size_t class_getInstanceSize(Class cls) {
    if (!cls) return 0;
    return cls->alignedInstanceSize();
}

// Class's ivar size rounded up to a pointer-size boundary.
// 返回的是Class's ivar size,类的成员变量的大小，NSObject对象只有一个isa成员变量，
// 62位系统下返回的是8个字节
uint32_t alignedInstanceSize() {
	return word_align(unalignedInstanceSize());
}

// alloc的时候分配了多大的内存大小，allocWithZone然后找到_objc_rootAllocWithZone
// 在这个方法中返回的是class_createInstance(cls, 0)，然后跳转进去，返回值再点击去
// 可以看到instanceSize，可以看到，CF要求至少得返回16个字节的内存大小。
size_t instanceSize(size_t extraBytes) {
	size_t size = alignedInstanceSize() + extraBytes;
	// CF requires all objects be at least 16 bytes.
	if (size < 16) size = 16;	
	return size;
}
```

但获取 Obj-C 指针所指向的内存的大小，实际上是16 字节

```
NSObject *obj = [NSObject new];
size_t m_size = malloc_size((__bridge const void *)obj);
NSLog(@"malloc_size ---- %zu", m_size);
打印：malloc_size ---- 16
```
最终而言：

> 一个NSObject占用多少内存？
>  1、系统分配了16个字节给NSObject对象（可以通过malloc_size函数得到）
>  2、但NSObject对象内部只使用了8个字节空间（在64bit环境下，可以通过class_getInstanceSize函数获得）

### 2、isa源码分析

在Runtime源码查看isa_t是共用体。简化结构如下：

```
union isa_t 
{
    Class cls;
    uintptr_t bits;
    # if __arm64__ // arm64架构
#   define ISA_MASK        0x0000000ffffffff8ULL //用来取出33位内存地址使用（&）操作
#   define ISA_MAGIC_MASK  0x000003f000000001ULL
#   define ISA_MAGIC_VALUE 0x000001a000000001ULL
    struct {
        uintptr_t nonpointer        : 1; //0：代表普通指针，1：表示优化过的，可以存储更多信息。
        uintptr_t has_assoc         : 1; //是否设置过关联对象。如果没有，释放时会更快
        uintptr_t has_cxx_dtor      : 1; //是否有C++的析构函数。如果没有，释放时会更快
        uintptr_t shiftcls          : 33; // 存储着Class 或者 Meta-Class对象的内存地址信息
        uintptr_t magic             : 6; //用于在调试时分辨对象是否未完成初始化
        uintptr_t weakly_referenced : 1; //是否有被弱引用指向过。如果没有，释放时会更快
        uintptr_t deallocating      : 1; //是否正在释放
        uintptr_t has_sidetable_rc  : 1; //引用计数器是否过大无法存储在ISA中。如果为1，那么引用计数会存储在一个叫做SideTable的类的属性中
        uintptr_t extra_rc          : 19; //里面存储的值是引用计数器减1

#       define RC_ONE   (1ULL<<45)
#       define RC_HALF  (1ULL<<18)
    };

# elif __x86_64__ // arm86架构,模拟器是arm86
#   define ISA_MASK        0x00007ffffffffff8ULL
#   define ISA_MAGIC_MASK  0x001f800000000001ULL
#   define ISA_MAGIC_VALUE 0x001d800000000001ULL
    struct {
        uintptr_t nonpointer        : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t shiftcls          : 44; // MACH_VM_MAX_ADDRESS 0x7fffffe00000
        uintptr_t magic             : 6;
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 8;
#       define RC_ONE   (1ULL<<56)
#       define RC_HALF  (1ULL<<7)
    };

# else
#   error unknown architecture for packed isa
# endif

}
```

### 3、cache_t源码分析

`cache_t`增量扩展的哈希表结构。哈希表内部存储的 `bucket_t`。

`bucket_t` 中存储的是 `SEL` 和 `IMP`的键值对。

- 如果是有序方法列表，采用二分查找

- 如果是无序方法列表，直接遍历查找

```
// 缓存曾经调用过的方法，提高查找速率
struct cache_t {
    struct bucket_t *_buckets; // 散列表
    mask_t _mask; //散列表的长度 - 1
    mask_t _occupied; // 已经缓存的方法数量，散列表的长度使大于已经缓存的数量的。
    //...
}

struct bucket_t {
    cache_key_t _key; //SEL作为Key @selector()
    IMP _imp; // 函数的内存地址
    //...
}
```

散列表查找过程，在`objc-cache.mm`文件中

```
// 查询散列表，k
bucket_t * cache_t::find(cache_key_t k, id receiver)
{
    assert(k != 0); // 断言

    bucket_t *b = buckets(); // 获取散列表
    mask_t m = mask(); // 散列表长度 - 1
    mask_t begin = cache_hash(k, m); // & 操作
    mask_t i = begin; // 索引值
    do {
        if (b[i].key() == 0  ||  b[i].key() == k) {
            return &b[i];
        }
    } while ((i = cache_next(i, m)) != begin);
    // i 的值最大等于mask,最小等于0。

    // hack
    Class cls = (Class)((uintptr_t)this - offsetof(objc_class, cache));
    cache_t::bad_cache(receiver, (SEL)k, cls);
}
```

上面是查询散列表函数，其中`cache_hash(k, m)`是静态内联方法，将传入的`key`和`mask`进行`&`操作返回`uint32_t`索引值。`do-while`循环查找过程，当发生冲突`cache_next`方法将索引值减1。

### 4、class_rw_t 与 class_ro_t

`class_rw_t` 、`class_ro_t`中的`rw`、`ro`应该是readwrite、readonly的意思。

`ObjC` 类中的属性、方法还有遵循的协议等信息都保存在 `class_rw_t` 中：

```
// 可读可写
struct class_rw_t {
    // Be warned that Symbolication knows the layout of this structure.
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro; // 指向只读的结构体,存放类初始信息

    /*
     这三个都是二维数组，是可读可写的，包含了类的初始内容、分类的内容。
     methods中，存储 method_list_t ----> method_t
     二维数组，method_list_t --> method_t
     这三个二位数组中的数据有一部分是从class_ro_t中合并过来的。
     */
    method_array_t methods; // 方法列表（类对象存放对象方法，元类对象存放类方法）
    property_array_t properties; // 属性列表
    protocol_array_t protocols; //协议列表

    Class firstSubclass;
    Class nextSiblingClass;
    
    //...
}
```

`class_ro_t`存储了当前类在编译期就已经确定的属性、方法以及遵循的协议。

```
struct class_ro_t {  
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
    uint32_t reserved;

    const uint8_t * ivarLayout;

    const char * name;
    method_list_t * baseMethodList;
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;
};
// baseMethodList，baseProtocols，ivars，baseProperties四个都是一维数组。
```

### 5、weak底层分析

runtime 对注册的类会进行布局，对于 weak 修饰的对象会放入一个 hash 表中。 用 weak 指向的对象内存地址作为key，当此对象的引用计数为0的时候会 dealloc，假如 weak 指向的对象内存地址是a，那么就会以a为键， 在这个 weak表中搜索，找到所有以a为键的 weak 对象，从而设置为 nil。

**1.初始化时：runtime会调用objc_initWeak函数，初始化一个新的weak指针指向对象的地址。**

```
{
    NSObject *obj = [[NSObject alloc] init];
    id __weak obj1 = obj;
}
```

当我们初始化一个weak变量时，runtime会调用 `NSObject.mm` 中的`objc_initWeak`函数。

```
// 编译器的模拟代码
id obj1;
objc_initWeak(&obj1, obj);
...

// obj引用计数变为0，变量作用域结束
objc_destroyWeak(&obj1);
```

通过`objc_initWeak`函数初始化"附有weak修饰符的变量(obj1)"，在变量作用域结束时通过`objc_destoryWeak`函数释放该变量 (obj1)。

**2.添加引用时：objc_initWeak函数会调用objc_storeWeak() 函数， objc_storeWeak()的作用是更新指针指向，创建对应的弱引用表。**

`objc_initWeak`函数将"附有weak修饰符的变量 (obj1)"初始化为0 (nil)后，会将"赋值对象" (obj) 作为参数，调用`objc_storeWeak`函数。

```
obj1 = 0；
obj_storeWeak(&obj1, obj);
```

也就是说：weak 修饰的指针默认值是 nil （在Objective-C中向nil发送消息是安全的）

然后`obj_destroyWeak`函数将0（nil）作为参数，调用`objc_storeWeak`函数。

```
objc_storeWeak(&obj1, 0);
```

前面的源代码与下列源代码相同。

```
// 编译器的模拟代码
id obj1;
obj1 = 0;
objc_storeWeak(&obj1, obj);
...

// obj的引用计数变为0，被置nil 
objc_storeWeak(&obj1, 0);
```

`objc_storeWeak`函数把第二个参数的赋值对象（obj）的内存地址作为键值，将第一个参数__weak修饰的属性变量（obj1）的内存地址注册到 weak 表中。如果第二个参数（obj）为0（nil），那么把变量（obj1）的地址从weak表中删除。

由于一个对象可同时赋值给多个附有__weak修饰符的变量中，所以对于一个键值，可注册多个变量的地址。

可以把`objc_storeWeak(&a, b)`理解为：`objc_storeWeak(value, key)`，并且当key变nil，将value置nil。在b非nil时，a和b指向同一个内存地址，在b变nil时，a变nil。此时向a发送消息不会崩溃：在Objective-C中向nil发送消息是安全的。

**3.释放时,调用clearDeallocating函数**

clearDeallocating函数首先根据对象地址获取所有weak指针地址的数组，然后遍历这个数组把其中的数据设为nil，最后把这个entry从weak表中删除，最后清理对象的记录。

weak引用指向的对象被释放时，又是如何去处理weak指针的呢？当释放对象时，其基本流程如下：

1. 调用`objc_release`
2. 因为对象的引用计数为0，所以执行`dealloc`
3. 在dealloc中，调用了`_objc_rootDealloc`函数
4. 在`_objc_rootDealloc`中，调用了`object_dispose`函数
5. 调用`objc_destructInstance`
6. 最后调用`objc_clear_deallocating`

对象被释放时调用的`objc_clear_deallocating`函数:

1. 从weak表中获取废弃对象的地址为键值的记录
2. 将包含在记录中的所有附有 weak修饰符变量的地址，赋值为nil
3. 将weak表中该记录删除
4. 从引用计数表中删除废弃对象的地址为键值的记录

**总结:**

其实weak表是一个hash（哈希）表，Key是weak所指对象的地址，Value是weak指针的地址（这个地址的值是所指对象指针的地址）数组。

