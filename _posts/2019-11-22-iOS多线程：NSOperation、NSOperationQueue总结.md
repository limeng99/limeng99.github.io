---
layout: post
title: "iOS多线程：NSOperation、NSOperationQueue总结"
author: "李萌"
categories: learning
tags: [learning]
feature-img: "assets/img/article/operation.jpg"
thumbnail: "assets/img/article/operation.jpg"
---

NSOperation、NSOperationQueue 是苹果提供给我们的一套多线程解决方案。NSOperation、NSOperationQueue是基于GCD更高一层的封装，完全面向对象。但是比GCD更简单易用、代码可读性也更高。

## 1. 简介

NSOperation本身是抽象基类，不能直接使用，但是他封装了需要执行的操作和执行操作所需的数据方法等。在NSOperation基础上，系统提供了两个子类NSBlockOperation和NSInvocationOperation供我们具体使用，当然，我们也可以自己封装自定义的NSOperation

使用NSOperation、NSOperationQueue优势：

- 可添加完成的代码块，在操作完成后执行；
- 添加操作之间的依赖关系，方便的控制执行顺序；
-  设定操作执行的优先级；
-  可以很方便的取消一个操作的执行；
- 使用 KVO 观察对操作执行状态的更改：`isExecuteing`、`isFinished`、`isCancelled`

## 2.  操作-NSOperation和操作队列-NSOperationQueue

在 NSOperation、NSOperationQueue中也有类似的**任务（操作）**和**队列（操作队列）**的概念。

 **操作（Operation）：**

- 执行操作的意思，换句话说就是你在线程中执行的那段代码。

- 在 GCD 中是放在 block中的。在 NSOperation 中，我们使用 NSOperation 子类 NSInvocationOperation、NSBlockOperation，或者自定义子类来封装操作。

**操作队列（Operation Queues）：**

- 这里的队列指操作队列，即用来存放操作的队列。不同于 GCD中的调度队列 FIFO（先进先出的原则。NSOperationQueue对于添加到队列中的操作，首先进入准备就绪的状态（就绪状态取决于操作之间的依赖关系），然后进入就绪状态的操作的**开始执行顺序**（非结束执行顺序）由操作之间相对的优先级决定（优先级是操作对象自身的属性）
- 操作队列通过设置**最大并发操作数（`maxConcurrentOperationCount`）**来控制并发、串行。
- NSOperationQueue 为我们提供了两种不同类型的队列：主队列和自定义队列。主队列运行在主线程之上，而自定义队列在后台执行。

## 3. NSOperation的常用方法、属性介绍

### 3.1 NSOperation方法介绍

```
// NSOperation实例初始化方法，用于创建一个NSOperation实例。
- (instancetype)init;

// 该方法是operation的起点，如果需要创建并发operation必须，覆盖start方法。同时调用start方法。
- (void)start;

// 该property，是用于标示某个operation是否cancel。对于多线程来说需要不断检测这个值。
- (BOOL)isCancelled;

// 调用cancel方法会取消一个operation，但是如果operation加入到Queue中或者operation已经start了，则无法取消成功，调用cancel也不一定立即执行cancel操作，需要等待时间周期。
- (void)cancel;

// 判定operation是否正在执行。
- (BOOL)isExecuting;

// 判定operation是否完成，cancel掉某个operation，也会将该operation的该字段设置成为YES。
- (BOOL)isFinished;

// 判定该线程是否是并发线程，即调用该operation的start方法的线程是否与operation所在线程相同。
- (BOOL)isConcurrent;

// 在start方法开始之前，需要确定operation是否ready，默认为YES，如果该operation没有ready，则不会start。
- (BOOL)isReady;

// 该方法用于配置operation之间的依赖关系，涉及执行顺序。如果不是手动调用start去执行operation，一定要在将其加入到Queue之前做好依赖，因为一旦加入到Queue中，其也许很快会执行，依赖关系将不会起作用。
- (void)addDependency:(NSOperation *)op;

// 相对应add，其为移除两个operation之间的依赖关系。
- (void)removeDependency:(NSOperation *)op;

// 获取operation的依赖关系的数组。
- (NSArray *)dependencies;

// //如果将operation加入到Queue中，设定其在Queue中的优先级，优先级高的先执行的概率大，但并不代表一定会先执行，执行顺序稍后介绍。
- (NSOperationQueuePriority)queuePriority;

// setter方法。
- (void)setQueuePriority:(NSOperationQueuePriority)p;

// 在operation完成之后会调用completionBlock，你可以自定义执行行为。
- (void (^)(void))completionBlock NS_AVAILABLE(10_6, 4_0);

// 设定是否等待operation执行结束，如果为YES，该线程会一直等待operation执行结束，才会执行接下来的代码。
- (void)waitUntilFinished NS_AVAILABLE(10_6, 4_0);

// 设定operation的线程优先级，涉及执行顺序稍后介绍。线程优先级默认为0.5，最低为0，最大为1.即使设定了线程优先级，也只能保证其在main方法范围内有效，operation的其他代码仍然执行在默认线程。
- (double)threadPriority NS_AVAILABLE(10_6, 4_0);
```

