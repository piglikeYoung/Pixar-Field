
## 前言
在多线程环境下，需要处理的一个很大的问题就是`资源竞争`。在项目中我也遇到了多线程操作同一个数据源的问题，经常会遇到意想不到的Crash，在iOS中有同步锁，互斥锁的方式解决这个问题。

一般情况下的锁有：`NSLock`, `@synchronized`, `pthread_mutex_t`等等，来保证线程的安全。

## 问题
使用锁有很大的问题：加锁和解锁的操作非常昂贵，对性能会有影响。

```objc
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    
NSMutableSet *items = [NSMutableSet set];
    
NSLock *lock = [[NSLock alloc] init];
    
uint64_t start = mach_absolute_time();
    
dispatch_apply(50, queue, ^(size_t index) {
   for (NSInteger i = 0; i<10000; i++) {
       [lock lock];
       [items addObject:@"hello"];
       [lock unlock];
   }
});
    
uint64_t stop = mach_absolute_time();
    
NSLog(@"%f", subtractTimes(stop, start));
```
上面这段代码在我的机器上输出是1.7+s的时间。

## 优化
实际上频繁的加解锁是不必要的，我们可以借用GCD提供的信号量来优化这种操作。在我们创建一个信号量时，如果将信号的个数设置为1，则可以实现类似于锁的功能。

```objc
NSMutableSet *items = [NSMutableSet set];
    
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

dispatch_semaphore_t itemLock = dispatch_semaphore_create(1);
    
uint64_t start = mach_absolute_time();
    
dispatch_apply(50, queue, ^(size_t index) {
   for (NSInteger i = 0; i<10000; i++) {
       dispatch_semaphore_wait(itemLock, DISPATCH_TIME_FOREVER);
       [items addObject:@"hello"];
       dispatch_semaphore_signal(itemLock);
   }
});
    
uint64_t stop = mach_absolute_time();
    
NSLog(@"%f", subtractTimes(stop, start));
```
优化后的代码在我的机器上输出是0.1+s的时间，执行时间上，两者相差10+倍。

### 简单介绍GCD semaphore
* `dispatch_semaphore_create` 创建一个信号量，参数就是信号个数
* `dispatch_semaphore_signal` 发送一个信号，会让信号总量+1
* `dispatch_semaphore_wait` 等待信号，当信号总量≤0的时候就会一直等待，当信号总量>0的时候，正常执行，并且信号总量-1。

通过这样的方式，我们就可以快速创建一个线程锁控制资源访问和并发。

## 参考链接
[南峰子_老驴#iOS知识小集](http://huati.weibo.com/k/iOS%E7%9F%A5%E8%AF%86%E5%B0%8F%E9%9B%86?from=501)

