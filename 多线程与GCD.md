#多线程总结

##1.一些概念
首先复习一下操作系统，回顾一下几个核心的概念

- 进程：可以理解成一个运行中的应用程序，是系统进行资源分配和调度的基本单位，是操作系统结构的基础，主要管理资源。
- 线程：是进程的基本执行单元，一个进程对应多个线程。
- 主线程：处理UI，所有更新UI的操作都必须在主线程上执行。不要把耗时操作放在主线程，会卡界面。

- 多线程：在同一时刻，一个CPU只能处理1条线程，但CPU可以在多条线程之间快速的切换，只要切换的足够快，就造成了多线程一同执行的假象。

Q&A :那么为什么我们需要多线程？ 因为我们耗时操作不能放在主线程上执行，必须把耗时操作放在后台执行。

##2.iOS线程的生命周期
总体来说，线程的生命周期是:  新建 - 就绪 - 运行 - 阻塞 - 死亡

- 新建: 实例化线程对象，例如getThread等，这个不用过多阐述。
- 就绪: 也就是runnable状态，可运行状态。
- 运行: running状态，在这个状态下CPU获取了线程的全部控制权，程序员没有办法对该状态下的线程进行干预(kill应用除外.jpg
- 阻塞&死亡 阻塞状态下的线程停止执行/死亡状态下的线程会被中止执行并回收

##3.多线程的实现方式
在iOS上，多线程一般可以分为四种解决方案，分别是 pthread,NSThread,GCD,NSperation四种。但由于前两种都是需要手动对线程的生命周期进行管理，使用频率较低，所以只讨论后续两种。


###3.1 GCD

####3.1.1 GCD是什么
> GCD(Grand Central Dispatch) 是 Apple 將複雜且不易使用的 thread 操作方式簡化過的 API，雖然使用起來比起傳統的 thread 會有些限制，但整體來說使用起來輕鬆許多也較安全。它是一个在线程池模式的基础上执行的并发任务.

因此，我们可以简单的认为GCD就是一系列帮我们实现多线程控制的高效Api。（虽然鉴于苹果最近几年的咖喱味，这Api也不见得有多高效。）它会自动帮我们管理线程与任务的生命周期，coder只需要告诉GCD如何执行，不用编写管理的代码。

####3.1.2 GCD的一些基本概念

- 队列：队列是GCD最重要的概念，所有的任务都需要放置在队列中才能执行。它是任务的载体，具体实现可以参考苹果开源的Runloop代码，我们只要知道它是线性结构的就行。队列分为串行队列和并发队列两种。
- 任务：任务就是需要在线程中执行的代码，通常都是放在block中然后添加到对应的队列，等待cpu执行。

####3.1.3 GCD的使用
GCD的使用非常的便利，coder只需要将任务block添加到正确的队列并指定任务执行的方式即可。

以下是具体代码

- 创建队列

`dispatch_queue_t queue = dispatch_queue_creat("test1",DISPATCH_QUEUE_SERIAL)、
`
`dispatch_queue_t queue = dispatch_queue_creat("test2",DISPATCH_QUEUE_CONCURRENT)、
`

其实还存在两个特殊的队列：`dispatch_get_main_queue ``dispatch_get_global_queue`。

`dispatch_get_main_queue`：主队列，本质是一个串行队列，它负责调度主线程的任务。

`dispatch_get_global_queue`：全局并发队列，旨在让用户便捷的使用并发多线程。

- 创建任务
- 
    // 同步执行任务
    dispatch_sync(dispatch_get_global_queue(0, 0), ^{
        // 任务放在这个block里
        NSLog(@"我是同步执行的任务");

    });
    // 异步执行任务
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        // 任务放在这个block里
        NSLog(@"我是异步执行的任务");

    });

####3.1.4 GCD的排列组合
接下来就是最让人头秃的多种队列与任务的组合。

- 主队列同步 

	首先我们来讨论最特殊的，主队列➕同步执行。这种组合会导致主线程卡死。

- 串行同步

- 串行异步

- 并发同步

- 并发异步



- 主队列异步

