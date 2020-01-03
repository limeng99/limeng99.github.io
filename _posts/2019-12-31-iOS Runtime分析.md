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

#### Method(Objc_method)

```
typedef struct objc_method *Method;

struct objc_method {
    SEL _Nonnull method_name                                 OBJC2_UNAVAILABLE;
    char * _Nullable method_types                            OBJC2_UNAVAILABLE;
    IMP _Nonnull method_imp                                  OBJC2_UNAVAILABLE;
}                                                            
```

`Method`和平时理解的函数是一致的，就是表示能够独立完成一个功能的一段代码。
`objc_method`结构体内容：
- `SEL _Nonnull method_name` 方法名
- `char * _Nullable method_types` 方法类型
- `IMP _Nonnull method_imp` 方法实现

从此看出，`SEL`和`IMP`均是`Method`的属性。

#### SEL(objc_selector)

```
typedef struct objc_selector *SEL;
```

`objc_msgSend`函数第二个参数类型是`SEL`，它是`selector`在`Objective-C`中的表示类型（`Swift`中是`Selector`类）。`selector`是方法选择器，可以理解区分方法的`ID`，而这个`ID`的数据结构是`SEL`。

```
@property SEL selector;
```

可以看出来`selector`是`SEL`的一个实例。
其实`selector`就是个映射到方法的`C`字符串，你可以用`Objective-C`编译器命令`@selector()`或者`Runtime`系统的`sel_registerName`函数来获取一个`SEL`类型的方法选择器。

`selector`既然是一个`string`，应该是类似`className+method`的组合，命名规则有两条：

- 同一个类，`selector`不能重复
- 不同的类，`selector`可以重复

这样会有一个弊端，我们在写`C`代码的时候，经常会用到函数重载，就是函数名相同，参数不同，但是这在`Objective-C`中是行不通的，因为`selector`只记了`className+method`，没有参数，所以没法取费不同的`method`。

比如：

```
- (void)caculate(NSInteger)num;
- (void)caculate(CGFloat)num;
```

这样是会报错的。

我们只能通过命名来区分：

```
- (void)caculateWithInt(NSInteger)num;
- (void)caculateWithFloat(CGFloat)num;
```

在不同类中相同名字的方法所对应的选择器是相同的，即使方法名字相同而变量类型不同也会导致他们具有相同的方法选择器。

#### IMP

```
typedef id _Nullable (*IMP)(id _Nonnull, SEL _Nonnull, ...); 
```

就是指向最终实现程序的内存地址的指针。

在`iOS`的`Runtime`中，`Method`通过`selector`和`IMP`两个属性，实现了快速查询方法及实现，相对提高了性能，又保持了灵活性。

#### 类缓存(objc_cache)

当`Objective-C`运行时通过跟踪它的`isa`指针检查对象时，它可以找到一个实现许多方法的对象。然而，你可能只调用它们的一小部分，并且每次查找时，搜索所有选择器的类分派表没有意义。所以类实现一个缓存，每当你搜索一个类分派表，并找到相应的选择器，它把它放入它的缓存。所以当`objc_msgSend`查找一个类的选择器，它首先搜索类缓存。这是基于这样的理论：如果你在类上调用一个消息，你可能以后再次调用该消息。

为了加速消息分发， 系统会对方法和对应的地址进行缓存，就放在上述的`objc_cache`，所以在实际运行中，大部分常用的方法都是会被缓存起来的，`Runtime`系统实际上非常快，接近直接执行内存地址的程序速度。

#### Category(objc_category)

`Category`是表示一个指向分类的结构体的指针，其定义如下：

```
struct category_t { 
    const char *name; 
    classref_t cls; 
    struct method_list_t *instanceMethods; 
    struct method_list_t *classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;
};

name：是指 class_name 而不是 category_name。
cls：要扩展的类对象，编译期间是不会定义的，而是在Runtime阶段通过name对 应到对应的类对象。
instanceMethods：category中所有给类添加的实例方法的列表。
classMethods：category中所有添加的类方法的列表。
protocols：category实现的所有协议的列表。
instanceProperties：表示Category里所有的properties，这就是我们可以通过objc_setAssociatedObject和objc_getAssociatedObject增加实例变量的原因，不过这个和一般的实例变量是不一样的。
```

