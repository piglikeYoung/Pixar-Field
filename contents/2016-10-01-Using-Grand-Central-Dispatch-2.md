
## 前言
前一篇文章（[链接](/2016-09-25-Using-Grand-Central-Dispatch-1.md)）已经简单的介绍GCD的简单使用，这篇文章继续使用GCD。

## dispatch_async
GCD中有2个异步的API：

```objc
void dispatch_async(dispatch_queue_t queue, dispatch_block_t block);
void dispatch_async_f(dispatch_queue_t queue, void *context, dispatch_function_t work);
```

这两个API都是将一个任务提交到queue中，提交之后立刻返回，不等待任务执行完成。系统会对queue做retain操作，任务执行完成后，queue才被release。两者的区别在于`dispatch_async`接受block作为参数，`dispatch_async_f`接受函数。

`dispatch_async_f`中，**context**作为第一个参数传给**work**函数。如果**work**不需要参数，**context**可以传入**NULL**。**work**参数不能传入**NULL**，否则会发生不可预料的事。

> 使用GCD的时候，block会被copy，block会在执行完之后release，而且block是被系统持有的，所以不用担心循环引用的问题，**block里面的self不需要weak**。

`async`是异步的意思，至于怎么异步，看下面的例子：
```objc
- (void)dispatchAsync {
    NSLog(@"1");
    dispatch_async(dispatch_get_main_queue(), ^{
        // do Something
        NSLog(@"2");
    });
    NSLog(@"3");// 1，3，2
}
```
因为block是异步的，打印不是按照1，2，3的顺序打印的，简单点说，就是block里面的任务添加到队列后，GCD立刻返回，不阻塞线程，不需要等待任务执行完成。

就像上面的例子，先打印 ”1“ ，block添加主队列里，添加完成立刻返回，然后打印 ”3“，等主队列里面前面的任务执行完成再打印 ”2“。


## dispatch_sync

和异步一样，GCD同步也有两个API：

```objc
void dispatch_sync(dispatch_queue_t queue, dispatch_block_t block);
void dispatch_sync_f(dispatch_queue_t queue, void *context, dispatch_function_t work);
```
参数内容和`dispatch_async`一样，不同的是任务加入queue之后不会立刻返回，阻塞线程，等待任务执行完成后再返回。


```objc
- (void)dispatchSync {
    NSLog(@"1");
    dispatch_sync(dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0), ^{
        // do Something
        NSLog(@"2");
    });
    NSLog(@"3");// 1，2，3
}
```

从打印可以看出，打印是按照顺序来的
1. 先打印 ”1“ 
2. 然后dispatch_sync会将block提交到global queue中，等待block的执行
3. 全局队列中block前面的任务执行完成后，block执行
4. block执行完成后，dispatch_sync返回
5. dispatch_sync之后的代码执行，打印 ”3“

从示例代码中可以看出我在全局队列做同步操作的，没有在主队列里面做，是因为dispatch_sync需要等待block被执行，非常容易发生死锁，特别是在串行队列里面，而且主队列就是串行队列。


```objc
- (void)dispatchSyncDeadLock {
    NSLog(@"1");
    dispatch_sync(dispatch_get_main_queue(), ^{ // Dead Lock
        // do Something
        NSLog(@"2");
    });
    NSLog(@"3");
}
```

分析一下死锁怎么发生的：
1. 任务是放到主队列（串行队列）里面，串行队列任务是一个一个执行的
2. dispatch_sync需要等待block执行完成发返回，block需要等待前面的任务执行完成，也就是dispatch_sync执行完成。两者相互等待，永远不会执行完成，造成了死锁。

从中看出死锁是：`使用了dispatch_sync将任务加入串行队列`

如果是并行队列，则不会造成死锁。

## dispatch_after
在开发中经常需要到延迟几秒再执行的代码的情况，GCD也有这样的API:

```objc
void dispatch_after(dispatch_time_t when, dispatch_queue_t queue, dispatch_block_t block);
void dispatch_after_f(dispatch_time_t when, dispatch_queue_t queue, void *context, dispatch_function_t work);
```
when表示时间，如果传入 `DISPATCH_TIME_NOW` 会等同于 `dispatch_async` ，另外不允许传入 `DISPATCH_TIME_FOREVER` ，这会永远阻塞线程。

```objc
- (void)dispatchAfter {
    NSLog(@"1");
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(3 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        NSLog(@"2");
    });
    NSLog(@"3");
}
```
需要注意的是，dispatch_after 函数并不是在指定时间后执行处理，而只是在指定时间追加处理到 Dispatch Queue。此源代码与在3秒后用 dispatch_async 函数追加 Block 到 Main Dispatch Queue 相同。

