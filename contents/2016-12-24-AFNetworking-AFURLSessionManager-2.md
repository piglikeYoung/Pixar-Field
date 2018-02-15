
## 前言
上一章对 AFNetworking进行了概述，这一章重点对 AFN 的核心 **AFURLSessionManager**进行分析

## AFURLSessionManager属性

主要属性：

```objc
/**
 AFURLSessionManager管理的Session对象
 */
@property (readonly, nonatomic, strong) NSURLSession *session;

/**
 delegate的回调所在队列
 */
@property (readonly, nonatomic, strong) NSOperationQueue *operationQueue;

/**
 解析网络请求返回的数据，遵循AFURLResponseSerialization协议，并且不能为nil
 */
@property (nonatomic, strong) id <AFURLResponseSerialization> responseSerializer;

/**
 securityPolicy 是处理网络连接安全的对象，无特别的要求一般使用‘defaultPolicy’
 */
@property (nonatomic, strong) AFSecurityPolicy *securityPolicy;

/**
 reachabilityManager 是用于监控网络状态的对象，默认使用‘sharedManager’
 */
@property (readwrite, nonatomic, strong) AFNetworkReachabilityManager *reachabilityManager;

/**
 Session中正在运行的 Task ，包括 dataTask，uploadTask，downloadTask
 */
@property (readonly, nonatomic, strong) NSArray <NSURLSessionTask *> *tasks;

/**
 Session中正在运行的 dataTask
 */
@property (readonly, nonatomic, strong) NSArray <NSURLSessionDataTask *> *dataTasks;

/**
 Session中正在运行的 uploadTask
 */
@property (readonly, nonatomic, strong) NSArray <NSURLSessionUploadTask *> *uploadTasks;

/**
 Session中正在运行的 downloadTask
 */
@property (readonly, nonatomic, strong) NSArray <NSURLSessionDownloadTask *> *downloadTasks;

/**
 completionBlock的回调GCD队列，如果未主动设置，默认是 main queue
 */
@property (nonatomic, strong, nullable) dispatch_queue_t completionQueue;

/**
 completionBlock的回调Group-GCD队列，如果未主动设置，默认是 private dispatch group
 */
@property (nonatomic, strong, nullable) dispatch_group_t completionGroup;

///---------------------------------
/// @name Working Around System Bugs
///---------------------------------

/**
 当在后台session创建 upload task 返回‘nil’的时候，是否尝试重新创建 upload tasks，默认是NO

 @bug 这是 iOS 7.0的 bug，这个 bug 是当在后台session 创建 upload task 时，upload task 有时候会初始化为‘nil’。作为一种替代方法，如果这个属性设置为 YES ，AFN 会按照 Apple 推荐的方式创建 Task。

 @详见 https://github.com/AFNetworking/AFNetworking/issues/1675
 */
@property (nonatomic, assign) BOOL attemptsToRecreateUploadTasksForBackgroundSessions;
```

AFURLSessionManager 的属性不是很多，也不是很复杂，注释也非常详尽。

## 使用AFURLSessionManager完成网络请求

使用AFURLSessionManager完成网络获取网络数据，仅需要如下几个步骤：

```objc
// 1.初始化AFURLSessionManager对象
NSURLSessionConfiguration *configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
AFURLSessionManager *manager = [[AFURLSessionManager alloc] initWithSessionConfiguration:configuration];
    
// 初始请求
NSURLRequest *request = [[NSURLRequest alloc] initWithURL:[[NSURL alloc] initWithString:@"https://github.com"]];
    
// 2.获取AFURLSessionManager的task对象
NSURLSessionDataTask *dataTask = [manager dataTaskWithRequest:request uploadProgress:nil downloadProgress:nil completionHandler:^(NSURLResponse * _Nonnull response, id  _Nullable responseObject, NSError * _Nullable error) {
   if (error) {
       NSLog(@"Error: %@", error);
   } else {
       NSLog(@"Get Net data success!");
   }
}];
// 3.启动task
[dataTask resume];
```
除了初始化 NSURLRequest ，完成整个数据请求只需要的三步，👇逐步分析如何实现的