### 3.2 NSBlockOperation方法介绍

```
// 创建NSBlockOperation对象
+ (instancetype)blockOperationWithBlock:(void (^)(void))block;

// 通过addExecutionBlock:方法添加更多的操作
- (void)addExecutionBlock:(void (^)(void))block;
```

### 3.3 NSInvocationOperation方法介绍

```
// 创建NSInvocationOperation对象 
- (instancetype)initWithTarget:(id)target selector:(SEL)sel object:(id)arg;
```

## 4、NSOperationQueue的常用方法、属性介绍

```
// 取消队列的所有操作
- (void)cancelAllOperations;

// 设置操作的暂停和恢复，YES 代表暂停队列，NO 代表恢复队列
- (void)setSuspended:(BOOL)b;

// 判断队列是否处于暂停状态
- (BOOL)isSuspended;

// 阻塞当前线程，直到队列中的操作全部执行完毕
- (void)waitUntilAllOperationsAreFinished;

// 队列中添加一个 NSBlockOperation 类型操作对象
- (void)addOperationWithBlock:(void (^)(void))block;

// 向队列中添加操作数组，wait 标志是否阻塞当前线程直到所有操作结束
- (void)addOperations:(NSArray *)ops waitUntilFinished:(BOOL)wait;

// 当前在队列中的操作数组（某个操作执行结束后会自动从这个数组清除）
- (NSArray *)operations;

// 当前队列中的操作数
- (NSUInteger)operationCount;

//  获取当前队列，如果当前线程不是在 NSOperationQueue 上运行则返回 nil
+ (instancetype)currentQueue;

// 获取主队列
+ (instancetype)mainQueue; 

这里的暂停和取消（包括操作的取消和队列的取消）并不代表可以将当前的操作立即取消，而是当当前的操作执行完毕之后不再执行新的操作。
暂停和取消的区别就在于：暂停操作之后还可以恢复操作，继续向下执行；而取消操作之后，所有的操作就清空了，无法再接着执行剩下的操作。
```

## 5、 NSOperation 和 NSOperationQueue 基本使用

### 5.1 创建操作

NSOperation是个抽象类，不能用来封装操作。我们只有使用它的子类来封装操作。我们有三种方式来封装操作。

- 使用子类 NSInvocationOperation
- 使用子类 NSBlockOperation
- 自定义继承自 NSOperation 的子类，通过实现内部相应的方法来封装操作

#### 5.1.1 使用子类 NSInvocationOperation

```
// 使用子类 NSInvocationOperation
- (void)invocationOperation {

    // 1.创建 NSInvocationOperation 对象
    NSInvocationOperation *op = [[NSInvocationOperation alloc] initWithTarget:self selector:@selector(operationTask) object:nil];

    // 2.调用 start 方法开始执行操作
    [op start];
}

// 操作任务
- (void)operationTask {
    for (int i = 0; i < 2; i++) {
        [NSThread sleepForTimeInterval:1]; // 模拟耗时操作
        NSLog(@"thread --- %@", [NSThread currentThread]); // 打印当前线程
    }
}
```