算了没必要再抄一遍别人写的东西，关于GCD的东西这篇文章写的已经很好了 [多线程总结](https://imlifengfeng.github.io/article/533/)

####3.1.5 GCD的一些其他用法

- 1.我第一讨厌的延时执行

```
//参数1：从现在开始经过多少纳秒，参数2：调度任务的队列，参数3：异步执行的任务
dispatch_after(when, queue, block)
```
！！！！请不要滥用延时执行，经常有人遇到一些bug解决不不了就延时1s看看情况，会带来大坑！！！！！！

- 2.一次执行

```
// 使用dispatch_once函数能保证某段代码在程序运行过程中只被执行1次
static dispatch_once_t onceToken;
dispatch_once(&onceToken, ^{
    // 只执行1次的代码(这里面默认是线程安全的)
});
```

一般用在创建单例

- 3.调度组 （重点，考试会考）

使用场景:有多个耗时操作，需要等这几个耗时操作都完成以后才执行后续的统一操作。

```
//创建调度组
dispatch_group_t group = dispatch_group_create();
//将调度组添加到队列，执行 block 任务
dispatch_group_async(group, queue, block);
//当调度组中的所有任务执行结束后，获得通知，统一做后续操作
dispatch_group_notify(group, dispatch_get_main_queue(), block);
```

一个例子

```
// 分别异步执行2个耗时的操作、2个异步操作都执行完毕后，再回到主线程执行操作
dispatch_group_t group =  dispatch_group_create();
dispatch_group_async(group, dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    // 执行1个耗时的异步操作
});
dispatch_group_async(group, dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    // 执行1个耗时的异步操作
});
dispatch_group_notify(group, dispatch_get_main_queue(), ^{
    // 等前面的异步操作都执行完毕后，回到主线程...
});
```



###3.2 NSOperation

####3.2.1 什么是NSOperation
NSOperation、NSOperationQueue 是苹果提供给我们的一套多线程解决方案。实际上 NSOperation、NSOperationQueue 是基于 GCD 更高一层的封装，完全面向对象。但是比 GCD 更简单易用、代码可读性也更高。

####3.2.2 基本概念

- 操作（Operation）：执行操作的意思，换句话说就是你在线程中执行的那段代码。操作的代码在GCD中是放在block中的。在NSOperation中，我们使用 NSOperation子类NSInvocationOperation、NSBlockOperation，或者自定义子类来封装操作。（其实还有一个和优先级有关的NSOperationQueuePriority，但是用的极少，这里就不写了。

- 操作队列（Operation Queues）：这里的队列指操作队列，即用来存放操作的队列。不同于 GCD 中的调度队列 FIFO（先进先出）的原则.
NSOperationQueue对于添加到队列中的操作，首先进入准备就绪的状态（就绪状态取决于操作之间的依赖关系），然后进入就绪状态的操作的开始执行顺序（非结束执行顺序）由操作之间相对的优先级决定（优先级是操作对象自身的属性）。
操作队列通过设置最大并发操作数（maxConcurrentOperationCount）来控制并发、串行。
NSOperationQueue 为我们提供了两种不同类型的队列：主队列和自定义队列。主队列运行在主线程之上，而自定义队列在后台执行。

####3.2.3 NSOperation/NSOperationQueue的使用
NSOperation创建的操作如果使用start则默认使用当前线程进行同步操作，而NSOperation则是开启异步线程。
NSOperation 实现多线程的使用步骤分为三步：

- 1.创建操作：先将需要执行的操作封装到一个 NSOperation 对象中。
NSOperation 是个抽象类，不能用来封装操作。我们只有使用它的子类来封装操作。我们有三种方式来封装操作。

	- 使用子类 NSInvocationOperation 
	- 使用子类 NSBlockOperation
	- 自定义继承自 NSOperation 的子类，通过实现内部相应的方法来封装操作。

```
	/**
 * 使用子类 NSInvocationOperation
 */
- (void)useInvocationOperation {

    // 1.创建 NSInvocationOperation 对象
    NSInvocationOperation *op = [[NSInvocationOperation alloc] initWithTarget:self selector:@selector(task1) object:nil];

    // 2.调用 start 方法开始执行操作
    [op start];
}
/**
 * 任务1
 */
- (void)task1 {
    for (int i = 0; i < 2; i++) {
        [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
        NSLog(@"1---%@", [NSThread currentThread]); // 打印当前线程
    }
}
```

```
/**
 * 使用子类 NSBlockOperation
 */
- (void)useBlockOperation {

    // 1.创建 NSBlockOperation 对象
    NSBlockOperation *op = [NSBlockOperation blockOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
            NSLog(@"1---%@", [NSThread currentThread]); // 打印当前线程
        }
    }];

    // 2.调用 start 方法开始执行操作
    [op start];
}
```
一般情况下，如果一个 NSBlockOperation 对象封装了多个操作。NSBlockOperation 是否开启新线程，取决于操作的个数。如果添加的操作的个数多，就会自动开启新线程。当然开启的线程数是由系统来决定的。

如果使用子类 NSInvocationOperation、NSBlockOperation 不能满足日常需求，我们可以使用自定义继承自 NSOperation 的子类。可以通过重写 main 或者 start 方法 来定义自己的 NSOperation 对象。重写main方法比较简单，我们不需要管理操作的状态属性 isExecuting 和 isFinished。当 main 执行完返回的时候，这个操作就结束了。

```
// WSLOperation.h 文件
#import <Foundation/Foundation.h>

@interface WSLOperation : NSOperation

@end

// WSLOperation.m 文件
#import "WSLOperation.h"

@implementation WSLOperation

- (void)main {
    if (!self.isCancelled) {
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:2];
            NSLog(@"1---%@", [NSThread currentThread]);
        }
    }
}

@end

```

以上都是直接创建的任务并且直接执行，所以会在当前线程执行。下面创建队列来执行。


- 2.创建队列：创建 NSOperationQueue 对象。
NSOperationQueue 一共有两种队列：主队列、自定义队列。其中自定义队列同时包含了串行、并发功能。凡是添加到主队列中的操作，都会放到主线程中执行。而添加到自定义队列的op会自动放到子线程执行，

```
// 主队列获取方法
NSOperationQueue *queue = [NSOperationQueue mainQueue];
// 自定义队列创建方法
NSOperationQueue *queue = [[NSOperationQueue alloc] init];
```
创建完队列以后，就要把op放到队列中执行

- 3.将操作加入到队列中：将 NSOperation 对象添加到 NSOperationQueue 对象中。一共有两种方法 

```
///需要先创建操作，再将创建好的操作加入到创建好的队列中去。
- (void)addOperation:(NSOperation *)op;
```

```
///无需先创建操作，在 block 中添加操作，直接将包含操作的 block 加入到队列中。
- (void)addOperationWithBlock:(void (^)(void))block;
```

```
/**
 * 使用 addOperation: 将操作加入到操作队列中
 */
- (void)addOperationToQueue {

    // 1.创建队列
    NSOperationQueue *queue = [[NSOperationQueue alloc] init];

    // 2.创建操作
    // 使用 NSInvocationOperation 创建操作1
    NSInvocationOperation *op1 = [[NSInvocationOperation alloc] initWithTarget:self selector:@selector(task1) object:nil];

    // 使用 NSInvocationOperation 创建操作2
    NSInvocationOperation *op2 = [[NSInvocationOperation alloc] initWithTarget:self selector:@selector(task2) object:nil];

    // 使用 NSBlockOperation 创建操作3
    NSBlockOperation *op3 = [NSBlockOperation blockOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
            NSLog(@"3---%@", [NSThread currentThread]); // 打印当前线程
        }
    }];
    [op3 addExecutionBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
            NSLog(@"4---%@", [NSThread currentThread]); // 打印当前线程
        }
    }];

    // 3.使用 addOperation: 添加所有操作到队列中
    [queue addOperation:op1]; // [op1 start]
    [queue addOperation:op2]; // [op2 start]
    [queue addOperation:op3]; // [op3 start]
}

```

```
/**
 * 使用 addOperationWithBlock: 将操作加入到操作队列中
 */

- (void)addOperationWithBlockToQueue {
    // 1.创建队列
    NSOperationQueue *queue = [[NSOperationQueue alloc] init];

    // 2.使用 addOperationWithBlock: 添加操作到队列中
    [queue addOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
            NSLog(@"1---%@", [NSThread currentThread]); // 打印当前线程
        }
    }];
    [queue addOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
            NSLog(@"2---%@", [NSThread currentThread]); // 打印当前线程
        }
    }];
    [queue addOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
            NSLog(@"3---%@", [NSThread currentThread]); // 打印当前线程
        }
    }];
}

```

####3.2.4 maxConcurrentOperationCount,Dependency

- 1. maxConcurrentOperationCount:最大并发操作数。用来控制一个特定队列中可以有多少个操作同时参与并发执行。默认情况下为-1，表示不进行限制，可进行并发执行。为1时，队列为串行队列。只能串行执行。大于1时，队列为并发队列。操作并发执行，当然这个值不应超过系统限制，即使自己设置一个很大的值，系统也会自动调整为 min{自己设定的值，系统设定的默认最大值}。

> 注意：这里 maxConcurrentOperationCount 控制的不是并发线程的数量，而是一个队列中同时能并发执行的最大操作数。而且一个操作也并非只能在一个线程中运行。


```
/**
 * 设置 MaxConcurrentOperationCount（最大并发操作数）
 */
- (void)setMaxConcurrentOperationCount {

    // 1.创建队列
    NSOperationQueue *queue = [[NSOperationQueue alloc] init];

    // 2.设置最大并发操作数
    queue.maxConcurrentOperationCount = 1; // 串行队列
// queue.maxConcurrentOperationCount = 2; // 并发队列
// queue.maxConcurrentOperationCount = 8; // 并发队列

    // 3.添加操作
    [queue addOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
            NSLog(@"1---%@", [NSThread currentThread]); // 打印当前线程
        }
    }];
    [queue addOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
            NSLog(@"2---%@", [NSThread currentThread]); // 打印当前线程
        }
    }];
    [queue addOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
            NSLog(@"3---%@", [NSThread currentThread]); // 打印当前线程
        }
    }];
    [queue addOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
            NSLog(@"4---%@", [NSThread currentThread]); // 打印当前线程
        }
    }];
}

```

- 2.Dependency 操作依赖。NSoperation能添加操作之间的依赖关系。通过操作依赖，我们可以很方便的控制操作之间的执行先后顺序。NSOperation 提供了3个接口供我们管理和查看依赖。
```
-(void)addDependency:(NSOperation *)op; //添加依赖，使当前操作依赖于操作 op 的完成。
```
```
-(void)removeDependency:(NSOperation *)op;//移除依赖取消当前操作对操作 op 的依赖。
```
```
@property (readonly, copy) NSArray <NSOperation * > *dependencies; //在当前操作开始执行之前完成执行的所有操作对象数组。
```

有 A、B 两个操作，其中 A 执行完操作，B 才能执行操作。如果使用依赖来处理的话，那么就需要让操作 B 依赖于操作 A。具体代码如下：

```
/**
 * 操作依赖
 * 使用方法：addDependency:
 */
- (void)addDependency {

    // 1.创建队列
    NSOperationQueue *queue = [[NSOperationQueue alloc] init];

    // 2.创建操作
    NSBlockOperation *op1 = [NSBlockOperation blockOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
            NSLog(@"1---%@", [NSThread currentThread]); // 打印当前线程
        }
    }];
    NSBlockOperation *op2 = [NSBlockOperation blockOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
            NSLog(@"2---%@", [NSThread currentThread]); // 打印当前线程
        }
    }];

    // 3.添加依赖
    [op2 addDependency:op1]; // 让op2 依赖于 op1，则先执行op1，在执行op2

    // 4.添加操作到队列中
    [queue addOperation:op1];
    [queue addOperation:op2];
}


```
####3.2.5 实战例子

在 iOS 开发过程中，我们一般在主线程里边进行 UI 刷新，例如：点击、滚动、拖拽等事件。我们通常把一些耗时的操作放在其他线程，比如说图片下载、文件上传等耗时操作。而当我们有时候在其他线程完成了耗时操作时，需要回到主线程.

```
/**
 * 线程间通信
 */
- (void)communication {

    // 1.创建队列
    NSOperationQueue *queue = [[NSOperationQueue alloc]init];

    // 2.添加操作
    [queue addOperationWithBlock:^{
        // 异步进行耗时操作
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
            NSLog(@"1---%@", [NSThread currentThread]); // 打印当前线程
        }

        // 回到主线程
        [[NSOperationQueue mainQueue] addOperationWithBlock:^{
            // 进行一些 UI 刷新等操作
            for (int i = 0; i < 2; i++) {
                [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
                NSLog(@"2---%@", [NSThread currentThread]); // 打印当前线程
            }
        }];
    }];
}

```


## 4 三种技术的对比

###4.1 NSThread:

- 优点：NSThread 比其他两个轻量级，使用简单
- 缺点：需要自己管理线程的生命周期、线程同步、加锁、睡眠以及唤醒等。线程同步对数据的加锁会有一定的系统开销

###4.2 GCD

- 优点: GCD可用于多核的并行运算,它会自动利用更多的CPU内（比如双核、四核。除此之外，它会自动管理线程的生命周期（创建线程、调度任务、销毁线程)。程序员只需要告诉 GCD 想要执行什么任务，不需要编写任何线程管理代码

- 缺点: 从异步操作之间的事务性，顺序行，依赖关系。GCD需要自己写更多的代码来实现，而NSOperationQueue已经内建了这些支持

###4.3 NSOperation 
- 优点: NSOperation基于GCD异步队列进行封装，比GCD更加简洁易用。
- 缺点: GCD更接近底层，而NSOperationQueue则更高级抽象，所以GCD在追求性能的底层操作来说，是速度最快的。

###4.4 总结

> GCD是比较底层的封装，我们知道较低层的代码一般性能都是比较高的，相对于NSOperationQueue。所以追求性能，而功能够用的话就可以考虑使用GCD。如果异步操作的过程需要更多的用户交互和被UI显示出来，NSOperationQueue会是一个好选择。如果任务之间没有什么依赖关系，而是需要更高的并发能力，GCD则更有优势。
高德纳的教诲：“在大概97%的时间里，我们应该忘记微小的性能提升。过早优化是万恶之源。”只有Instruments显示有真正的性能提升时才有必要用低级的GCD。