### 初始化AFURLSessionManager

调用这个方法来初始化 AFURLSessionManager
```objc
- (instancetype)initWithSessionConfiguration:(NSURLSessionConfiguration *)configuration
```
它只有一个 NSURLSessionConfiguration 参数。

实现的代码：

```objc
- (instancetype)initWithSessionConfiguration:(NSURLSessionConfiguration *)configuration {
    self = [super init];
    if (!self) {
        return nil;
    }

    // ======================== 设置NSURLSession ==============================
    // 如果 configuration 为 ‘nil’，使用 defaultSessionConfiguration
    if (!configuration) {
        configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
    }

    self.sessionConfiguration = configuration;

    self.operationQueue = [[NSOperationQueue alloc] init];
    // delegate 回调队列，最大并发数为 1
    self.operationQueue.maxConcurrentOperationCount = 1;

    // 初始化 Session
    self.session = [NSURLSession sessionWithConfiguration:self.sessionConfiguration delegate:self delegateQueue:self.operationQueue];
    
    // ======================== 初始化 response 序列化 ==============================
    self.responseSerializer = [AFJSONResponseSerializer serializer];
    
    // ======================== 初始化 网络安全策略 ==============================
    self.securityPolicy = [AFSecurityPolicy defaultPolicy];

    // ======================== 初始化 网络监控 ==============================
#if !TARGET_OS_WATCH
    self.reachabilityManager = [AFNetworkReachabilityManager sharedManager];
#endif

    // ======================== 初始化 Task 和 AFURLSessionManagerTaskDelegate 字典 ==============================
    // key 为 taskId，value 为 AFURLSessionManagerTaskDelegate
    self.mutableTaskDelegatesKeyedByTaskIdentifier = [[NSMutableDictionary alloc] init];

    // ======================== 初始化 mutableTaskDelegatesKeyedByTaskIdentifier 字典锁  ==============================
    // 保证 mutableTaskDelegatesKeyedByTaskIdentifier 的多线程安全
    self.lock = [[NSLock alloc] init];
    self.lock.name = AFURLSessionManagerLockName;

    // ======================== 获取当前 Session 中所有的 Task  ==============================
    [self.session getTasksWithCompletionHandler:^(NSArray *dataTasks, NSArray *uploadTasks, NSArray *downloadTasks) {
        // 为已有的 task 设置代理（在`生成NSURLSessionTask`细说）
        for (NSURLSessionDataTask *task in dataTasks) {
            [self addDelegateForDataTask:task uploadProgress:nil downloadProgress:nil completionHandler:nil];
        }

        for (NSURLSessionUploadTask *uploadTask in uploadTasks) {
            [self addDelegateForUploadTask:uploadTask progress:nil completionHandler:nil];
        }

        for (NSURLSessionDownloadTask *downloadTask in downloadTasks) {
            [self addDelegateForDownloadTask:downloadTask progress:nil destination:nil completionHandler:nil];
        }
    }];

    return self;
}
```

这就是 AFURLSessionManager 的初始化方法，主要对其属性进行出初始化。
需要注意的是这两个私有属性：

```objc
@property (readwrite, nonatomic, strong) NSMutableDictionary *mutableTaskDelegatesKeyedByTaskIdentifier;
@property (readwrite, nonatomic, strong) NSLock *lock;
```
AFURLSessionManager 会给每个管理的 task 对应的创建一个 AFURLSessionManagerTaskDelegate 对象，Manager把 task 的具体处理交给 delegate 对象，这样一个 AFURLSessionManager 就能同时管理多个 task。

### 生成 NSURLSessionTask
初始化 AFURLSessionManager 实例后，通过👇的方法创建 **NSURLSessionDataTask**：

