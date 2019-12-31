---
layout: post
title: "iOS Runtime分析"
author: "李萌"
categories: learning
tags: [learning]
feature-img: "assets/img/article/runtime.jpg"
thumbnail: "assets/img/article/runtime.jpg"
typora-root-url: ../assets
---

Runtime的特性主要是消息(`方法`)传递，如果消息(`方法`)在对象中找不到，就进行转发，具体怎么实现的呢。我们从下面几个方面探寻Runtime的实现机制。

### Runtime介绍

> Objective-C 扩展了 C 语言，并加入了面向对象特性和 Smalltalk 式的消息传递机制。而这个扩展的核心是一个用 C 和 编译语言 写的 Runtime 库。它是 Objective-C 面向对象和动态机制的基石。

> Objective-C 是一个动态语言，这意味着它不仅需要一个编译器，也需要一个运行时系统来动态得创建类和对象、进行消息传递和转发。理解 Objective-C 的 Runtime 机制可以帮我们更好的了解这个语言，适当的时候还能对语言进行扩展，从系统层面解决项目中的一些设计或技术问题。了解 Runtime ，要先了解它的核心 - 消息传递 （Messaging）。

`Runtime` 基本是用 `C` 和`汇编`写的，可见苹果为了动态系统的高效而作出的努力。你可以在[这里](http://www.opensource.apple.com/source/objc4/)下到苹果维护的开源代码。苹果和GNU各自维护一个开源的 [runtime](https://github.com/RetVal/objc-runtime) 版本，这两个版本之间都在努力的保持一致。

平时的业务中主要是使用[官方Api](https://developer.apple.com/reference/objectivec/objective_c_runtime#//apple_ref/doc/uid/TP40001418-CH1g-126286)，解决我们框架性的需求。

高级编程语言想要成为可执行文件需要先编译为汇编语言再汇编为机器语言，机器语言也是计算机能够识别的唯一语言，但是`OC`并不能直接编译为汇编语言，而是要先转写为纯`C`语言再进行编译和汇编的操作，从`OC`到`C`语言的过渡就是由runtime来实现的。然而我们使用`OC`进行面向对象开发，而`C`语言更多的是面向过程开发，这就需要将面向对象的类转变为面向过程的结构体。

### Runtime消息传递

一个对象的方法像这样`[obj message]`，编译器转成消息发送`objc_msgSend(obj, message)`，`Runtime`时执行流程是这样的：

- 首先，通过`obj`的`isa`指针找到它的`classs`；
- 在`class`的`method list`找`messge`；
- 如果`class`中没有找到`messge`，继续往它的`superclass`中找；
- 一旦找到`message`这个函数，就去执行它的实现`IMP`。

但这种实现存在问题，效率低。因为一个`class`通常只有`20%`的函数会经常被调用，可能占总调用次数的`80%`。每个消息都需要遍历一次`objc_method_list`并不合理。而`objc_class`中另外一个重要成员`objc_cache`就会消除上述问题，在找到`message`之后，会把`message`的`method_name`作为`key`，`method_imp`作为`value`存储起来。当再次受到`message`消息的时候，就可以直接在`objc_cache`里找到，避免去遍历`objc_method_list`。

`objc_msgSend`的方法定义如下：

```
OBJC_EXPORT void objc_msgSend(void /* id self, SEL op, ... */ )
```

那么消息传递是怎么实现的呢？我们看看对象(`object`)，类(`class`)，方法(`method`)这几个结构体：

```
// 对象
struct objc_object {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;
};

// 类
struct objc_class {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;
#if !__OBJC2__
    Class _Nullable super_class                              OBJC2_UNAVAILABLE;
    const char * _Nonnull name                               OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list * _Nullable ivars                  OBJC2_UNAVAILABLE;
    struct objc_method_list * _Nullable * _Nullable methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache * _Nonnull cache                       OBJC2_UNAVAILABLE;
    struct objc_protocol_list * _Nullable protocols          OBJC2_UNAVAILABLE;
#endif
} OBJC2_UNAVAILABLE;

// 方法列表
struct objc_method_list {
    struct objc_method_list * _Nullable obsolete             OBJC2_UNAVAILABLE;
    int method_count                                         OBJC2_UNAVAILABLE;
#ifdef __LP64__
    int space                                                OBJC2_UNAVAILABLE;
#endif
    /* variable length structure */
    struct objc_method method_list[1]                        OBJC2_UNAVAILABLE;
}                                                            OBJC2_UNAVAILABLE

// 方法
struct objc_method {
    SEL _Nonnull method_name                                 OBJC2_UNAVAILABLE;
    char * _Nullable method_types                            OBJC2_UNAVAILABLE;
    IMP _Nonnull method_imp                                  OBJC2_UNAVAILABLE;
}                                                            OBJC2_UNAVAILABLE;
```

1. 系统首先找到消息的接收对象，然后通过对象`isa`指针找到它的类`class`。

2. 在它的类中查找`method_list`，是否有`selector`方法。

3. 没有则查找父类的`method_list`。

4. 找到对应的`method`，执行它的`IMP`。

5. 转发`IMP`的`return`值。

#### 类对象(objc_class)

`Objective-C`类是由`Class`类型来表示的，它实际上是一个指向`objc_class`结构体的指针。

```
typedef struct objc_class *Class;
```

查看`objc/runtime.h`中`objc_class`结构体定义如下：

```
struct objc_class {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;
    
#if !__OBJC2__
    Class _Nullable super_class                              OBJC2_UNAVAILABLE;
    const char * _Nonnull name                               OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list * _Nullable ivars                  OBJC2_UNAVAILABLE;
    struct objc_method_list * _Nullable * _Nullable methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache * _Nonnull cache                       OBJC2_UNAVAILABLE;
    struct objc_protocol_list * _Nullable protocols          OBJC2_UNAVAILABLE;
#endif

} OBJC2_UNAVAILABLE;
```

`struct objc_class`结构体里保存了指向父类的指针、类的名称、版本、实例大小、实例变量列表，方法列表、缓存、遵循协议列表等。
类对象就是一个结构体`struct objc_class`，这个结构体存放的数据成为元数据(`metadata`)。
该结构体的第一个成员变量`isa`指针，这就说明类本身其实也是一个对象，因此我们称`objc_class`其为类对象，类对象在编译期产生用于创建实例对象，类对象是单例。

#### 实例对象(objc_object)

```
struct objc_object {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;
};

typedef struct objc_object *id;
```

实例对象中`isa`指针指向类对象`objc_class`。
类对象中的元数据存储的都是如何创建一个实例的相关信息，那么类对象和类方法应该从哪里创建呢？
就是从`isa`指针指向的结构体创建，类对象的`isa`指针

#### 元类(Meta Class)

![runtime-meta](https://raw.githubusercontent.com/limeng99/limeng99.github.io/master/assets/img/screenshots/runtime-meta.png)

通过上图我们可以看出整个体系构成了一个自闭环，`struct objc_object`结构体`实例`它的`isa`指针指向类对象；
类对象的`isa`指针指向了元类，`super_class`指针指向父类的类对象；
而元类的`super_class`指针指向了父类的元类，那元类的`isa`指针又指向了`NSObject元类`；
`NSObject`的元类`meta-class`的`super_class`指针指向了`NSObject类对象`，`isa`指针指向了自己。

元类(Meta Class)是一个类对象的类。
所有的类自身也是一个对象，我们可以向这个对象发送消息(即调用类方法)。
为了调用类方法，这个类的`isa`指针必须指向一个包含这些类方法的一个`objc_class`结构体。这就引出了`meta-class`的概念，元类中保存了创建类对象及类方法所需的所有信息。
任何`NSObject`继承体系下的`meta-class`都使用`NSObject`的`meta-class`作为自己的所属类，而基类的`meta-class`的`isa`指针指向它自己。