打印日志

```
2019-11-22 10:44:48.218825+0800 多线程-demo[65383:21238597] thread --- <NSThread: 0x600002ac0bc0>{number = 1, name = main}
2019-11-22 10:44:49.220306+0800 多线程-demo[65383:21238597] thread --- <NSThread: 0x600002ac0bc0>{number = 1, name = main}
```

在没有使用 NSOperationQueue、在主线程中单独使用使用子类 NSInvocationOperation 执行一个操作的情况下，操作是在当前线程执行的，并没有开启新线程。

#### 5.1.2 使用子类 NSBlockOperation

```
// 使用子类 NSBlockOperation
- (void)blockOperation {

    // 1.创建 NSBlockOperation 对象
    NSBlockOperation *op = [NSBlockOperation blockOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:1]; // 模拟耗时操作
            NSLog(@"thread --- %@", [NSThread currentThread]); // 打印当前线程
        }
    }];

    // 2.调用 start 方法开始执行操作
    [op start];
}
```

打印日志

```
2019-11-22 10:48:24.270508+0800 多线程-demo[65399:21245204] thread --- <NSThread: 0x6000016693c0>{number = 1, name = main}
2019-11-22 10:48:25.271132+0800 多线程-demo[65399:21245204] thread --- <NSThread: 0x6000016693c0>{number = 1, name = main}
```

在没有使用 NSOperationQueue、在主线程中单独使用NSBlockOperation执行一个操作的情况下，操作是在当前线程执行的，并没有开启新线程

NSBlockOperation还提供了一个方法 `addExecutionBlock:`，通过 `addExecutionBlock:` 就可以为 NSBlockOperation添加额外的操作。这些操作（包括 `blockOperationWithBlock` 中的操作）可以在不同的线程中同时（并发）执行。只有当所有相关的操作已经完成执行时，才视为完成。

```
- (void)blockOperationAddExecutionBlock {

    // 1.创建 NSBlockOperation 对象
    NSBlockOperation *op = [NSBlockOperation blockOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"thread1 --- %@", [NSThread currentThread]); // 打印当前线程
        }
    }];

    // 2.添加额外的操作
    [op addExecutionBlock:^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"thread2 --- %@", [NSThread currentThread]); // 打印当前线程
        }
    }];
    [op addExecutionBlock:^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"thread3 --- %@", [NSThread currentThread]); // 打印当前线程
        }
    }];
    // 3.调用 start 方法开始执行操作
    [op start];
}
```

打印日志

```
2019-11-22 10:54:03.107072+0800 多线程-demo[65435:21257518] thread2 --- <NSThread: 0x6000036b06c0>{number = 5, name = (null)}
2019-11-22 10:54:03.107073+0800 多线程-demo[65435:21257522] thread3 --- <NSThread: 0x6000036dab80>{number = 6, name = (null)}
2019-11-22 10:54:03.107072+0800 多线程-demo[65435:21257472] thread1 --- <NSThread: 0x6000036b5c80>{number = 1, name = main}
2019-11-22 10:54:04.107443+0800 多线程-demo[65435:21257472] thread1 --- <NSThread: 0x6000036b5c80>{number = 1, name = main}
2019-11-22 10:54:04.107468+0800 多线程-demo[65435:21257518] thread2 --- <NSThread: 0x6000036b06c0>{number = 5, name = (null)}
2019-11-22 10:54:04.107494+0800 多线程-demo[65435:21257522] thread3 --- <NSThread: 0x6000036dab80>{number = 6, name = (null)}
```

使用子类 NSBlockOperation，并调用方法 `AddExecutionBlock:` 的情况下，`blockOperationWithBlock:`方法中的操作 和 `addExecutionBlock:` 中的操作是在不同的线程中异步执行的。而且，这次执行结果中 `blockOperationWithBlock:`方法中的操作也不是在当前线程（主线程）中执行的。从而印证了`blockOperationWithBlock:` 中的操作也可能会在其他线程（非当前线程）中执行。

一般情况下，如果一个 NSBlockOperation 对象封装了多个操作。NSBlockOperation是否开启新线程，取决于操作的个数。如果添加的操作的个数多，就会自动开启新线程。当然开启的线程数是由系统来决定的。

