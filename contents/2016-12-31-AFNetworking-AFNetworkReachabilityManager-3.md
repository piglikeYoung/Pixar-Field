
## 前言
最近公司项目赶进度，博客更新的不勤，AFNetworking源码一直也没有时间看。今天看一下 AFNetworkReachabilityManager 这个网络监听类。

AFNetworkReachabilityManager 是对 **SystemConfiguration** 网络模块的封装，苹果其实也有个监听网络状态的开源项目 [Reachability](https://developer.apple.com/library/content/samplecode/Reachability/Introduction/Intro.html)，不过它们的实现都是类似的。

## AFNetworkReachabilityManager

AFURLSessionManager 对网络状态的监控是由 AFNetworkReachabilityManager 来负责的，它仅仅是持有一个 AFNetworkReachabilityManager 的对象。

AFNetworkReachabilityManager 可以用来做一些网络判断，比如一个直播的APP，非常耗费流量，有时候用户在不知不觉中使用了3G/4G来看，这样用户估计会把这个APP给删了，那么在用户点击某个直播时候，提示用户一下，他正在使用3G/4G，或者有个开关直接来设置只允许在WiFi下看直播。

AFNetworkReachabilityManager 的使用很简单：

```objc
// 1.初始化AFNetworkReachabilityManager，获取单例
AFNetworkReachabilityManager *manager = [AFNetworkReachabilityManager sharedManager];
// 2.设置networkReachabilityStatusBlock，根据不同网络状态，用户自定义处理方式
[manager setReachabilityStatusChangeBlock:^(AFNetworkReachabilityStatus status) {
    NSLog(@"network status '%@'", AFStringFromNetworkReachabilityStatus(status));
}];
// 3.启动网络监听
[manager startMonitoring];
```

## 初始化 AFNetworkReachabilityManager

AFNetworkReachabilityManager 的初始化可以参考这张图，一步一步看它的调用就可以知道：

![Snip20170125_3](http://p44bkxib3.bkt.clouddn.com/Snip20170125_3.png)

其中有个参数 **SCNetworkReachabilityRef** 是监听网络的句柄，它在两个初始化方法中会被创建：

```objc
+ (instancetype)managerForDomain:(NSString *)domain {
    SCNetworkReachabilityRef reachability = SCNetworkReachabilityCreateWithName(kCFAllocatorDefault, [domain UTF8String]);

    AFNetworkReachabilityManager *manager = [[self alloc] initWithReachability:reachability];
    
    CFRelease(reachability);

    return manager;
}

+ (instancetype)managerForAddress:(const void *)address {
    SCNetworkReachabilityRef reachability = SCNetworkReachabilityCreateWithAddress(kCFAllocatorDefault, (const struct sockaddr *)address);
    AFNetworkReachabilityManager *manager = [[self alloc] initWithReachability:reachability];

    CFRelease(reachability);
    
    return manager;
}
```

1. 这两个方法一个通过**域名**，另一个通过**sockaddr_in6**生成一个 SCNetworkReachabilityRef
2. 调用**- [AFNetworkReachabilityManager initWithReachability:]**方法把生成的SCNetworkReachabilityRef 引用传递给 **networkReachability**
3. 设置默认的网络状态 networkReachabilityStatus

```objc
- (instancetype)initWithReachability:(SCNetworkReachabilityRef)reachability {
    self = [super init];
    if (!self) {
        return nil;
    }

    _networkReachability = CFRetain(reachability);
    self.networkReachabilityStatus = AFNetworkReachabilityStatusUnknown;

    return self;
}
```

## 启动网络监听

初始化完成 AFNetworkReachabilityManager 后，可以开始监听网络状态。

```objc
- (void)startMonitoring {
    // 先停止之前的监听
    [self stopMonitoring];
    // networkReachability表示的是需要检测的网络地址的句柄
    if (!self.networkReachability) {
        return;
    }

    __weak __typeof(self)weakSelf = self;
    // networkReachabilityStatusBlock 就是设置的回调block
    // 创建每次网络状态改变的回调
    AFNetworkReachabilityStatusBlock callback = ^(AFNetworkReachabilityStatus status) {
        __strong __typeof(weakSelf)strongSelf = weakSelf;
        // 每次回调被调用时重新设置 networkReachabilityStatus
        strongSelf.networkReachabilityStatus = status;
        // 调用 networkReachabilityStatusBlock
        if (strongSelf.networkReachabilityStatusBlock) {
            strongSelf.networkReachabilityStatusBlock(status);
        }

    };
    
    // 创建网络状态结构体，之后会说明
    SCNetworkReachabilityContext context = {0, (__bridge void *)callback, AFNetworkReachabilityRetainCallback, AFNetworkReachabilityReleaseCallback, NULL};
    // 当网络状态改变时，会调用传入的回调
    SCNetworkReachabilitySetCallback(self.networkReachability, AFNetworkReachabilityCallback, &context);
    // 在main RunLoop中以kCFRunLoopCommonModes形式处理self.networkingReachability
    SCNetworkReachabilityScheduleWithRunLoop(self.networkReachability, CFRunLoopGetMain(), kCFRunLoopCommonModes);

    // 创建子线程，在后台获取当前的 self.networkReachability 的网络状态，调用callback
  dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0),^{
        SCNetworkReachabilityFlags flags;
        if (SCNetworkReachabilityGetFlags(self.networkReachability, &flags)) {
            // AFPostReachabilityStatusChange函数就是先将flags转化为对应的AFNetworkReachabilityStatus变量，然后给我们的callback处理，后面会详解此函数
            AFPostReachabilityStatusChange(flags, callback);
        }
    });
}
```

### 创建一个 SCNetworkReachabilityContext

```objc
typedef struct {
	CFIndex		version;
	void *		__nullable info;
	const void	* __nonnull (* __nullable retain)(const void *info);
	void		(* __nullable release)(const void *info);
	CFStringRef	__nonnull (* __nullable copyDescription)(const void *info);
} SCNetworkReachabilityContext;

SCNetworkReachabilityContext context = {
    0,
    (__bridge void *)callback,
    AFNetworkReachabilityRetainCallback, 
    AFNetworkReachabilityReleaseCallback, 
    NULL
};
```

* callback 就是上一步创建的 AFNetworkReachabilityStatusBlock
* AFNetworkReachabilityRetainCallback 和 AFNetworkReachabilityReleaseCallback 都只是对 block 使用 `Block_copy` 和 `Block_release`，传入的 info 会以参数的形式在 AFNetworkReachabilityCallback 执行时传入




## 设置 networkReachabilityStatusBlock 以及回调
在 Main RunLoop 中对网络状态进行监控扣，每次网络状态改变，都会调用 **AFNetworkReachabilityCallback**

```objc
static void AFNetworkReachabilityCallback(SCNetworkReachabilityRef __unused target, SCNetworkReachabilityFlags flags, void *info) {
    AFPostReachabilityStatusChange(flags, (__bridge AFNetworkReachabilityStatusBlock)info);
}
```
这里会从 info 中取出之前存在 context 中的 **AFNetworkReachabilityStatusBlock**。

取出 block 后，调用 **AFPostReachabilityStatusChange** 方法，传入 block。

### AFPostReachabilityStatusChange
```objc
static void AFPostReachabilityStatusChange(SCNetworkReachabilityFlags flags, AFNetworkReachabilityStatusBlock block) {
    // 使用AFNetworkReachabilityStatusForFlags函数获取当前网络可达性，将flags转化为status，提供给下面block使用
    AFNetworkReachabilityStatus status = AFNetworkReachabilityStatusForFlags(flags);
    dispatch_async(dispatch_get_main_queue(), ^{
        if (block) {
            block(status);
        }
        // 发出一个网络状态变化的通知，对于用户，可以使用KVO来监听status的变化，用户可以在userInfo[AFNetworkingReachabilityNotificationStatusItem]中获取相应的status
        NSNotificationCenter *notificationCenter = [NSNotificationCenter defaultCenter];
        NSDictionary *userInfo = @{ AFNetworkingReachabilityNotificationStatusItem: @(status) };
        [notificationCenter postNotificationName:AFNetworkingReachabilityDidChangeNotification object:nil userInfo:userInfo];
    });
}
```

### AFNetworkReachabilityStatusForFlags
因为 flags 是 SCNetworkReachabilityFlags，它的不同位代表了不同的网络可达性状态，通过 flags 的位操作，获取当前的状态信息 AFNetworkReachabilityStatus。

```objc
static AFNetworkReachabilityStatus AFNetworkReachabilityStatusForFlags(SCNetworkReachabilityFlags flags) {
    // 该网络地址可达
    BOOL isReachable = ((flags & kSCNetworkReachabilityFlagsReachable) != 0);
    // 该网络地址虽然可达，但是需要先建立一个connection
    BOOL needsConnection = ((flags & kSCNetworkReachabilityFlagsConnectionRequired) != 0);
    // 该网络虽然也需要先建立一个connection，但是它是可以自动去connect的
    BOOL canConnectionAutomatically = (((flags & kSCNetworkReachabilityFlagsConnectionOnDemand ) != 0) || ((flags & kSCNetworkReachabilityFlagsConnectionOnTraffic) != 0));
    // 不需要用户交互，就可以connect上（用户交互一般指的是提供网络的账户和密码）
    BOOL canConnectWithoutUserInteraction = (canConnectionAutomatically && (flags & kSCNetworkReachabilityFlagsInterventionRequired) == 0);
    // 如果isReachable==YES，那么就需要判断是不是得先建立一个connection，如果需要，那就认为不可达，或者虽然需要先建立一个connection，但是不需要用户交互，那么认为也是可达的
    BOOL isNetworkReachable = (isReachable && (!needsConnection || canConnectWithoutUserInteraction));
    
    // AFNetworkReachabilityStatus 的四种状态根据字面意思很好理解
    AFNetworkReachabilityStatus status = AFNetworkReachabilityStatusUnknown;
    if (isNetworkReachable == NO) {
        status = AFNetworkReachabilityStatusNotReachable;
    }
#if	TARGET_OS_IPHONE
    else if ((flags & kSCNetworkReachabilityFlagsIsWWAN) != 0) {
        status = AFNetworkReachabilityStatusReachableViaWWAN;
    }
#endif
    else {
        status = AFNetworkReachabilityStatusReachableViaWiFi;
    }

    return status;
}

```

## 与 AFNetworking 关联
AFNetworkReachabilityManager 与 整个框架无太大耦合，在 AFURLSessionManager 中也只是持有一个 AFNetworkReachabilityManager 对象。

```objc
self.reachabilityManager = [AFNetworkReachabilityManager sharedManager];
```

搜索整个工程，除了 AFNetworkReachabilityManager.h/m 文件，只有在 AFURLSessionManager 中唯一引用了这个类。

## 总结
1. AFNetworkReachabilityManager 是对底层 SystemConfiguration 库网络状态获取的封装，对外提供了 Objective-C 的 API
2. AFNetworkReachabilityManager是个即插即用的模块



## 参考链接
* [AFNetworking](https://github.com/AFNetworking/AFNetworking)
* [AFNetworking源码阅读（六）](http://www.cnblogs.com/polobymulberry/p/5174298.html)
* [AFNetworkReachabilityManager 监控网络状态（四）](https://github.com/Draveness/iOS-Source-Code-Analyze/blob/master/contents/AFNetworking/AFNetworkReachabilityManager%20%E7%9B%91%E6%8E%A7%E7%BD%91%E7%BB%9C%E7%8A%B6%E6%80%81%EF%BC%88%E5%9B%9B%EF%BC%89.md)