```objc
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request
                               uploadProgress:(nullable void (^)(NSProgress *uploadProgress))uploadProgressBlock
                             downloadProgress:(nullable void (^)(NSProgress *downloadProgress))downloadProgressBlock
                            completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject,  NSError * _Nullable error))completionHandler;
                            
- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request
           fromFile:(NSURL *)fileURL
           progress:(nullable void (^)(NSProgress *uploadProgress))uploadProgressBlock
  completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject, NSError  * _Nullable error))completionHandler;
  
- (NSURLSessionDownloadTask *)downloadTaskWithRequest:(NSURLRequest *)request
                                        progress:(nullable void (^)(NSProgress *downloadProgress))downloadProgressBlock
                                     destination:(nullable NSURL * (^)(NSURL *targetPath, NSURLResponse *response))destination
                               completionHandler:(nullable void (^)(NSURLResponse *response, NSURL * _Nullable filePath, NSError * _Nullable error))completionHandler;
                               
......
```
这只是部分方法，这些方法都是类似的。

这里以**- [AFURLSessionManager dataTaskWithRequest:uploadProgress:downloadProgress:completionHandler:]**方法类举例说明，分析它如何实例化返回一个 **NSURLSessionTask**：

```objc
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request
                               uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgressBlock
                             downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgressBlock
                            completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject,  NSError * _Nullable error))completionHandler {

    __block NSURLSessionDataTask *dataTask = nil;
    url_session_manager_create_task_safely(^{
        dataTask = [self.session dataTaskWithRequest:request];
    });

    [self addDelegateForDataTask:dataTask uploadProgress:uploadProgressBlock downloadProgress:downloadProgressBlock completionHandler:completionHandler];

    return dataTask;
}
```
> **url_session_manager_create_task_safely** 是 为了修复 iOS 8以下偶发的taskIdentifiers不唯一的 bug[#2093](https://github.com/AFNetworking/AFNetworking/issues/2093)。

上述方法完成了：
1. 调用**- [NSURLSession dataTaskWithRequest:]**方法传入 request 生成 NSURLSessionDataTask
2. 调用**- [AFURLSessionManager addDelegateForDataTask:uploadProgress:downloadProgress:completionHandler:]**为该 dataTask 对象生成一个对应的 AFURLSessionManagerTaskDelegate 对象，并且关联起来。


```objc
- (void)addDelegateForDataTask:(NSURLSessionDataTask *)dataTask
                uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgressBlock
              downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgressBlock
             completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
{
	 // 创建 AFURLSessionManagerTaskDelegate 对象
	 AFURLSessionManagerTaskDelegate *delegate = [[AFURLSessionManagerTaskDelegate alloc] initWithTask:dataTask];
	 // AFURLSessionManagerTaskDelegate与AFURLSessionManager建立相互关系
	 delegate.manager = self;
	 // delegate 设置 完成block
	 delegate.completionHandler = completionHandler;
	 // 设置 task 的 taskDescription
	 dataTask.taskDescription = self.taskDescriptionForSessionTasks;
	 // 将 delegate 和 Task 关联起来
	 [self setDelegate:delegate forTask:dataTask];
	 // 设置 上传block 和 下载block
	 delegate.uploadProgressBlock = uploadProgressBlock;
	 delegate.downloadProgressBlock = downloadProgressBlock;
}
```
将 **completionHandler**， **uploadProgressBlock** 和 **downloadProgressBlock** 传入该对象并在相应事件发生时进行回调。

AFNetworking 通过封装 AFURLSessionManagerTaskDelegate 对象 ，让 delegate 对每个 task 进行管理，而 AFURLSessionManager 仅需要管理 保存 task 与 delegate 的 字典即可，实现了功能下放。

这个方法里面调用了另一个方法**- [AFURLSessionManager setDelegate:forTask:]**设置代理：

```objc
- (void)setDelegate:(AFURLSessionManagerTaskDelegate *)delegate
            forTask:(NSURLSessionTask *)task
{
	// 检查参数
	NSParameterAssert(task);
	NSParameterAssert(delegate);
	// 上锁，保证线程安全
	[self.lock lock];
	// 把 delegate 放入字典，key 是 taskIdentifier
	self.mutableTaskDelegatesKeyedByTaskIdentifier[@(task.taskIdentifier)] = delegate;
	// 监听 task 的 resume 和 Suspend
	[self addNotificationObserverForTask:task];
	// 解锁
	[self.lock unlock];
}
```

## AFURLSessionManager 实现了 NSURLSessionDelegate 等多个协议

在 AFURLSessionManager 的头文件中，遵循了很多协议，包括：
* NSURLSessionDelegate
* NSURLSessionTaskDelegate
* NSURLSessionDataDelegate
* NSURLSessionDownloadDelegate

在初始化方法**- [AFURLSessionManager initWithSessionConfiguration:]**时，把 NSURLSession 的 delegate 指向了 self（AFURLSessionManager）。

AFURLSessionManager 也为 所有的 delegate协议 提供了对应 block 接口设置：

```objc
- (void)setSessionDidBecomeInvalidBlock:(nullable void (^)(NSURLSession *session, NSError *error))block;

- (void)setSessionDidReceiveAuthenticationChallengeBlock:(nullable NSURLSessionAuthChallengeDisposition (^)(NSURLSession *session, NSURLAuthenticationChallenge *challenge, NSURLCredential * _Nullable __autoreleasing * _Nullable credential))block;

......
```
👆只是其中一部分。

拿 **- [AFNRLSessionManager setSessionDidBecomeInvalidBlock:]** 举例

AFURLSessionManager 开放对外的接口，把 sessionDidBecomeInvalid 回调的 block 传入：

```objc
- (void)setSessionDidBecomeInvalidBlock:(void (^)(NSURLSession *session, NSError *error))block {
    self.sessionDidBecomeInvalid = block;
}
```

当 **- [URLSession:didBecomeInvalidWithError:]** 代理方法调用时，判断对应的 block 是否存在，就会执行对应的 block：

```objc
- (void)URLSession:(NSURLSession *)session
didBecomeInvalidWithError:(NSError *)error
{
    if (self.sessionDidBecomeInvalid) {
        self.sessionDidBecomeInvalid(session, error);
    }

    [[NSNotificationCenter defaultCenter] postNotificationName:AFURLSessionDidInvalidateNotification object:session];
}
```

其他的接口也类似这样的实现，只是实现的内容不一样。

## AFURLSessionManagerTaskDelegate

👆提到的 AFURLSessionManagerTaskDelegate 是管理 task 的类，在 **- [AFURLSessionManager addDelegateForDataTask:uploadProgress:downloadProgress:completionHandler:]** 的方法里面创建。

首先看下它的初始化方法：

```objc
- (instancetype)initWithTask:(NSURLSessionTask *)task {
    self = [super init];
    if (!self) {
        return nil;
    }
    
    // 创建 NSData 存放请求回来的数据
    _mutableData = [NSMutableData data];
    // 上传的进度
    _uploadProgress = [[NSProgress alloc] initWithParent:nil userInfo:nil];
    // 下载的进度
    _downloadProgress = [[NSProgress alloc] initWithParent:nil userInfo:nil];
    
    __weak __typeof__(task) weakTask = task;
    // 给两个进度设置 取消，暂停，启动block 回调
    for (NSProgress *progress in @[ _uploadProgress, _downloadProgress ])
    {
        progress.totalUnitCount = NSURLSessionTransferSizeUnknown;
        progress.cancellable = YES;
        progress.cancellationHandler = ^{
            [weakTask cancel];
        };
        progress.pausable = YES;
        progress.pausingHandler = ^{
            [weakTask suspend];
        };
        if ([progress respondsToSelector:@selector(setResumingHandler:)]) {
            progress.resumingHandler = ^{
                [weakTask resume];
            };
        }
        // kVO 监听状态改变
        [progress addObserver:self
                   forKeyPath:NSStringFromSelector(@selector(fractionCompleted))
                      options:NSKeyValueObservingOptionNew
                      context:NULL];
    }
    return self;
}
```

> 设置 block 回调，主要是在对应 NSProgress 的状态改变时，调用 resume suspend 等方法改变 task 的状态。

在 **observeValueForKeypath:ofObject:change:context:** 方法中调用 block

```objc
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSString *,id> *)change context:(void *)context {
   if ([object isEqual:self.downloadProgress]) {
        if (self.downloadProgressBlock) {
            self.downloadProgressBlock(object);
        }
    }
    else if ([object isEqual:self.uploadProgress]) {
        if (self.uploadProgressBlock) {
            self.uploadProgressBlock(object);
        }
    }
}
```
当 Progress对象 某些属性改变时，调用block。

往下看你会发现 AFURLSessionManagerTaskDelegate 中也实现了 **NSURLSession** 的一些协议

![Snip20170125_1](http://p44bkxib3.bkt.clouddn.com/Snip20170125_1.png)

之前在 AFURLSessionManager 中，也实现了 **NSURLSession** 的协议

![Snip20170125_2](http://p44bkxib3.bkt.clouddn.com/Snip20170125_2.png)

> 因为 AFURLSessionManager 所管理的 NSURLSession 对象的 delegate 被设置为 AFURLSessionManager 自身，所以所有的 NSURLSession 协议回调都是 AFURLSessionManager，然后 AFURLSessionManager 根据具体需要，将 响应的 delegate 传递到 AFURLSessionManagerTaskDelegate 里面。

每当需要 AFURLSessionManagerTaskDelegate 具体处理的时候，AFURLSessionManager 都会取出相应的 taskDelegate 原封不动的传入。

### 代理方法 URLSession:task:didCompleteWithError:

当每一个 **NSURLSessionTask** 结束时都会进入这个回调方法中：

```objc
- (void)URLSession:(__unused NSURLSession *)session
              task:(NSURLSessionTask *)task
didCompleteWithError:(NSError *)error
{
    #1：获取数据 `responseSerializer` 和 `downloadFileURL`，存储到字典里面

    if (error) {
        #2：在存在错误时调用 `completionHandler`
    } else {
        #3：调用 `completionHandler`
    }
}
```
👆是整体的思路，先看看第一部分：

```objc
__strong AFURLSessionManager *manager = self.manager;

__block id responseObject = nil;

__block NSMutableDictionary *userInfo = [NSMutableDictionary dictionary];
userInfo[AFNetworkingTaskDidCompleteResponseSerializerKey] = manager.responseSerializer;

//Performance Improvement from #2672
NSData *data = nil;
if (self.mutableData) {
// 从 mutableData 中取出数据 
   data = [self.mutableData copy];
   //We no longer need the reference, so nil it out to gain back some memory.
   self.mutableData = nil;
}

// 设置 userInfo
if (self.downloadFileURL) {
   // 文件路径
   userInfo[AFNetworkingTaskDidCompleteAssetPathKey] = self.downloadFileURL;
} else if (data) {
   userInfo[AFNetworkingTaskDidCompleteResponseDataKey] = data;
}
```

第二部分：

```objc
// 错误存在，设置 userInfo 错误
userInfo[AFNetworkingTaskDidCompleteErrorKey] = error;

// 如果 manager 持有的 completionGroup 存在，就使用 completionGroup，否则使用默认创建的 url_session_manager_completion_group() GCD组队列
// 如果 manager 持有的 completionQueue 存在，就使用 completionQueue，否则使用 dispatch_get_main_queue() 主队列
dispatch_group_async(manager.completionGroup ?: url_session_manager_completion_group(), manager.completionQueue ?: dispatch_get_main_queue(), ^{
  // 调用完成block
  if (self.completionHandler) {
      self.completionHandler(task.response, responseObject, error);
  }

  // 子线程中发送 Task 结束通知 
  dispatch_async(dispatch_get_main_queue(), ^{
      [[NSNotificationCenter defaultCenter] postNotificationName:AFNetworkingTaskDidCompleteNotification object:task userInfo:userInfo];
  });
});
```

第三部分：

```objc
dispatch_async(url_session_manager_processing_queue(), ^{
  NSError *serializationError = nil;
  // 序列化响应数据
  responseObject = [manager.responseSerializer responseObjectForResponse:task.response data:data error:&serializationError];
  // 👇还是设置 userInfo， 调用 block ，发出通知
  if (self.downloadFileURL) {
      responseObject = self.downloadFileURL;
  }

  if (responseObject) {
      userInfo[AFNetworkingTaskDidCompleteSerializedResponseKey] = responseObject;
  }

  if (serializationError) {
      userInfo[AFNetworkingTaskDidCompleteErrorKey] = serializationError;
  }

  dispatch_group_async(manager.completionGroup ?: url_session_manager_completion_group(), manager.completionQueue ?: dispatch_get_main_queue(), ^{
      if (self.completionHandler) {
          self.completionHandler(task.response, responseObject, serializationError);
      }

      dispatch_async(dispatch_get_main_queue(), ^{
          [[NSNotificationCenter defaultCenter] postNotificationName:AFNetworkingTaskDidCompleteNotification object:task userInfo:userInfo];
      });
  });
});
```

### 代理方法 URLSession:dataTask:didReceiveData: 和 - URLSession:downloadTask:didFinishDownloadingToURL:


```objc
- (void)URLSession:(__unused NSURLSession *)session
          dataTask:(__unused NSURLSessionDataTask *)dataTask
    didReceiveData:(NSData *)data
{
    self.downloadProgress.totalUnitCount = dataTask.countOfBytesExpectedToReceive;
    self.downloadProgress.completedUnitCount = dataTask.countOfBytesReceived;
    // 追加数据
    [self.mutableData appendData:data];
}

- (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
didFinishDownloadingToURL:(NSURL *)location
{
    self.downloadFileURL = nil;

    if (self.downloadTaskDidFinishDownloading) {
        // 下载完成文件的路径
        self.downloadFileURL = self.downloadTaskDidFinishDownloading(session, downloadTask, location);
        if (self.downloadFileURL) {
            NSError *fileManagerError = nil;
            // 文件转移
            if (![[NSFileManager defaultManager] moveItemAtURL:location toURL:self.downloadFileURL error:&fileManagerError]) {
                // 文件转移失败通知
                [[NSNotificationCenter defaultCenter] postNotificationName:AFURLSessionDownloadTaskDidFailToMoveFileNotification object:downloadTask userInfo:fileManagerError.userInfo];
            }
        }
    }
}
```

## _AFURLSessionTaskSwizzling

_AFURLSessionTaskSwizzling 的唯一功能就是修改 NSURLSessionTask 的 resume 和 suspend 方法，使用下面的方法替换原有的实现：

```objc
- (void)af_resume {
    NSAssert([self respondsToSelector:@selector(state)], @"Does not respond to state");
    NSURLSessionTaskState state = [self state];
    // 经过 method swizzling 后，af_resume 就是之前的 resume ，所以这里是调用系统的 resume 方法
    [self af_resume];
    // 如果之前是其他状态，发出通知，更改回 resume 状态，通知调用 taskDidResume
    if (state != NSURLSessionTaskStateRunning) {
        [[NSNotificationCenter defaultCenter] postNotificationName:AFNSURLSessionTaskDidResumeNotification object:self];
    }
}
// 同上
- (void)af_suspend {
    NSAssert([self respondsToSelector:@selector(state)], @"Does not respond to state");
    NSURLSessionTaskState state = [self state];
    [self af_suspend];
    
    if (state != NSURLSessionTaskStateSuspended) {
        [[NSNotificationCenter defaultCenter] postNotificationName:AFNSURLSessionTaskDidSuspendNotification object:self];
    }
}
```

👆这样做的目的是为了在方法 resume 或者 suspend 被调用是发出通知。

具体方法调剂的过程发生在 **+load** 方法中进行
> load 方法会在加载类的时候就被调用，也就是 iOS 应用启动的时候就会加载所有的类，就会调用每个类的 +load 方法。

```objc
+ (void)load {
    // 判断当前 iOS 版本中是否存在 NSURLSessionTask
    if (NSClassFromString(@"NSURLSessionTask")) {
        // 1.创建 NSURLSession 实例，通过实例创建 NSURLSessionDataTask
        NSURLSessionConfiguration *configuration = [NSURLSessionConfiguration ephemeralSessionConfiguration];
        NSURLSession * session = [NSURLSession sessionWithConfiguration:configuration];
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wnonnull"
        // 因为 iOS7 和 iOS8 上对于 NSURLSessionTask 的实现不同，所以会通过 - [NSURLSession dataTaskWithURL:] 方法返回一个 NSURLSessionTask 实例
        NSURLSessionDataTask *localDataTask = [session dataTaskWithURL:nil];
#pragma clang diagnostic pop
        // 2.获取 af_resume 的方法实现指针
        IMP originalAFResumeIMP = method_getImplementation(class_getInstanceMethod([self class], @selector(af_resume)));
        Class currentClass = [localDataTask class];
        
        // 3.判断当前class是否实现了resume
        while (class_getInstanceMethod(currentClass, @selector(resume))) {
            // 4.获取当前class的父类
            Class superClass = [currentClass superclass];
            // 5.获取当前class的resume的方法实现指针
            IMP classResumeIMP = method_getImplementation(class_getInstanceMethod(currentClass, @selector(resume)));
            // 6.获取当前class父类的resume的方法实现指针
            IMP superclassResumeIMP = method_getImplementation(class_getInstanceMethod(superClass, @selector(resume)));
            // 7.如果当前class对于resume的实现和父类不一样（类似iOS7上的情况），并且当前class的resume实现和af_resume不一样，才进行method swizzling
            if (classResumeIMP != superclassResumeIMP &&
                originalAFResumeIMP != classResumeIMP) {
                [self swizzleResumeAndSuspendMethodForClass:currentClass];
            }
            // 8.设置当前操作的class为其父类class，重复while步骤3~8
            currentClass = [currentClass superclass];
        }
        
        [localDataTask cancel];
        [session finishTasksAndInvalidate];
    }
}
```
这里复杂的实现是为了解决 bug[#2702](https://github.com/AFNetworking/AFNetworking/pull/2702)

## NSSecureCoding和NSCopying

### NSSecureCoding
关于NSSecureCoding的讲解请参考[使用NSSecureCoding协议进行编解码](http://codingobjc.com/blog/2014/04/15/shi-yong-nssecurecodingxie-yi-jin-xing-bian-jie-ma/)。

因为要支持secure coding，所以要在supportsSecureCoding返回YES。

AFURLSessionManager保存的信息是其NSURLSessionConfiguration变量，然后根据获取到的configuration构建出AFURLSessionManager对象，节省了存储空间。

```objc
+ (BOOL)supportsSecureCoding {
    return YES;
}

- (instancetype)initWithCoder:(NSCoder *)decoder {
    NSURLSessionConfiguration *configuration = [decoder decodeObjectOfClass:[NSURLSessionConfiguration class] forKey:@"sessionConfiguration"];

    self = [self initWithSessionConfiguration:configuration];
    if (!self) {
        return nil;
    }

    return self;
}

- (void)encodeWithCoder:(NSCoder *)coder {
    [coder encodeObject:self.session.configuration forKey:@"sessionConfiguration"];
}
```

### NSCopying

先构建一个AFURLSessionManager空间，并使用原先session的configuration来初始化空间内容。
```objc
- (instancetype)copyWithZone:(NSZone *)zone {
    return [[[self class] allocWithZone:zone] initWithSessionConfiguration:self.session.configuration];
}
```

## 总结
AFURLSessionManager 里面的内容大致讲完了，具体实现细节还是需要细细推敲的：
1. AFURLSessionManager 是对 NSURLSession 的封装
2. 它通过 - [AFURLSessionManager dataTaskWithRequest:completionHandler:] 等接口创建 NSURLSessionDataTask 的实例
3. 通过一个字典 mutableTaskDelegatesKeyedByTaskIdentifier 来管理 dataTask 对象
4. AFURLSessionManager 是通过 AFURLSessionManagerTaskDelegate 来对传入的 **uploadProgressBlock**， **downloadProgressBlock**， **completionHandler** 在合适的时间进行调用

## 参考链接
* [AFNetworking](https://github.com/AFNetworking/AFNetworking)
* [AFNetworking源码阅读（四）](http://www.cnblogs.com/polobymulberry/p/5160946.html)










