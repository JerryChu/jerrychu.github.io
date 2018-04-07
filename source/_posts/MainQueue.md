---
title: iOS中的主线程（Main Thread）与主队列（Main Queue）
date: 2018-03-25 19:11:20
tags:
- iOS
---

iOS编程中，需要在主线程中进行操作时，我们经常会用到以下代码：

	dispatch_async(dispatch_get_main_queue(), ^{ 
	    // TODO 0: 	
    });

仔细观察这部分代码， *dispatch\_get\_main_queue()* 实际获取的是 __主队列（Main Queue）__ 。我们看*dispatch\_get\_main_queue()* 的官方文档：

> Returns the main queue. This queue is created automatically on behalf of the main thread before main() is called.
               

而当我们需要判断当前线程是不是 **主线程（Main Thread）** 时，会这样写：
	
	if ([NSThread isMainThread]) {
		// TODO 1:
	}
	
那么我们会想到以下几个问题： 

**1. 主线程和主队列到底有什么关系？**  
**2. 为什么通过 *dispatch\_get\_main_queue()* 就可以确保在代码在主线程执行了？**   
**3. 主线程可以执行非主队列里的任务吗？** 

我们知道，主队列是系统自动为我们创建的一个串行队列，因此不用我们手动创建。在每个应用程序只有一个主队列，专门负责调度主线程里的任务，不允许开辟新的线程。也就是说，__在主队列里的任务，即使是异步的，最终也只能在主线程中运行__。因此，开头的第一段代码是可以保证在主线程中运行的。我们使用如下代码做测试：
	
	- (void)testMainThread {
	    NSLog(@"begin");
	    for (int i = 0 ; i < 10; i ++) {
	        dispatch_async(dispatch_get_main_queue(), ^{
	            NSLog(@"current Thread: %@; Task: %@",[NSThread currentThread], @(i));
	        });
	    }
	    NSLog(@"end");
    }
    
输出为：

	2016-08-28 21:22:18.384 Test[72358:1633765] begin
	2016-08-28 21:22:18.384 Test[72358:1633765] end
	2016-08-28 21:22:18.388 Test[72358:1633765] isMainThread: 1; Task: 0
	2016-08-28 21:22:18.388 Test[72358:1633765] isMainThread: 1; Task: 1
	2016-08-28 21:22:18.388 Test[72358:1633765] isMainThread: 1; Task: 2
	2016-08-28 21:22:18.388 Test[72358:1633765] isMainThread: 1; Task: 3
	2016-08-28 21:22:18.389 Test[72358:1633765] isMainThread: 1; Task: 4
	
由于block内的任务是异步执行，主线程在将当前方法（_testMainThread_ ）执行完毕之后，才会去继续执行主队列里的任务。那么如果我们把异步任务换成同步的出现什么结果呢？

	- (void)testMainThread {
	    NSLog(@"begin");
	    for (int i = 0 ; i < 10; i ++) {
	        dispatch_sync(dispatch_get_main_queue(), ^{
	            NSLog(@"current Thread: %@; Task: %@",[NSThread currentThread], @(i));
	        });
	    }
	    NSLog(@"end");
    }

输出为：
	
	2016-08-28 21:26:29.918 Test[72386:1636378] begin

也就是说，主线层被阻塞了。这个也好理解，因为此时主队列里的任务是同步执行的，同步任务必需被立刻执行。但同时由于主线程里的_testMainThread_ 还没有执行完，主线程没法去处理主队列里的任务，导致程序死锁。也就是说**主线程空闲时才会调度队列中的任务在主线程执行, 如果当前主线程正在有任务执行，那么无论主队列中当前被添加了什么任务，都不会被调度。** 这也是我们平常写代码需要注意的。

主队列的任务一定在主线程中执行，那么我们再来看最后一个问题：_主线程可以执行非主队列里的任务吗？_   我们使用下面的代码做测试：
	
	 NSLog(@"isMainThread: %@", @([NSThread isMainThread]));
    dispatch_sync(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSLog(@"isMainThread: %@", @([NSThread isMainThread]));
    });
	
在主线程中同步执行一个后台队列（使用_dispatch_get_global_queue_() 方法获取系统创建的全局并发队列）的任务。运行输出为：
	
	2016-08-28 21:55:26.262 Test[72533:1649394] isMainThread: 1
	2016-08-28 21:55:26.263 Test[72533:1649394] isMainThread: 1
	
可以看到，这个后台队列的任务也是在主线程中执行的。下面是从 [http://blog.benjamin-encz.de 的一篇博客](http://blog.benjamin-encz.de/post/main-queue-vs-main-thread) 摘抄下来的一段说明：

>While doing some research for this post I found [a commit to libdispatch that ensures that blocks dispatched with dispatch\_sync are always executed on the current thread](https://libdispatch.macosforge.org/trac/changeset/156). This means if you use dispatch\_sync to dispatch a block from the main queue to a concurrent background queue, the code executing on the background queue will actually be executed on the main thread. While this might not be entirely intuitive, it makes sense: since the main queue needs to wait until the dispatched block completed, the main thread will be available to process blocks from queues other than the main queue.

所以，主线程是可以执行主队列之外其他队列的任务的。即使_[NSThread mainThread]_ 判断当前线程是主线程，也不能保证当前执行的任务是主队列的任务（系统并没有为我们提供一个判断是不是在主队列的API）。