从上面的`objc_category`的结构体中可以看出，分类中可以添加实例方法，类方法，甚至可以实现协议，添加属性，不可以添加成员变量。

### Runtime消息转发

上文中介绍了进行一次消息发送会在相关类对象中搜索方法列表，如果找不到则会沿着继承树向上一直搜索直到继承树的根部(通常为`NSObject`)，如果还是找不到会怎么样呢？接下来会逐一介绍消息转发的流程，先看下图：

![runtime-forward](https://raw.githubusercontent.com/limeng99/limeng99.github.io/master/assets/img/screenshots/runtime-forward.png)

消息转发三个步骤：动态方法解析；备用接收者；完整消息转发。

#### 动态方法解析

首先，`Objective-C`运行时会调用`+resolveInstanceMethod:`或者`+resolveClassMethod:`，让你有机会提供一个函数实现。如果你添加了函数并返回`Yes`，那运行时系统就会重新启动一次消息发送的过程。

```
- (void)viewDidLoad {
    [super viewDidLoad];

    // 执行msg函数
    [self performSelector:@selector(msg:)];
}

+ (BOOL)resolveInstanceMethod:(SEL)sel {
    // 检测到如果是执行msg函数，就动态解析，指定新的IMP
    if (sel == @selector(msg:)) {
        class_addMethod([self class], sel, (IMP)msgMethod, "v@:");
        return YES;
    }
    return [super resolveInstanceMethod:sel];
}

// 新的msg函数
void msgMethod(id obj, SEL _cmd) {
    NSLog(@"Receive message");
}
```

打印日志：

```
2020-01-02 10:01:00.294697+0800 Runtime-Demo[44489:1441571] Receive message
```

可以看到虽然没有实现`msg:`函数，但是我们通过`class_addMethod`动态添加`msgMethod`函数，并执行`msgMethod`这个函数的`IMP`。从打印日志看，成功实现了消息转发。

如果`+resolveInstanceMethod:`方法`NO`，运行时就会移到下一步`-forwardingTargetForSelector:`。

#### 备用接收者

如果目标对象实现了`-forwardingTargetForSelector:`，`Runtime`这时就会调用这个方法，给你把这个消息转发给其他对象的机会。

```
#import "ViewController.h"
#import <objc/runtime.h>

@interface Receiver: NSObject

@end

@implementation Receiver

// Receiver的msg函数
- (void)msg {
    NSLog(@"Receive message");
}

@end

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];

    // 执行msg函数
    [self performSelector:@selector(msg)];
}

+ (BOOL)resolveInstanceMethod:(SEL)sel {
    return NO; //返回NO，进入下一步转发
}

- (id)forwardingTargetForSelector:(SEL)aSelector {
    if (aSelector == @selector(msg)) {
        return [Receiver new]; //返回Receiver对象，让Receiver对象接收这个消息
    }
    return [super forwardingTargetForSelector:aSelector];
}

@end
```

打印日志：

```
2020-01-02 10:16:20.360133+0800 Runtime-Demo[44635:1453911] Receive message
```

可以看到我们通过`-forwardingTargetForSelector:`把当前`ViewController`的方法转发给了`Receiver`去执行了。打印结果也证明我们成功实现了转发。

#### 完整消息转发

如果以上两步还不能处理未知消息，则唯一能做的就是启动完整的消息转发机制了。首先它会发送`methodSignatureForSelector:`消息获得函数的参数和返回值类型。如果`methodSignatureForSelector:`返回`nil`，运行时系统则会发出`doesNotRecognizeSelector:`消息，程序也就挂掉了。如果返回了一个函数签名，运行时就会创建一个`NSInvocation`对象并发送`-forwardInvocation:`消息给目标对象。

```
#import "ViewController.h"
#import <objc/runtime.h>

@interface Receiver: NSObject

@end

@implementation Receiver

// Receiver的msg函数
- (void)msg {
    NSLog(@"Receive message");
}

@end

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];

    // 执行msg函数
    [self performSelector:@selector(msg)];
}

+ (BOOL)resolveInstanceMethod:(SEL)sel {
    return NO; //返回NO，进入下一步转发
}

- (id)forwardingTargetForSelector:(SEL)aSelector {
    return nil;//返回nil，进入下一步转发
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    if ([NSStringFromSelector(aSelector) isEqualToString:@"msg"]) {
    		//签名，进入forwardInvocation
        return [NSMethodSignature signatureWithObjCTypes:"v@:"];
    }
    
    return [super methodSignatureForSelector:aSelector];
}

- (void)forwardInvocation:(NSInvocation *)anInvocation {
    SEL sel = anInvocation.selector;
    
    Receiver *receiver = [Receiver new];
    if([receiver respondsToSelector:sel]) {
        [anInvocation invokeWithTarget:receiver];
    }
    else {
        [self doesNotRecognizeSelector:sel];
    }
}

@end
```

打印日志：

```
2020-01-02 11:06:11.223700+0800 Runtime-Demo[46932:1491762] Receive message
```

从打印日志看，我们实现了完整的消息转发。通过签名，`Runtime`生成了一个对象`anInvocation`，发送给了`forwardInvocation`，我们在`forwardInvocation`方法里面让`Receiver`对象去执行了`msg`函数。签名参数`v@:`怎么解释呢，这里苹果文档[Type Encodings](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100-SW1)有详细的解释。

### Runtime应用

`Runtime`简直就是做大型框架的利器。它的应用场景非常多。

#### 关联对象(Objective-C Associated Objects)给分类增加属性

分类是不能定义属性和变量的，下面通过关联对象实现给分类添加属性。

```
//关联对象
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)
//获取关联的对象
id objc_getAssociatedObject(id object, const void *key)
//移除关联的对象
void objc_removeAssociatedObjects(id object)

id object：被关联的对象
const void *key：关联的key，要求唯一
id value：关联的对象
objc_AssociationPolicy policy：内存管理的策略


// 内存管理的策略
typedef OBJC_ENUM(uintptr_t, objc_AssociationPolicy) {
    OBJC_ASSOCIATION_ASSIGN = 0,           /**< Specifies a weak reference to the associated object. */
    OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1, /**< Specifies a strong reference to the associated object. 
                                            *   The association is not made atomically. */
    OBJC_ASSOCIATION_COPY_NONATOMIC = 3,   /**< Specifies that the associated object is copied. 
                                            *   The association is not made atomically. */
    OBJC_ASSOCIATION_RETAIN = 01401,       /**< Specifies a strong reference to the associated object.
                                            *   The association is made atomically. */
    OBJC_ASSOCIATION_COPY = 01403          /**< Specifies that the associated object is copied.
                                            *   The association is made atomically. */
};
```


| 内存策略                          | 属性修饰                                            | 描述                                                         |
| --------------------------------- | --------------------------------------------------- | ------------------------------------------------------------ |
| OBJC_ASSOCIATION_ASSIGN           | @property (assign) 或 @property (unsafe_unretained) | 指定一个关联对象的弱引用。                                   |
| OBJC_ASSOCIATION_RETAIN_NONATOMIC | @property (nonatomic, strong)                       | @property (nonatomic, strong)   指定一个关联对象的强引用，不能被原子化使用。 |
| OBJC_ASSOCIATION_COPY_NONATOMIC   | @property (nonatomic, copy)                         | 指定一个关联对象的copy引用，不能被原子化使用。               |
| OBJC_ASSOCIATION_RETAIN           | @property (atomic, strong)                          | 指定一个关联对象的强引用，能被原子化使用。                   |
| OBJC_ASSOCIATION_COPY             | @property (atomic, copy)                            | 指定一个关联对象的copy引用，能被原子化使用。                 |

实现`UIImage`分类添加自定义属性`name`。

```
// UIImage+Runtime.h
@interface UIImage (Runtime)

@property (nonatomic, copy) NSString *name;

@end

// UIImage+Runtime.m
#import "UIImage+Runtime.h"
#import <objc/runtime.h>

@implementation UIImage (Runtime)

- (void)setName:(NSString *)name {
    objc_setAssociatedObject(self, @selector(name), name, OBJC_ASSOCIATION_COPY_NONATOMIC);
}

- (NSString *)name {
    return objc_getAssociatedObject(self, _cmd);
}

@end

// ViewController.m
- (void)viewDidLoad {
    [super viewDidLoad];

    UIImage *image = [UIImage new];
    image.name = @"image_pic";
    NSLog(@"name: %@", image.name);
}
```

打印日志：

```
2020-01-02 11:27:39.232196+0800 Runtime-Demo[47163:1508917] name: image_pic
```

打印结果来看，我们成功在分类上添加了一个属性，实现了它的`setter`和`getter`方法。
 通过关联对象实现的属性的内存管理也是有`ARC`管理的，所以我们只需要给定适当的内存策略就行了，不需要操心对象的释放。

#### 方法魔法(Method Swizzling)

对上面的`UIImage`的分类进行进一步扩展：

```
// UIImage+Runtime.h
@interface UIImage (Runtime)

@property (nonatomic, copy) NSString *name;

@end

// UIImage+Runtime.m
#import "UIImage+Runtime.h"
#import <objc/runtime.h>

@implementation UIImage (Runtime)

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class class = [self class];
        
        Method originalMethod = class_getClassMethod(class, @selector(imageNamed:));
        Method swizzledMethod = class_getClassMethod(class, @selector(rep_imageNamed:));
        
        BOOL add = class_addMethod(class, @selector(imageNamed:), method_getImplementation(swizzledMethod), method_getTypeEncoding(swizzledMethod));
        if (add) {
            class_replaceMethod(class, @selector(rep_imageNamed:), method_getImplementation(originalMethod), method_getTypeEncoding(originalMethod));
        } else {
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}

+ (UIImage *)rep_imageNamed:(NSString *)imageName {
    UIImage *image = [self rep_imageNamed:imageName];
    image.name = imageName;
    return image;
}

- (void)setName:(NSString *)name {
    objc_setAssociatedObject(self, @selector(name), name, OBJC_ASSOCIATION_COPY_NONATOMIC);
}

- (NSString *)name {
    return objc_getAssociatedObject(self, _cmd);
}

@end

// ViewController.m
- (void)viewDidLoad {
    [super viewDidLoad];

    UIImage *image = [UIImage imageNamed:@"image_pic"];
    NSLog(@"name: %@", image.name);
}
```

打印日志：

```
2020-01-02 11:45:01.073499+0800 Runtime-Demo[47296:1522764] name: image_pic
```

`swizzling`应该只在`+load`中完成。 在 `Objective-C` 的运行时中，每个类有两个方法都会自动调用。`+load` 是在一个类被初始装载时调用，`+initialize` 是在应用第一次调用该类的类方法或实例方法前调用的。两个方法都是可选的，并且只有在方法被实现的情况下才会被调用。

`swizzling`应该只在`dispatch_once` 中完成,由于`swizzling` 改变了全局的状态，所以我们需要确保每个预防措施在运行时都是可用的。原子操作就是这样一个用于确保代码只会被执行一次的预防措施，就算是在不同的线程中也能确保代码只执行一次。`Grand Central Dispatch 的 dispatch_once`满足了所需要的需求，并且应该被当做使用`swizzling` 的初始化单例方法的标准。

#### KVO实现

> 全称是Key-value observing，翻译成键值观察。提供了一种当其它对象属性被修改的时候能通知当前对象的机制。再MVC大行其道的Cocoa中，KVO机制很适合实现model和controller类之间的通讯。

`KVO`的实现依赖于`Objective-C`强大的`Runtime`，当观察某对象`A`时，`KVO`机制动态创建一个对象`A`当前类的子类，并为这个新的子类重写了被观察属性`keyPath`的`setter`方法。`setter`方法随后负责通知观察对象属性的改变状况。

`Apple`使用了`isa-swizzing`来实现`KVO`。当观察对象`A`时，`KVO`机制动态创建新的名为`NSKVONotifying_A`的新类，该类继承自对象A的本类，并且`KVO`为`NSKVONotofying_A`重写观察属性的`setter`方法，`setter`方法负责在调用元`setter`方法之前和之后，通知所有观察对象属性值的更改情况。

NSKVONotifying_A 类剖析

```
NSLog(@"self->isa:%@",self->isa);  
NSLog(@"self class:%@",[self class]);  

打印结果：
self->isa:A
self class:A

在建立KVO监听之后，打印结果为：
self->isa:NSKVONotifying_A
self class:A
```

在这个过程，被观察对象的 `isa` 指针从指向原来的 `A` 类，被`KVO` 机制修改为指向系统新创建的子类`NSKVONotifying_A` 类，来实现当前类属性值改变的监听；
当我们从应用层面上看来，完全没有意识到有新的类出现，这是系统隐藏了对`KVO`的底层实现过程，让我们误以为还是原来的类。但是此时如果我们创建一个新的名为`NSKVONotifying_A`的类，就会发现系统运行到注册`KVO`的那段代码时程序就崩溃，因为系统在注册监听的时候动态创建了名为`NSKVONotifying_A`的中间类，并指向这个中间类。

NSKVONotifying_A中`setter`方法剖析

```
// KVO为子类的观察者属性重写调用存取方法的工作原理在代码中相当于
- (void)setName:(NSString *)newName { 
      [self willChangeValueForKey:@"name"];    //KVO 在调用存取方法之前总调用 
      [super setValue:newName forKey:@"name"]; //调用父类的存取方法 
      [self didChangeValueForKey:@"name"];     //KVO 在调用存取方法之后总调用
}
```

`KVO`的键值观察通知依赖于`NSObject`的方格方法:`willChangeValueForKey:`和`didChangeValueForKey:`，在存值的前后分别调用这2个方法；
被观察属性发生改变之前，`willChangeValueForKey:`被调用，通知系统该`keyPath`的属性值即将变更；当改变发生后，`didChangeValueForKey:`被调用通知系统该`keyPath`的属性值已经变更；之后，`observeValueForKey:ofObject:change:context:`也会被调用。且重写观察属性的`setter`方法这种继承方式的注入在运行时而不是在编译时实现的。

#### NSCoding的自动归档与解档

原理描述：用`runtime`提供的函数遍历`Model`自身所有属性，并对属性进行`encode`和`decode`操作。
核心方法：在`Model`的基类中重写方法：

```
- (id)initWithCoder:(NSCoder *)aDecoder {
    if (self = [super init]) {
        unsigned int outCount;
        Ivar * ivars = class_copyIvarList([self class], &outCount);
        for (int i = 0; i < outCount; i ++) {
            Ivar ivar = ivars[i];
            NSString * key = [NSString stringWithUTF8String:ivar_getName(ivar)];
            [self setValue:[aDecoder decodeObjectForKey:key] forKey:key];
        }
    }
    return self;
}

- (void)encodeWithCoder:(NSCoder *)aCoder {
    unsigned int outCount;
    Ivar * ivars = class_copyIvarList([self class], &outCount);
    for (int i = 0; i < outCount; i ++) {
        Ivar ivar = ivars[i];
        NSString * key = [NSString stringWithUTF8String:ivar_getName(ivar)];
        [aCoder encodeObject:[self valueForKey:key] forKey:key];
    }
}
```

#### 实现字典和模型的自动转换(MJExtension)

原理描述：用`runtime`提供的函数遍历`Model`自身所有属性，如果属性在`json`中有对应的值，则将其赋值。

核心方法：在`NSObject`的分类中添加方法

```
- (instancetype)initWithDict:(NSDictionary *)dict {

    if (self = [super init]) {
        //(1)获取类的属性及属性对应的类型
        NSMutableArray * keys = [NSMutableArray array];
        NSMutableArray * attributes = [NSMutableArray array];
        /*
         * 例子
         * name = value3 attribute = T@"NSString",C,N,V_value3
         * name = value4 attribute = T^i,N,V_value4
         */
        unsigned int outCount;
        objc_property_t * properties = class_copyPropertyList([self class], &outCount);
        for (int i = 0; i < outCount; i ++) {
            objc_property_t property = properties[i];
            //通过property_getName函数获得属性的名字
            NSString * propertyName = [NSString stringWithCString:property_getName(property) encoding:NSUTF8StringEncoding];
            [keys addObject:propertyName];
            //通过property_getAttributes函数可以获得属性的名字和@encode编码
            NSString * propertyAttribute = [NSString stringWithCString:property_getAttributes(property) encoding:NSUTF8StringEncoding];
            [attributes addObject:propertyAttribute];
        }
        //立即释放properties指向的内存
        free(properties);

        //(2)根据类型给属性赋值
        for (NSString * key in keys) {
            if ([dict valueForKey:key] == nil) continue;
            [self setValue:[dict valueForKey:key] forKey:key];
        }
    }
    return self;

}
```