
## 前言
GCD的全称是Grand Central Dispatch，可译为“牛逼的中枢调度器”，属于系统级的的线程管理，会自动管理线程的生命周期（创建线程、调度任务、销毁线程），现在GCD这块已经开源了（[开源地址](http://libdispatch.macosforge.org)）。

## GCD概要
* GCD的队列可以分为2大类型：并发队列（Concurrent Dispatch Queue）和串行队列（Serial Dispatch Queue）
* GCD会自动将队列的任务取出放到对应的形成中执行，遵循队列的FIFO原则：先进先出，后进后出
* 有五中不同的队列：主队列main queue，运行在主线程，用于刷新UI。还有四个不同优先级的全局并发队列（High Priority Queue，Default Priority Queue，Low Priority Queue，Background Queue）

## 几个容易混淆的术语

同步和异步决定了要不要开启新的线程

* 同步：在当前线程中执行任务，不具备开启新线程的能力
* 异步：在新的线程中执行任务，具备开启新线程的能力

并发和串行决定了任务的执行方式

* 并发：多个任务并发（同时）执行
* 串行：一个任务执行完毕后，再执行下一个任务

## 基本概念

* 系统标准的两个队列

```objc
// 主线程中的唯一队列，一种特殊的串行队列，放在主队列的任务，都会放到主线程中执行
dispatch_get_main_queue
// 包含四种不同优先级的并发队列，由系统创建，供整个应用使用。
dispatch_get_global_queue
```

* 自定义队列

```objc
// 串行队列
dispatch_queue_create("com.piglikeyoung.serialqueue", DISPATCH_QUEUE_SERIAL)
// 并行队列
dispatch_queue_create("com.piglikeyoung.concurrentqueue", DISPATCH_QUEUE_CONCURRENT)
```

* 同步异步线程创建

```objc
// 同步线程
dispatch_sync(..., ^(block))
// 异步线程
dispatch_async(..., ^(block))
```

## 队列类型

* Serial Dispatch Queue（串行队列）：要等待现在执行中的处理结束才执行下一个任务，同时只能执行1个任务。当创建多个Serial Queue时，它们各自是同步的，但Serial Queue之间是并发执行的。
* Concurrent Dispatch Queue（并发队列）：不需要等待现在执行中的处理结束，并行多个处理，并行执行的处理数量取决于当前系统的状态。所谓“并行执行”，就是使用多个线程同时执行处理。


```objc
// 主队列
dispatch_queue_t mainQueue = dispatch_get_main_queue();
// 全局并发队列
dispatch_queue_t globalQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0); 
```

## dispatch_queue_create
通过 dispatch_queue_create 函数可以生成 Dispatch Queue，函数有两个参数，第一个是自定义队列名，第二个是队列类型，输入`NULL`或者`DISPATCH_QUEUE_SERIAL`表示**串行队列**，输入`DISPATCH_QUEUE_CONCURRENT`表示**并行队列**。

```objc
dispatch_queue_t queue = dispatch_queue_create("com.piglikeyoung.gcd.concurrentqueue", DISPATCH_QUEUE_CONCURRENT);
```

第二个参数还有第三种传值方式，参考下面说到的`dipatch_queue_attr_make_with_qos_class`

## dispatch_set_target_queue
dispatch_queue_create 函数生成的 Dispatch Queue 不管是 Serial Dispatch Queue 还是 Concurrent Dispatch Queue ，都是使用与默认优先级 Global Dispatch Queue 相同执行优先级的线程。而变更生成的 Dispatch Queue 的执行优先级需要使用 `dispatch_set_target_queue` 函数。

```objc
- (void)setTargetQueue {
    dispatch_queue_t mySerialDispatchQueue = dispatch_queue_create("com.piglikeyoung.gcd.MySerialDispatchQueue", NULL);
    dispatch_queue_t globalDispatchQueueBackground = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0);
    dispatch_set_target_queue(mySerialDispatchQueue, globalDispatchQueueBackground);
}
```

指定要变更执行优先级的 Dispatch Queue 为 dispatch_set_target_queue 函数的第一个参数，指定与要使用的执行优先级相同优先级的 Global Dispatch Queue 为第二个参数（目标）。第一个参数如果指定系统提供的 Main Dispatch Queue 和 Global Dispatch Queue 则不知道会出现什么状况，因此这些均不可指定。


dispatch_set_target_queue 不仅可以变更 Dispatch Queue 的执行优先级，还可以作成 Dispatch Queue 的执行阶层。如果在多个 Serial Dispatch Queue 中用 dispatch_set_target_queue 函数指定目标为某个 Serial Dispatch Queue ，那么原先本应并行执行的多个 Serial Dispatch Queue ，在目标 Serial Dispatch Queue 上只能同时执行一个处理。

```objc
- (void)dispatchSetTargetQueueDemo {
    dispatch_queue_t serialQueue = dispatch_queue_create("com.piglikeyoung.gcd.serialqueue", DISPATCH_QUEUE_SERIAL);
    dispatch_queue_t firstQueue = dispatch_queue_create("com.piglikeyoung.gcd.firstqueue", DISPATCH_QUEUE_SERIAL);
    dispatch_queue_t secondQueue = dispatch_queue_create("com.piglikeyoung.gcd.secondqueue", DISPATCH_QUEUE_CONCURRENT);
    
    dispatch_set_target_queue(firstQueue, serialQueue);
    dispatch_set_target_queue(secondQueue, serialQueue);
    
    dispatch_async(firstQueue, ^{
        NSLog(@"1");
        sleep(3);
    });
    dispatch_async(secondQueue, ^{
        NSLog(@"2");
        sleep(2);
    });
    dispatch_async(secondQueue, ^{
        NSLog(@"3");
        sleep(1);
    });
}
```

## dipatch_queue_attr_make_with_qos_class
iOS8之后苹果框架概念基础中将不再过分强调线程这个概念。新的 qualityOfService 属性替换了 ThreadPriority。

qualityOfService 主要用于 dispatch_queue_create函数。适用于OSX v10.10及以后或iOS v8.0及以后的版本。

自定义队列通过 dipatch_queue_attr_make_with_qos_class 来指定队列的执行优先级：

```objc
- (void)attrMakeWithQosClass{
	// 自定义属性
	dispatch_queue_attr_t attr = dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_SERIAL, QOS_CLASS_UTILITY, -1);
	dispatch_queue_t queue = dispatch_queue_create("com.piglikeyoung.gcd.qosqueue", attr);
}
```

dispatch_queue_attr_make_with_qos_class 参数有三个：

* 第一个参数**attr**：传入**DISPATCH_QUEUE_SERIAL**、**NULL**或者**DISPATCH_QUEUE_CONCURRENT**，表示串行或者并行
* 第二个参数**qos_class**：传入qos_class枚举，表示优先级级别
* 带三个参数**relative_priority**：相对于qos_class的相对优先级，qos_class用于区分大的优先级级别，relative_priority表示大级别下的小级别。`relative_priority必须大于QOS_MIN_RELATIVE_PRIORITY并且小于0，否则将返回NULL`。从GCD源码中可以查到QOS_MIN_RELATIVE_PRIORITY等于-15。

qos_class枚举定义了以下值：

* **QOS_CLASS_USER_INTERACTIVE**：最高优先级，交互级别。使用这个优先级会占用几乎所有的系统CUP和I/O带宽，仅限用于交互的UI操作，比如处理点击事件，绘制图像到屏幕上，动画等
* **QOS_CLASS_USER_INITIATED**：次高优先级，user initiated等级表示任务由UI发起异步执行。适用场景是需要及时结果同时又可以继续交互的时候。例如，如果用户打开email app马上查看邮件。
* **QOS_CLASS_DEFAULT**：默认优先级，当没有设置优先级的时候，线程默认优先级。一般情况下用的都是这个优先级。
* **QOS_CLASS_UTILITY**：普通优先级，utility等级表示需要长时间运行的任务，伴有用户可见进度指示器。经常会用来做计算，I/O，网络，持续的数据填充等任务。这个任务节能。例如，电子邮件应用程序可以被配置为每隔5分钟自动检查邮件。如果系统是非常有限的资源，而电子邮件检查被推迟几分钟这也是被允许的。
* **QOS_CLASS_BACKGROUND**：后台优先级，background等级表示用户不会察觉的任务，使用它来处理预加载，或者不需要用户交互和对时间不敏感的任务。比如email app可能使用它来执行索引搜索。
* **QOS_CLASS_UNSPECIFIED**：未知优先级，表示服务质量信息缺失。

和 ThreadPriority 的对比表格：

| Global queue | Corresponding QoS class | 说明 |
| :------------ | :--------------- | :----- |
| Main thread | NSQualityOfServiceUserInteractive | UI相关，交互等 |
| DISPATCH_QUEUE_PRIORITY_HIGH | QOS_CLASS_USER_INITIATED | 用于执行类似初始化等需要立即返回的事件 |
| DISPATCH_QUEUE_PRIORITY_DEFAULT | QOS_CLASS_DEFAULT | 当没有设置优先级的时候，线程默认优先级。一般情况下用的都是这个优先级 |
| DISPATCH_QUEUE_PRIORITY_LOW | QOS_CLASS_UTILITY | 主要用于不需要立即返回的任务，花费时间稍多比如下载，需要几秒或几分钟的 |
| DISPATCH_QUEUE_PRIORITY_BACKGROUND | QOS_CLASS_BACKGROUND | 用于用户几乎不感知的任务，在后台的操作可能需要好几分钟甚至几小时的 |


```objc
// 两种使用方式等价
dispatch_async(dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0), ^{
   // do Something
});
    
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
   // do Something
});
```

## 参考资料
* [官方文档](https://developer.apple.com/reference/dispatch)
* Objective-C 高级编程 iOS与OS X多线程和内存管理
* [细说GCD（Grand Central Dispatch）如何用](https://github.com/ming1016/study/wiki/%E7%BB%86%E8%AF%B4GCD%EF%BC%88Grand-Central-Dispatch%EF%BC%89%E5%A6%82%E4%BD%95%E7%94%A8)
* [那些开发者应该知道但又略显模糊的iOS 8 API](http://www.cocoachina.com/ios/20140630/8984.html)
* [小笨狼漫谈多线程：GCD(一)](http://www.jianshu.com/users/1f93e3b1f3da/latest_articles)