因为 Main Dispatch Queue 在主线程的 RunLoop 中执行，所以在比如每隔1/60秒执行的 RunLoop 中，Block 最快在3秒后执行，最慢在3秒+1/60秒后执行，并且在 Main Dispatch Queue 有大量处理追加或者主线程的处理本身有延迟时，这个时间会更长。

虽然在有严格时间的要求下使用时会出现问题，但在想大致延迟处理时，该函数是非常有效的。

例子中 `dispatch_time` 参数原型是这样的

```objc
dispatch_time_t dispatch_time ( dispatch_time_t when, int64_t delta );
```
第一个参数用 `DISPATCH_TIME_NOW`表示当前。第二个参数delta表示纳秒，1秒=1000000000纳秒，系统用一些宏来简化

```objc
 #define NSEC_PER_SEC 1000000000ull //每秒有多少纳秒
 #define USEC_PER_SEC 1000000ull    //每毫秒有多少纳秒
 #define NSEC_PER_USEC 1000ull      //每秒有多少毫秒
```

这样1秒可以这样写：

```objc
dispatch_time(DISPATCH_TIME_NOW, 1 * NSEC_PER_SEC);
dispatch_time(DISPATCH_TIME_NOW, 1000 * USEC_PER_SEC);
dispatch_time(DISPATCH_TIME_NOW, USEC_PER_SEC * NSEC_PER_USEC);
```

## dispatch_once
GCD中还有保证在应用程序中只执行一次指定处理的API。

```objc
- (void)dispatchOne {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        // do Something
    });
}
```
通常通过这种方式生成单例对象。

## dispatch_groups

在追加到 Dispatch Queue 中的多个处理全部结束后想执行结束处理，这种情况会经常出现。如果使用一个 Serial Dispatch Queue  时，只要将想执行的处理全部追加到该 Serial Dispatch Queue 中并在最后追加结束处理，即可实现。但是在使用 Concurrent Dispatch Queue 时或同时使用多个Dispatch Queue 时，代码将会变得复杂。

这是 Dispatch Group 将派上用场了。

```objc
dispatch_queue_t queue = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
dispatch_group_t group = dispatch_group_create();
    
dispatch_group_async(group, queue, ^{
   NSLog(@"1");
});
dispatch_group_async(group, queue, ^{
   NSLog(@"2");
});
dispatch_group_async(group, queue, ^{
   NSLog(@"3");
});
    
dispatch_group_notify(group, dispatch_get_main_queue(), ^{
   NSLog(@"done");
});
```
例子中追加了3个 Block 到 Global Dispatch Queue，这些 Block 执行完成完毕后，执行 Main Dispatch Queue 中的结束处理。

在监听多个异步任务的时候，用到了`dispatch_group_notify`函数，它是异步执行，不会阻塞线程，第一个参数指定为要监视的 Dispatch Group。在追加到该 Dispatch Group 的全部处理执行结束时，将第三个参数的 Block 追加到的第二个参数的 Dispatch Queue 中。在 dispatch_group_notify 函数中不管指定什么样的 Dispatch Queue，属于 Dispatch Group 的全部处理在追加指定的 Block 时都已执行结束。

在 GCD API 中还有一个监听多个异步任务的函数 `dispatch_group_wait` 会阻塞当前线程，等待所有任务都完成或等待超时。这里的等待是指一旦调用 dispatch_group_wait 函数，该函数就处于调用状态而不返回。即执行 dispatch_group_wait 函数的当前线程停止。在经过 dispatch_group_wait 函数中指定的时间或属于指定 Dispatch Group 的处理全部执行结束之前，执行该函数的线程停止。

```objc
dispatch_queue_t queue = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
dispatch_group_t group = dispatch_group_create();
    
dispatch_group_async(group, queue, ^{
   NSLog(@"1");
});
dispatch_group_async(group, queue, ^{
   NSLog(@"2");
});
dispatch_group_async(group, queue, ^{
   NSLog(@"3");
});
    
dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
```

dispatch_group_wait 函数的第二个参数指定为等待的时间（超时）。它是 dispatch_time_t 类型的值。例子中用了 **DISPATCH_TIME_FOREVER** ，意思是永久等待。只要属于 Dispatch Group 的处理 尚未执行结束，就会一直等待，中途不能取消。

如果指定等待间隔为1秒是应做如下处理：

```objc
dispatch_queue_t queue = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
dispatch_group_t group = dispatch_group_create();
    
dispatch_group_async(group, queue, ^{
   NSLog(@"1");
});
dispatch_group_async(group, queue, ^{
   NSLog(@"2");
});
dispatch_group_async(group, queue, ^{
   NSLog(@"3");
});
    
dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, 1 * NSEC_PER_SEC);
NSInteger result = dispatch_group_wait(group, time);
if (result == 0) {
   // 属于 Dispatch Group 的全部处理执行结束
} else {
   // 属于 Dispatch Group 的某个处理还在执行中
}
```
如果 dispatch_group_wait 函数的返回值不为 0 ，就意味着虽然经过了指定的时间，但属于 Dispatch Group 的某个处理还在执行中。如果返回值为 0 ，那么全部处理执行结束。如果使用 **DISPATCH_TIME_FOREVER** ，属于 Dispatch Group 的全部处理必定全部执行结束，因此返回值恒为 0 。

