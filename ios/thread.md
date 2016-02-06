# iOS多线程GCD和NSOperation原理

>iPhone OS下的主线程的堆栈大小是1M，第二个线程开始就是512KB，并且该值不能通过编译器开关或线程API函数来更改。  
>只有主线程有直接修改UI的能力。

iOS有三种多线程编程的技术，分别是:

（一）NSThread  
（二）Cocoa NSOperation  
（三）GCD（全称：Grand Central Dispatch） 

##三种方式的优缺点介绍:
###1）NSThread  
优点：NSThread 比其他两个轻量级  
缺点：需要自己管理线程的生命周期，线程同步。线程同步对数据的加锁会有一定的系统开销  

###2）Cocoa  NSOperation  
不需要关心线程管理， 数据同步的事情，可以把精力放在自己需要执行的操作上。
Cocoa operation相关的类是NSOperation, NSOperationQueue.
NSOperation是个抽象类,使用它必须用它的子类，可以实现它或者使用它定义好的两个子类: NSInvocationOperation和NSBlockOperation.
创建NSOperation子类的对象，把对象添加到NSOperationQueue队列里执行。

###3) GCD
Grand Central dispatch(GCD)是Apple开发的一个多核编程的解决方案。在iOS4.0开始之后才能使用。GCD是一个替代NSThread,NSOperationQueue,NSInvocationOperation等技术的很高效强大的技术。

~~~cpp
////////////////////////////////////////
// GCD (Grand Central Dispatch) Thread
- (IBAction)GCDThreadButtonClick:(id)sender {

    // 主队列
    dispatch_queue_t queueMain = dispatch_get_main_queue();

    // 获得创建串行队列
    dispatch_queue_t queue1 = dispatch_queue_create("tk.bourne.testQueue1", NULL);
    dispatch_queue_t queue2 = dispatch_queue_create("tk.bourne.testQueue2", DISPATCH_QUEUE_SERIAL);
    // 创建并行队列
    dispatch_queue_t queue3 = dispatch_queue_create("tk.bourne.testQueue3", DISPATCH_QUEUE_CONCURRENT);
    // 获得全局并行队列
    dispatch_queue_t queueGlobe = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

    // 创建同步任务
/* 主线程的同步任务不能放在主线程里执行，也不能放在其他由主线程启动的同步线程中执行，
 只能放在主线程异步启动的线程中，否则会卡死主线程。任何线程队列都不能和自己同步！
 比如放到下面 dispatch_async(queue1 里面：
    dispatch_sync(queueMain, ^{
        NSLog(@"sync queueMain:[%@]", [NSThread currentThread]);
    });
*/
    dispatch_sync(queue1, ^{
        NSLog(@"sync queue1:[%@]", [NSThread currentThread]);
    });
    dispatch_sync(queue2, ^{
        NSLog(@"sync queue2:[%@]", [NSThread currentThread]);
    });
    dispatch_sync(queue3, ^{
        NSLog(@"sync queue3:[%@]", [NSThread currentThread]);
    });
    dispatch_sync(queueGlobe, ^{
        NSLog(@"sync queueGlobe:[%@]", [NSThread currentThread]);
    });

    // 创建异步任务
    dispatch_async(queueMain, ^{
        NSLog(@"run queueMain:[%@]", [NSThread currentThread]);
    });
    dispatch_async(queue1, ^{
        NSLog(@"run queue1:[%@]", [NSThread currentThread]);
    });
    dispatch_async(queue2, ^{
        NSLog(@"run queue2:[%@]", [NSThread currentThread]);
    });
    dispatch_async(queue3, ^{
        NSLog(@"run queue3:[%@]", [NSThread currentThread]);
    });
    dispatch_async(queueGlobe, ^{
        NSLog(@"run queueGlobe:[%@]", [NSThread currentThread]);
    });

    //1.创建队列组
    dispatch_group_t group = dispatch_group_create();

    //3.多次使用队列组的方法执行任务, 只有异步方法
    //3.1.执行3次循环
    dispatch_group_async(group, queueGlobe, ^{
        for (NSInteger i = 0; i < 3; i++) {
            NSLog(@"group-01-%ld - %@", (long)i, [NSThread currentThread]);
        }
    });

    //3.2.主队列执行8次循环
    dispatch_group_async(group, dispatch_get_main_queue(), ^{
        for (NSInteger i = 0; i < 8; i++) {
            NSLog(@"group-02-%ld - %@", (long)i, [NSThread currentThread]);
        }
    });

    //3.3.执行5次循环
    dispatch_group_async(group, queueGlobe, ^{
        for (NSInteger i = 0; i < 5; i++) {
            NSLog(@"group-03-%ld - %@", (long)i, [NSThread currentThread]);
        }
    });

    //4.都完成后会自动通知
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        NSLog(@"完成 - %@", [NSThread currentThread]);
    });