#### 5.1.3 使用自定义继承自 NSOperation 的子类

如果使用子类 NSInvocationOperation、NSBlockOperation不能满足日常需求，我们可以使用自定义继承自NSOperation 的子类。可以通过重写 `main` 或者 `start` 方法 来定义自己的 NSOperation 对象。重写`main`方法比较简单，我们不需要管理操作的状态属性 `isExecuting` 和 `isFinished`。当 `main` 执行完返回的时候，这个操作就结束了。

```
// LMOperation.h 文件
@interface LMOperation : NSOperation

@end

// LMOperation.m 文件
@implementation LMOperation

- (void)main {
    if (!self.isCancelled) {
        for (int i = 0; i < 2; i++) {
            NSLog(@"thread --- %@", [NSThread currentThread]);
        }
    }
}

@end

// 导入头文件 LMOperation.h
- (void)customOperation {

    // 1.创建 YSCOperation 对象
    LMOperation *op = [[LMOperation alloc] init];
    
    // 2.调用 start 方法开始执行操作
    [op start];
}
```

打印日志

```
2019-11-22 11:01:15.707199+0800 多线程-demo[65475:21275462] thread --- <NSThread: 0x600003e3a100>{number = 1, name = main}
2019-11-22 11:01:15.707402+0800 多线程-demo[65475:21275462] thread --- <NSThread: 0x600003e3a100>{number = 1, name = main}
```

在没有使用 NSOperationQueue、在主线程单独使用自定义继承自 NSOperation 的子类的情况下，是在主线程执行操作，并没有开启新线程。

### 5.2 创建队列

NSOperationQueue一共有两种队列：主队列、自定义队列。其中自定义队列同时包含了串行、并发功能。下边是主队列、自定义队列的基本创建方法和特点。

**主队列**

凡是添加到主队列中的操作，都会放到主线程中执行，但不包括操作使用`addExecutionBlock:`添加的额外操作，额外操作可能在其他线程执行

```
// 主队列获取方法
NSOperationQueue *queue = [NSOperationQueue mainQueue];
```

**自定义队列**

添加到这种队列中的操作，就会自动放到子线程中执行， 同时包含了：串行、并发功能。

```
// 自定义队列创建方法
NSOperationQueue *queue = [[NSOperationQueue alloc] init];
```

### 5.3 将操作加入到队列中

#### 5.3.1  addOperation添加操作到队列

```
// 使用 addOperation: 将操作加入到操作队列中
- (void)addOperationToQueue {

    // 1.创建队列
    NSOperationQueue *queue = [[NSOperationQueue alloc] init];

    // 2.创建操作
    // 使用 NSInvocationOperation 创建操作1
    NSInvocationOperation *op1 = [[NSInvocationOperation alloc] initWithTarget:self selector:@selector(operationTask) object:nil];

    // 使用 NSBlockOperation 创建操作3
    NSBlockOperation *op2 = [NSBlockOperation blockOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:1]; // 模拟耗时操作
            NSLog(@"thread2 --- %@", [NSThread currentThread]); // 打印当前线程
        }
    }];
    [op2 addExecutionBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:1]; // 模拟耗时操作
            NSLog(@"thread3 --- %@", [NSThread currentThread]); // 打印当前线程
        }
    }];

    // 3.使用 addOperation: 添加所有操作到队列中
    [queue addOperation:op1]; // [op1 start]
    [queue addOperation:op2]; // [op2 start]
}

// 操作任务
- (void)operationTask {
    for (int i = 0; i < 2; i++) {
        [NSThread sleepForTimeInterval:1]; // 模拟耗时操作
        NSLog(@"thread1 --- %@", [NSThread currentThread]); // 打印当前线程
    }
}
```

打印日志

