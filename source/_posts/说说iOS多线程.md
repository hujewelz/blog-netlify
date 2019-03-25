---
title: 说说iOS多线程
date: 2016-07-17 14:36:05
tags: iOS
category: iOS
---
线程和进程的区别在于,子进程和父进程有不同的代码和数据空间,而多个线程则共享数据空间,每个线程有自己的执行堆栈和程序计数器为其执行上下文.

<!--excerpt-->



在说多线程之前我们必须先弄懂两个概念：`进程` 和 `线程`

##### 进程
>进程(Process）是计算机中的程序关于某数据集合上的一次运行活动，是系统进行资源分配和调度的基本单位，是操作系统结构的基础。

简单来说，进程是指在系统中正在运行的一个应用程序，每一个程序都是一个进程，并且进程之间是独立的，每个进程均运行在其专用且受保护的内存空间内。

##### 线程
>线程是程序执行流的最小单元线程是程序中一个单一的顺序控制流程。是进程内一个相对独立的、可调度的执行单元，是系统独立调度和分派CPU的基本单位指运行中的程序的调度单位。

1个进程要想执行任务必须得有线程。线程中任务的执行是串行的，一个线程中的任务只能一个一个地按顺序执行，也就是说在同一时间内，1个线程只能执行1个任务。

线程和进程的区别在于,子进程和父进程有不同的代码和数据空间,而多个线程则共享数据空间,每个线程有自己的执行堆栈和程序计数器为其执行上下文.多线程主要是为了节约CPU时间,发挥利用,根据具体情况而定. 线程的运行中需要使用计算机的内存资源和CPU。
#### 多线程
>多线程（英语：multithreading），是指从软件或者硬件上实现多个线程并发执行的技术。具有多线程能力的计算机因有硬件支持而能够在同一时间执行多于一个线程，进而提升整体处理性能。

所谓多线程，就是在单个程序中同时运行多个线程完成不同的工作
注意，多线程是为了同步完成多项任务，不是为了提高运行效率，而是为了提高资源使用效率来提高系统的效率。线程是在同一时间需要完成多项任务的时候实现的。

### GCD
在说GCD之前我们得先弄懂4个比较容易混淆的术语：`同步`、`异步`、 `并发`、 `串行` 
同步和异步主要影响：能不能开启新的线程
* `同步`：只是在当前线程中执行任务，不具备开启新线程的能力
* `异步`：可以在新的线程中执行任务，具备开启新线程的能力
并行和串行主要影响：任务的执行方式
* `并发`：多个任务并发（同时）执行
* `串行`：一个任务执行完毕后，再执行下一个任务

GCD是最常用的管理并行代码和执行异步操作的Unix系统层的API。GCD构造和管理队列中的任务。首先，让我们看看队列是什么。
#### 队列是什么？
队列是按 `先进先出(FIFO)` 管理对象的数据结构。队列类似电影院的售票窗口，票的销售是谁先到谁先服务。在等待线前面的人先去买他们的门票，在其余的后抵达的人之前。
#### 调度队列
调度队列是一种简单的同步和异步任务的方法。任务以 `block` 的形式被提交到其中。系统有两种调度队列: `串行队列` 和 `并行队列` 。任务分配给这两个队列都是在单独的线程执行的，而不是在创建任务的线程上。换句话说，你创建任务(block)再提交到主线程的调度队列，但所有这些任务任务将运行在单独的线程而不是主线程。
#### 串行队列
当你创建一个串行队列，队列一次只能执行一个任务。同一队列中的任务将按着顺序依次执行，然而它们并不关心任务是不是在单独的线程，所以你可以通过使用多个串行队列来并行地执行任务。例如，你可以创建两个串行队列，每个队列一次只执行一个任务，不过多达两个任务仍可并行执行。
使用串行队列的优点：
1. 保证序列化访问共享资源，避免竞争条件。
2. 任务的执行顺序是可预测的。当你提交任务到一个串行调度队列，它们将按插入的顺序执行。
3. 你可以创建任意数量的串行队列。

#### 并行队列
并行队列可以并行执行多个任务。任务按添加到队列的顺序开始，但它们的执行会同时发生，不会相互等待。并行队列保证任务开始的顺序，但你不知道执行的顺序。

#### 使用队列
默认情况下，系统为每个应用提供了一个串行队列和四个并行队列。主调度队列是全局可用的串行队列，它在应用的主线程执行任务，主要用来更新UI,同时只有一个任务执行。
除了主队列，系统提供了4个并行队列，称之为全局调度队列。这些队列对于应用是全局的，区别只在于它们的优先级。使用`dispatch_get_global_queue`可以获取到一个全局队列，它有以下四个优先级：

* `DISPATCH_QUEUE_PRIORITY_HIGH`
* `DISPATCH_QUEUE_PRIORITY_DEFAULT`
* `DISPATCH_QUEUE_PRIORITY_LOW`
* `DISPATCH_QUEUE_PRIORITY_BACKGROUND`

以上优先级由高到低，所有你可以根据任务的优先级决定你使用的队列。不过，你也可以创建任意数量的串行或并行队列。

#### 任务
即操作，你想要干什么，说白了就是一段代码，在 GCD 中就是一个 `Block`，所以添加任务十分方便。任务有两种执行方式： `同步执行` 和 `异步执行`，他们之间的区别是在于会不会阻塞当前线程，直到 `Block` 中的任务执行完毕！

下面举几个栗子：
```
func GCD1() {
    print("task 1");
    dispatch_sync(dispatch_get_main_queue()) { //会阻塞当前线程，task 2不会执行
       print("task 2")
    }
}
```
运行会发现控制台打印出`task 1`，因为主线程被阻塞了，task2不会执行。

```
func GCD2() {
    print("task 1");
    
    let queue = dispatch_queue_create("come.jewelez.serial", DISPATCH_QUEUE_SERIAL);
    dispatch_async(queue) {
        print("task 2 \(NSThread.currentThread())")
        dispatch_sync(queue, { () -> Void in  //会阻塞当前线程，task 3不会执行
            print("task 3 \(NSThread.currentThread())")
        })
        print("task 4 \(NSThread.currentThread())")
    }
    print("task 5")
}
```
在这个例子中，你会发现控制台只会打印`task1, task5, task2`, 而`task3`和`task4`不会执行。首先我们创建了一个串行队列，然后以异步的方式提交了任务，所以`task2`可以执行，在任务中又以同步的方式向队列中提交了一个新的任务，由于是同步方式所以会阻塞当前线程，`task3`不会执行，因为是串行队列，当前线程又线程阻塞了，所以`task4`也不会执行。

```
func GCD3() {
    print("task 1")
    
    let queue = dispatch_queue_create("come.jewelez.serial", DISPATCH_QUEUE_SERIAL);
    dispatch_async(queue) {
        print("task 2 \(NSThread.currentThread())")
        
        dispatch_sync(dispatch_get_main_queue(), { () -> Void in  //异步遇到同步回主线程, task 3执行完后才会执行task 4
            print("task 3 \(NSThread.currentThread())")
        })
        
         print("task 4 \(NSThread.currentThread())")
    }
    
    print("task 5 \(NSThread.currentThread())")
}
```
这个就很容易理解了，不过要注意的一点是异步遇到同步回主线程, `task 3`执行完后才会执行`task 4`，控制台打印如下:
```
task 1
task 2 <NSThread: 0x7fe290e05d30>{number = 2, name = (null)}
task 5 <NSThread: 0x7fe290c045b0>{number = 1, name = main}
task 3 <NSThread: 0x7fe290c045b0>{number = 1, name = main}
task 4 <NSThread: 0x7fe290e05d30>{number = 2, name = (null)}
```
来看最后一个例子：
```
func GCD4() {
    print("task 1")
    
    let queue = dispatch_queue_create("come.jewelez.serial", DISPATCH_QUEUE_SERIAL)
    dispatch_async(queue) {
        print("task 2 \(NSThread.currentThread())")
        
        dispatch_async(dispatch_get_main_queue(), { () -> Void in
           
            for i in 0..<1000 {
                print("i: \(i)")
            }
            
            print("task 3 \(NSThread.currentThread())")
        })
        
        print("task 4 \(NSThread.currentThread())")
    }
    print("task 5 \(NSThread.currentThread())")
}
```
这个例子与上个不同在于，主线程中的任务也是以异步的方式执行的，所以`task 4`不用等到`task 3`执行完才执行。

#### dispatch_group
dispatch_group是用于监视一任务（Block）的机制。例如，当我们向一个队列里添加了多个任务，当队列中的所有任务执行完成后，我们需要做某种操作，这个时候就可以使用`dispatch_group`：
```
dispatch_queue_t queue = dispatch_queue_create("com.hujewelz.test", DISPATCH_QUEUE_CONCURRENT);
dispatch_group_t group = dispatch_group_create();
  
__block NSString *result1 = nil, *result2 = nil;
dispatch_group_async(group, queue, ^{
    NSLog(@"任务1");
    result1 = @"result 1";
});
  
dispatch_group_async(group, queue, ^{
    NSLog(@"任务2");
    result2 = @"result 2";
});
  
dispatch_group_notify(group, queue, ^{
    NSLog(@"notify--result: %@-%@", result1, result2);
});
```
运行结果：
```
2016-07-17 17:27:32.536 多线程[16626:593137] 任务1
2016-07-17 17:27:32.536 多线程[16626:593139] 任务2
2016-07-17 17:27:32.537 多线程[16626:593139] notify--result: result 1-result 2
```

### NSOperationQueue
NSOperationQueue 有两种不同类型的队列：主队列和自定义队列。主队列运行在主线程之上，而自定义队列在后台执行。在两种类型中，这些队列所处理的任务都使用 NSOperation 的子类来表述。
```
NSOperationQueue *mainQueue = [NSOperationQueue mainQueue];  //主队列
NSOperationQueue *queue = [[NSOperationQueue alloc] init]; //自定义队列
NSBlockOperation *operation = [NSBlockOperation blockOperationWithBlock:^{
                //任务执行
            }];
[queue addOperation:operation];
```
我们可以通过设置 maxConcurrentOperationCount 属性来控制并发任务的数量，当设置为 1 时， 那么它就是一个串行队列。主对列默认是串行队列，这一点和 dispatch_queue_t 是相似的。

### NSOperation
你可以使用系统提供的一些现成的 NSOperation 的子类， 如 `NSBlockOperation`、 `NSInvocationOperation`。

#### NSInvocationOperation
NSOperation的子类NSInvocationOperation为我们提供了一套简单的多线程编程方法：
```
  NSInvocationOperation *invo = [[NSInvocationOperation alloc]initWithTarget:self
                                                                     selector:@selector(handleInvocation)
                                                                       object:nil];
  [invo start];
```
调用 `start`方法，就会马上执行封装好的操作，也就是会调用`self`的`handleInvocation`方法.
> 注意：默认情况下，调用了start方法后并不会开一条新线程去执行操作，而是在当前线程同步执行操作。只有将operation放到一个NSOperationQueue中，才会异步执行操作。

#### NSBlockOperation
##### 1. 同步执行一个操作
```
NSBlockOperation *blckOp = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"执行一个新操作: %@", [NSThread currentThread]);
}];
[blckOp start];
```
输出结果为：
```
执行一个新操作: <NSThread: 0x60800007a940>{number = 1, name = main}
```
从结果可以看出，初始化一个NSBlockOperation对象后，调用`start`方法，
发现还是在当前线程同步执行操作，并没有异步执行。
##### 2. 并发执行多个操作
```
NSBlockOperation *blckOp = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"执行一个新操作: %@", [NSThread currentThread]);
}];

[blckOp addExecutionBlock:^{
    NSLog(@"又执行一个新操作 1: %@", [NSThread currentThread]);
}];
  
[blckOp addExecutionBlock:^{
    NSLog(@"又执行一个新操作 2: %@", [NSThread currentThread]);
}];
  
[blckOp addExecutionBlock:^{
    NSLog(@"又执行一个新操作 3: %@", [NSThread currentThread]);
}];

[blckOp start];
```
输出结果为：
```
又执行一个新操作 1: <NSThread: 0x608000079280>{number = 3, name = (null)}
执行一个新操作: <NSThread: 0x60000007c540>{number = 1, name = main}
又执行一个新操作 2: <NSThread: 0x60800007eac0>{number = 4, name = (null)}
又执行一个新操作 3: <NSThread: 0x60800007ecc0>{number = 5, name = (null)}
```
从结果可以看出，当我们通过 `addExecutionBlock:` 方法添加了新的操作后，就会并发地执行这些操作，也就是会在不同线程中执行。
> 结论：只要NSBlockOperation封装的操作数 > 1，就会异步执行操作。

#### 创建自己的Operation
你也可以实现自己的子类， 通过重写 `main` 或者 `start` 方法 来定义自己的 operation 。
使用 `main` 方法非常简单，开发者不需要管理一些状态属性（例如 `isExecuting` 和 `isFinished`），当 `main` 方法返回的时候，这个 operation 就结束了。这种方式使用起来非常简单，但是灵活性相对重写 `start` 来说要少一些， 因为`main`方法执行完就认为operation结束了，所以一般可以用来执行同步任务。
如果你希望拥有更多的控制权，或者想在一个操作中可以执行异步任务，那么就重写 `start` 方法, 但是注意：这种情况下，你必须手动管理操作的状态， 只有当发送 `isFinished` 的 KVO 消息时，才认为是 operation 结束.
```
@implementation YourOperation
- (void)start
{
  self.isExecuting = YES;
    // 任务代码 ...
}
- (void)finish //异步回调
{
  self.isExecuting = NO;
  self.isFinished = YES;
}
@end
```
当实现了 `start` 方法时，默认会执行 `start` 方法，而不执行 `main` 方法
为了让操作队列能够捕获到操作的改变，需要将状态的属性以配合 KVO 的方式进行实现。如果你不使用它们默认的 `setter` 来进行设置的话，你就需要在合适的时候发送合适的 KVO 消息。
需要手动管理的状态有：
* `isExecuting` 代表任务正在执行中
* `isFinished` 代表任务已经执行完成
* `isCancelled` 代表任务已经取消执行

手动的发送 KVO 消息， 通知状态更改如下 ：
```
[self willChangeValueForKey:@"isCancelled"];
_isCancelled = YES;
[self didChangeValueForKey:@"isCancelled"];
```
为了能使用操作队列所提供的取消功能，你需要在长时间操作中时不时地检查 `isCancelled ` 属性。
```
- (void)main {
    // 新建一个自动释放池，如果是异步执行操作，那么将无法访问到主线程的自动释放池
    @autoreleasepool {
        // 如果已经取消，释放资源，并返回
        if (self.isCancelled) {
            [self reset];
            return;
        }
        
        NSURLSessionConfiguration *configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
        self.session = [NSURLSession sessionWithConfiguration:configuration];
        
        NSURLRequest *repuest = [NSURLRequest requestWithURL:_url];
        __weak __typeof(self) wself = self;
        NSURLSessionDataTask *task = [_session dataTaskWithRequest:repuest completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
            __strong __typeof(self) sself = wself;
            if (!sself.completedBlock) {
                return ;
            }
            if (data == nil) {
                sself.completedBlock(nil, nil, error);
                return ;
            }
            // 如果已经取消，释放资源，并返回
            if (sself.isCancelled) {
                [sself reset];
                return;
            }
            
            UIImage *image = [UIImage hu_imageFromData:data];
            [[NSOperationQueue mainQueue] addOperationWithBlock:^{
                sself.completedBlock(image, data, nil);
            }];   
        }];
        [task resume];
    }
}

```


