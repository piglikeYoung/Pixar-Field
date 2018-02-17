
## 前言
GCD 的使用第三篇。前两篇链接：
* [使用GCD（一）](/2016-09-25-Using-Grand-Central-Dispatch-1.md)
* [使用GCD（二）](/2016-10-01-Using-Grand-Central-Dispatch-2.md)

## dispatch_suspend / dispatch_resume

dispatch_suspend 函数挂起指定的 Dispatch Queue。

```objc
dispatch_suspend(queue);
```

dispatch_resume 函数恢复指定的 Dispatch Queue。

```objc
dispatch_resume(queue);
```
这两个函数对已经执行的处理没有影响。挂起后，追加到 Dispatch Queue 中但没有执行的处理停止执行。恢复后，这些处理继续执行。

## Dispatch Semaphore
上一篇文章说到，并发执行更新数据处理时，会产生数据不一致的情况，有时APP会异常崩溃。虽然使用 Serial Dispatch Queue 和 dispatch_barrier_async 函数可避免这类问题，但是有必要进行更细粒度的排它处理。

比如这种情况：

```objc
dispatch_queue_t queue = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
NSMutableArray *array = [NSMutableArray array];
for (NSInteger i = 0; i < 100000; i++) {
   dispatch_async(queue, ^{
       [array addObject:@(i)];
   });
}
```
例子中的代码，并发大量的操作可变数组，执行后出现内存错误导致APP异常结束的概率非常高（大量的线程同时操作同一片内存）。这时候 Dispatch Semaphore 就派上用场了。

**Dispatch Semaphore** 是持有计数的信号，该计数是多线程编程中的计数类型信号。所谓信号，类似于过马路时常用的手旗。可以通过时举起手旗，不可通过时方式手旗。在 Dispatch Semaphore 中，使用计数类实现该功能。**计数为0时等待，计数为1或者大于1时，减去1而不等待**。

生成 Dispatch Semaphore：

```objc
dispatch_semaphore_t semphore = dispatch_semaphore_create(1);
```
参数表示计数的初始值。例子中计数值初始化为“1”。

**dispatch_semaphore_wait** 函数等待 Dispatch Semaphore 的计数值达到大于或者等于1 。

```objc
dispatch_semaphore_wait(semphore, DISPATCH_TIME_FOREVER);
```
当计数值大于等于1，或者在待机中计数值大于等于1时，对该计数进行减法并从 dispatch_semaphore_wait 函数返回。第二个参数与 dispatch_group_wait 函数等相同，是 dispatch_time_t 类型值。另外，dispatch_semaphore_wait 函数的返回值也和 dispatch_group_wait 相同，返回值参考[《使用GCD（二）》](http://piglikeyoung.com/2016/10/01/Using-Grand-Central-Dispatch-2/)的内容。

dispatch_semaphore_wait 函数做减法操作，而 **dispatch_semaphore_signal** 函数做加法操作，将 Dispatch Semaphore 的计数值加1 。

把前面的例子使用 Dispatch Semaphore 来处理：

```objc
/*
* 生成 Dispatch Semaphore
* Dispatch Semaphore 计数初始值设置为 “1”
* 保证可访问 NSMutableArray 类对象的线程，同时只有1个
*/
dispatch_semaphore_t semphore = dispatch_semaphore_create(1);
    
dispatch_queue_t queue = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
NSMutableArray *array = [NSMutableArray array];
for (NSInteger i = 0; i < 100000; i++) {
   dispatch_async(queue, ^{
       
       /*
        * 等待 Dispatch Semaphore
        * 一直等待，直到 Dispatch Semaphore 的计数值达到≥1
        */
       dispatch_semaphore_wait(semphore, DISPATCH_TIME_FOREVER);
       
       /*
        * 由于 Dispatch Semaphore 的计数值达到≥1
        * 所以将 Dispatch Semaphore 的计数值减去1，
        * dispatch_semaphore_wait 函数执行返回。
        * 
        * 即执行到这，Dispatch Semaphore 计数值恒为 “0”
        * 保证可访问 NSMutableArray 类对象的线程，同时只有1个
        * 可以安全更新
        */
       [array addObject:@(i)];
       
       /*
        * 排它处理结束
        * 所以通过 dispatch_semaphore_signal 将 Dispatch Semaphore 计数值加1
        * dispatch_semaphore_wait 函数执行返回。
        *
        * 如果有正在等待 Dispatch Semaphore 计数值增加的线程，加1操作结束后
        * 最先等待的线程先执行
        *
        */
       dispatch_semaphore_signal(semphore);
   });
}
```

> 这种变相给数据源上锁的方式相对比较节省性能，和传统的 NSLock 上锁的性能比较可以参考这篇文章[《GCD信号锁》](http://piglikeyoung.com/2016/09/11/GCD-signal-lock/)。


## Dispatch I/O
读取大文件时，将文件分割成合适大小，使用 Global Dispatch Queue 并发读取，速度会快上很多。这一功能 GCD 也提供了，使用到了 Dispatch I/O 和 Dispatch Data。

具体代码可以看 Apple System Log API 使用的源码（[Libc-763.11 gen/asl.c](https://github.com/Apple-FOSS-Mirror/Libc/blob/2ca2ae74647714acfc18674c3114b1a5d3325d7d/gen/asl.c)）

## 参考资料
* [官方文档](https://developer.apple.com/reference/dispatch)
* Objective-C 高级编程 iOS与OS X多线程和内存管理