如果使用 **DISPATCH_TIME_NOW** ，则不用任何等待即可判定属于 Dispatch Group 的处理是否执行结束。

## dispatch_barrier_async
在访问数据库或文件时，如果使用 Serial Dispatch Queue 可以避免数据竞争的问题。
写入操作不能和其他的写入操作以及包含读取处理的其他某些处理并行执行。如果只是读取操作和读取操作并行执行，那么多个并行执行不会发生问题。

GCD 为我们提供非常简便的解决方法 `dispatch_barrier_async` 函数。

首先 dispatch_queue_create 函数生成 Concurrent Dispatch Queue，在 dispatch_async 中追加读取操作：

```objc
dispatch_queue_t queue = dispatch_queue_create("com.piglikeyoung.gcd.concurrentqueue", DISPATCH_QUEUE_CONCURRENT);

dispatch_async(queue, blk0_for_reading);
dispatch_async(queue, blk1_for_reading);
dispatch_async(queue, blk2_for_reading);
dispatch_async(queue, blk3_for_reading);
dispatch_async(queue, blk4_for_reading);
dispatch_async(queue, blk5_for_reading);
dispatch_async(queue, blk6_for_reading);
```
在 blk3_for_reading 处理和 blk4_for_reading 处理之间执行写入操作，并将写入的内容在 blk4_for_reading 处理以及之后的处理中获取。

```objc
dispatch_async(queue, blk0_for_reading);
dispatch_async(queue, blk1_for_reading);
dispatch_async(queue, blk2_for_reading);
dispatch_async(queue, blk3_for_reading);
dispatch_async(queue, blk_for_writing);
dispatch_async(queue, blk4_for_reading);
dispatch_async(queue, blk5_for_reading);
dispatch_async(queue, blk6_for_reading);
```
如果是简单把写入操作加入 Concurrent Dispatch Queue，那么读取到的数据有可能和预期不符，如果是多个读取和多个写入混合并发，有可能引起数据竞争的问题。

这时候使用 `dispatch_barrier_async` 函数，**dispatch_barrier_async** 函数会等待追加到 Concurrent Dispatch Queue 上的并行执行的处理全部结束后，再将指定的处理追加到该 Concurrent Dispatch Queue 中。然后在由 dispatch_barrier_async 函数追加的处理执行完毕后，Concurrent Dispatch Queue 才恢复一般的动作，追加到该 Concurrent Dispatch Queue 的处理又开始并行执行。

```objc
dispatch_async(queue, blk0_for_reading);
dispatch_async(queue, blk1_for_reading);
dispatch_async(queue, blk2_for_reading);
dispatch_async(queue, blk3_for_reading);
dispatch_barrier_async(queue, blk_for_writing);
dispatch_async(queue, blk4_for_reading);
dispatch_async(queue, blk5_for_reading);
dispatch_async(queue, blk6_for_reading);
```
dispatch_barrier_async 处理流程图如下：
![Snip20161005_1](http://p44bkxib3.bkt.clouddn.com/Snip20161005_1.png)


> 注意：dispatch_barrier_async 只在自己创建的队列上有这种作用，在全局并发队列和串行队列上，效果和 dispatch_sync 一样


## dispatch_apply
dispatch_apply 函数是 dispatch_sync 函数和 Dispatch Group 的关联 API。该函数按指定的次数将指定的 Block 追加到指定的 Dispatch Queue 中，并等待全部处理执行结束。

```objc
dispatch_queue_t queue = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
dispatch_apply(10, queue, ^(size_t index) {
   NSLog(@"%zd", index);
});
NSLog(@"done");
```

因为是在全局并发队列中执行处理，所以打印的顺序不定。但输出结果中最后的 done 肯定是在最后。因为 dispatch_apply 函数会等待全部处理执行结束。

第一个参数是重复次数，第二个参数是追加对象的 Dispatch Queue ，第三个参数是追加的处理。不同的是第三个参数是带参数的block，这是为了按第一个参数重复追加 Block 并区分各个 Block 而使用。

由于 dispatch_apply 函数也和 dispatch_sync 函数相同，会等待处理执行结束，**因此推荐在 dispatch_async 函数中非同步地执行  dispatch_apply 函数**，而且能够避免线程爆炸，因为GCD会管理并发。

## 参考资料
* [官方文档](https://developer.apple.com/reference/dispatch)
* Objective-C 高级编程 iOS与OS X多线程和内存管理