/* 这两个有空可以看一下：
    func dispatch_barrier_async(_ queue: dispatch_queue_t, _ block: dispatch_block_t):
    func dispatch_barrier_sync(_ queue: dispatch_queue_t, _ block: dispatch_block_t):
*/
}

////////////////////////////////////////
// NSOperation Thread - 对GCD的封装
- (void)onNSOperation:(void*)data {

    NSLog(@"run NSOperation:[%@]", [NSThread currentThread]);
}

- (IBAction)nsoperationButtonClick:(id)sender {

    // 默认会在当前线程执行
    ////////////////////////////////////////
    //1.创建NSInvocationOperation对象
    NSInvocationOperation *operation = [[NSInvocationOperation alloc] initWithTarget:self selector:@selector(onNSOperation:) object:nil];
    //2.开始执行
    [operation start];

    //1.创建NSBlockOperation对象
    NSBlockOperation *operation2 = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"run NSOperation2:[%@]", [NSThread currentThread]);
    }];
    //2.开始任务
    [operation2 start];

    ////////////////////////////////////////
    //1.创建NSBlockOperation对象
    NSBlockOperation *operation3 = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"run NSOperation3:[%@]", [NSThread currentThread]);
    }];
    //添加多个Block，它会 在主线程和其它的多个线程 并发执行 这些任务
    for (NSInteger i = 0; i < 5; i++) {
        [operation3 addExecutionBlock:^{
            NSLog(@"第%ld次：%@", i, [NSThread currentThread]);
        }];
    }
    //2.开始任务
    [operation3 start];

    ////////////////////////////////////////
    //1.创建一个其他队列，确保在新建block中执行，而不在主线程执行
    NSOperationQueue *queue = [[NSOperationQueue alloc] init];

    //2.创建NSBlockOperation对象
    NSBlockOperation *operation4 = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"run NSOperation4:[%@]", [NSThread currentThread]);
    }];

    //3.添加多个Block
    for (NSInteger i = 0; i < 5; i++) {
        [operation4 addExecutionBlock:^{
            NSLog(@"第%ld次：%@", i, [NSThread currentThread]);
        }];
    }

    //4.队列添加任务
    [queue addOperation:operation4];

    [NSThread sleepForTimeInterval:1.0];
    ////////////////////////////////////////
    // 添加依赖
    //1.任务一：下载图片
    NSBlockOperation *operation5 = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"下载图片 - %@", [NSThread currentThread]);
        [NSThread sleepForTimeInterval:1.0];
    }];

    //2.任务二：打水印
    NSBlockOperation *operation6 = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"打水印   - %@", [NSThread currentThread]);
        [NSThread sleepForTimeInterval:1.0];
    }];

    //3.任务三：上传图片
    NSBlockOperation *operation7 = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"上传图片 - %@", [NSThread currentThread]);
        [NSThread sleepForTimeInterval:1.0];
    }];

    //4.设置依赖
    [operation6 addDependency:operation5];      //任务二依赖任务一
    [operation7 addDependency:operation6];      //任务三依赖任务二

    //5.创建队列并加入任务
    NSOperationQueue *queue2 = [[NSOperationQueue alloc] init];
    [queue2 addOperations:@[operation7, operation6, operation5] waitUntilFinished:NO];
}
~~~