```
2019-11-22 11:10:16.160387+0800 多线程-demo[65529:21296746] thread2 --- <NSThread: 0x600001eac940>{number = 3, name = (null)}
2019-11-22 11:10:16.160395+0800 多线程-demo[65529:21296740] thread1 --- <NSThread: 0x600001ec4300>{number = 5, name = (null)}
2019-11-22 11:10:16.160402+0800 多线程-demo[65529:21296743] thread3 --- <NSThread: 0x600001ec6c80>{number = 7, name = (null)}
2019-11-22 11:10:17.160850+0800 多线程-demo[65529:21296746] thread2 --- <NSThread: 0x600001eac940>{number = 3, name = (null)}
2019-11-22 11:10:17.160934+0800 多线程-demo[65529:21296740] thread1 --- <NSThread: 0x600001ec4300>{number = 5, name = (null)}
2019-11-22 11:10:17.160934+0800 多线程-demo[65529:21296743] thread3 --- <NSThread: 0x600001ec6c80>{number = 7, name = (null)}
```

使用NSOperation子类创建操作，并使用 `addOperation:` 将操作加入到操作队列后能够开启新线程，进行并发执行。

#### 5.3.2  addOperationWithBlock添加操作到队列

```
// 使用 addOperationWithBlock: 将操作加入到操作队列中
- (void)addOperationWithBlockToQueue {
    // 1.创建队列
    NSOperationQueue *queue = [[NSOperationQueue alloc] init];

    // 2.使用 addOperationWithBlock: 添加操作到队列中
    [queue addOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
            NSLog(@"thread1 --- %@", [NSThread currentThread]); // 打印当前线程
        }
    }];
    [queue addOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
            NSLog(@"thread2 ---%@", [NSThread currentThread]); // 打印当前线程
        }
    }];
}
```

打印日志

```
2019-11-22 11:14:29.227160+0800 多线程-demo[65543:21306313] thread1 --- <NSThread: 0x6000001ee080>{number = 6, name = (null)}
2019-11-22 11:14:29.227163+0800 多线程-demo[65543:21306314] thread2 ---<NSThread: 0x6000001b8800>{number = 5, name = (null)}
2019-11-22 11:14:31.231445+0800 多线程-demo[65543:21306314] thread2 ---<NSThread: 0x6000001b8800>{number = 5, name = (null)}
2019-11-22 11:14:31.231449+0800 多线程-demo[65543:21306313] thread1 --- <NSThread: 0x6000001ee080>{number = 6, name = (null)}
```

使用 `addOperationWithBlock:` 将操作加入到操作队列后能够开启新线程，进行并发执行。

## 6. NSOperationQueue 控制串行执行、并发执行

NSOperationQueue通过`maxConcurrentOperationCount`(最大并发操作数)。用来控制一个特定队列中可以有多少个操作同时参与并发执行。

<!--注意：这里 `maxConcurrentOperationCount` 控制的不是并发线程的数量，而是一个队列中同时能并发执行的最大操作数。而且一个操作也并非只能在一个线程中运行。-->

**最大并发操作数:maxConcurrentOperationCount**

- `maxConcurrentOperationCount` 默认情况下为-1，表示不进行限制，可进行并发执行

- `maxConcurrentOperationCount` 为1时，队列为串行队列。只能串行执行

- `maxConcurrentOperationCount` 大于1时，队列为并发队列。操作并发执行，当然这个值不应超过系统限制，即使自己设置一个很大的值，系统也会自动调整为 min{自己设定的值，系统设定的默认最大值}。

```
// 设置 MaxConcurrentOperationCount（最大并发操作数)
- (void)setMaxConcurrentOperationCount {

    // 1.创建队列
    NSOperationQueue *queue = [[NSOperationQueue alloc] init];

    // 2.设置最大并发操作数
    queue.maxConcurrentOperationCount = 1; // 串行队列
// queue.maxConcurrentOperationCount = 2; // 并发队列
// queue.maxConcurrentOperationCount = 6; // 并发队列

    // 3.添加操作
    [queue addOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:1]; // 模拟耗时操作
            NSLog(@"thread1 --- %@", [NSThread currentThread]); // 打印当前线程
        }
    }];
    [queue addOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:1]; // 模拟耗时操作
            NSLog(@"thread2 ---%@", [NSThread currentThread]); // 打印当前线程
        }
    }];
}
```

  最大并发操作数为1，为串行队列，打印日志：

```
2019-11-22 11:22:59.988321+0800 多线程-demo[65566:21319793] thread1 --- <NSThread: 0x60000388d6c0>{number = 6, name = (null)}
2019-11-22 11:23:00.992952+0800 多线程-demo[65566:21319793] thread1 --- <NSThread: 0x60000388d6c0>{number = 6, name = (null)}
2019-11-22 11:23:01.997176+0800 多线程-demo[65566:21319797] thread2 ---<NSThread: 0x6000038b4a00>{number = 3, name = (null)}
2019-11-22 11:23:03.001493+0800 多线程-demo[65566:21319797] thread2 ---<NSThread: 0x6000038b4a00>{number = 3, name = (null)}
```

最大并发操作数为2，为并发队列，打印日志：

```
2019-11-22 11:24:01.876410+0800 多线程-demo[65579:21321486] thread2 ---<NSThread: 0x6000031543c0>{number = 5, name = (null)}
2019-11-22 11:24:01.876436+0800 多线程-demo[65579:21321488] thread1 --- <NSThread: 0x60000315e540>{number = 6, name = (null)}
2019-11-22 11:24:02.876903+0800 多线程-demo[65579:21321488] thread1 --- <NSThread: 0x60000315e540>{number = 6, name = (null)}
2019-11-22 11:24:02.876903+0800 多线程-demo[65579:21321486] thread2 ---<NSThread: 0x6000031543c0>{number = 5, name = (null)}
```

最大并发操作数为6，为并发队列，打印日志：

```
2019-11-22 11:24:48.046063+0800 多线程-demo[65581:21322720] thread1 --- <NSThread: 0x600003248b80>{number = 5, name = (null)}
2019-11-22 11:24:48.046064+0800 多线程-demo[65581:21322718] thread2 ---<NSThread: 0x600003249c80>{number = 7, name = (null)}
2019-11-22 11:24:49.047850+0800 多线程-demo[65581:21322718] thread2 ---<NSThread: 0x600003249c80>{number = 7, name = (null)}
2019-11-22 11:24:49.047867+0800 多线程-demo[65581:21322720] thread1 --- <NSThread: 0x600003248b80>{number = 5, name = (null)}
```

当最大并发操作数为1时，操作是按顺序串行执行的，并且一个操作完成之后，下一个操作才开始执行。当最大操作并发数为2时，操作是并发执行的，可以同时执行两个操作。而开启线程数量是由系统决定的，不需要我们来管理。

## 7. NSOperation 操作依赖

NSOperation、NSOperationQueue最吸引人的地方是它能添加操作之间的依赖关系。通过操作依赖，我们可以很方便的控制操作之间的执行先后顺序。

- `- (void)addDependency:(NSOperation *)op;` 添加依赖，使当前操作依赖于操作 op 的完成。

-  `- (void)removeDependency:(NSOperation *)op;` 移除依赖，取消当前操作对操作 op 的依赖。

- `@property (readonly, copy) NSArray *dependencies;` 在当前操作开始执行之前完成执行的所有操作对象数组。

```
// 操作依赖
- (void)addDependency {

    // 1.创建队列
    NSOperationQueue *queue = [[NSOperationQueue alloc] init];

    // 2.创建操作
    NSBlockOperation *op1 = [NSBlockOperation blockOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:1]; // 模拟耗时操作
            NSLog(@"thread1 --- %@", [NSThread currentThread]); // 打印当前线程
        }
    }];
    NSBlockOperation *op2 = [NSBlockOperation blockOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:1]; // 模拟耗时操作
            NSLog(@"thread2 --- %@", [NSThread currentThread]); // 打印当前线程
        }
    }];

    // 3.添加依赖
    [op2 addDependency:op1]; // 让op2 依赖于 op1，则先执行op1，在执行op2

    // 4.添加操作到队列中
    [queue addOperation:op1];
    [queue addOperation:op2];
}
```

打印日志

```
2019-11-22 11:27:48.796309+0800 多线程-demo[65595:21327089] thread1 --- <NSThread: 0x6000005929c0>{number = 6, name = (null)}
2019-11-22 11:27:49.797401+0800 多线程-demo[65595:21327089] thread1 --- <NSThread: 0x6000005929c0>{number = 6, name = (null)}
2019-11-22 11:27:50.802102+0800 多线程-demo[65595:21327093] thread2 --- <NSThread: 0x6000005acf40>{number = 3, name = (null)}
2019-11-22 11:27:51.805816+0800 多线程-demo[65595:21327093] thread2 --- <NSThread: 0x6000005acf40>{number = 3, name = (null)}
```

通过添加操作依赖，无论运行几次，其结果都是 op1 先执行，op2 后执行。

## 8. NSOperation 优先级

NSOperation 提供了`queuePriority`（优先级）属性，`queuePriority`属性适用于同一操作队列中的操作，不适用于不同操作队列中的操作。默认情况下，所有新创建的操作对象优先级都是`NSOperationQueuePriorityNormal`。但是我们可以通过`setQueuePriority:`方法来改变当前操作在同一队列中的执行优先级。

```
// 优先级的取值
typedef NS_ENUM(NSInteger, NSOperationQueuePriority) {
    NSOperationQueuePriorityVeryLow = -8L,
    NSOperationQueuePriorityLow = -4L,
    NSOperationQueuePriorityNormal = 0,
    NSOperationQueuePriorityHigh = 4,
    NSOperationQueuePriorityVeryHigh = 8
};
```

对于添加到队列中的操作，首先进入准备就绪的状态（就绪状态取决于操作之间的依赖关系），然后进入就绪状态的操作的**开始执行顺序**（非结束执行顺序）由操作之间相对的优先级决定（优先级是操作对象自身的属性）。

**queuePriority 属性的作用**

- 一个操作的所有依赖都已经完成时，操作对象通常会进入准备就绪状态，等待执行。

- `queuePriority` 属性决定了**进入准备就绪状态下的操作**之间的开始执行顺序。并且，优先级不能取代依赖关系。

- 如果一个队列中既包含高优先级操作，又包含低优先级操作，并且两个操作都已经准备就绪，那么队列先执行高优先级操作。

- 如果，一个队列中既包含了准备就绪状态的操作，又包含了未准备就绪的操作，未准备就绪的操作优先级比准备就绪的操作优先级高。那么，虽然准备就绪的操作优先级低，也会优先执行。优先级不能取代依赖关系。如果要控制操作间的启动顺序，则必须使用依赖关系。

## 9. NSOperation、NSOperationQueue 线程间的通信

```
// 线程间通信
- (void)communication {

    // 1.创建队列
    NSOperationQueue *queue = [[NSOperationQueue alloc]init];

    // 2.添加操作
    [queue addOperationWithBlock:^{
        // 异步进行耗时操作
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:1]; // 模拟耗时操作
            NSLog(@"thread1 --- %@", [NSThread currentThread]); // 打印当前线程
        }

        // 回到主线程
        [[NSOperationQueue mainQueue] addOperationWithBlock:^{
            // 进行一些 UI 刷新等操作
            for (int i = 0; i < 2; i++) {
                [NSThread sleepForTimeInterval:1]; // 模拟耗时操作
                NSLog(@"thread2 --- %@", [NSThread currentThread]); // 打印当前线程
            }
        }];
    }];
}
```

打印日志

```
2019-11-22 11:37:11.257207+0800 多线程-demo[65626:21341680] thread1 --- <NSThread: 0x600003fa9c40>{number = 6, name = (null)}
2019-11-22 11:37:12.262269+0800 多线程-demo[65626:21341680] thread1 --- <NSThread: 0x600003fa9c40>{number = 6, name = (null)}
2019-11-22 11:37:13.263827+0800 多线程-demo[65626:21341609] thread2 --- <NSThread: 0x600003fc2180>{number = 1, name = main}
2019-11-22 11:37:14.265262+0800 多线程-demo[65626:21341609] thread2 --- <NSThread: 0x600003fc2180>{number = 1, name = main}
```

通过线程间的通信，先在其他线程中执行操作，等操作执行完了之后再回到主线程执行主线程的相应操作。