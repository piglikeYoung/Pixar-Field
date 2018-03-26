
## 前言

iOS 开发中总会用到各种缓存，本文介绍的是 **YYCache** 是一个线程安全的高性能键值缓存（该项目是 YYKit 组件之一）。YYKit 作者是 @ibireme 。

[YYCache](https://github.com/ibireme/YYCache) 的代码逻辑清晰，注释详尽，加上自身不算太大的代码量使得其阅读非常简单，更加厉害的是它的性能还非常高。

详情请看：[《YYCache 设计思路与技术细节》](https://blog.ibireme.com/2015/10/26/yycache/)

我把 YYCache 从头到尾看了一遍，最大的感触就是代码风格干净整洁，代码思路清晰明了，注释详细。

我们先来简单看一下 YYCache 的代码结构，YYCache 是由 YYMemoryCache 与 YYDiskCache 两部分组成的，其中 YYMemoryCache 作为高速内存缓存，而 YYDiskCache 则作为低速磁盘缓存。


```
YYCache
 	YYMemoryCache
 		_YYLinkedMap
		_YYLinkedMapNode
	YYDiskCache
		YYKVStorage
  		YYKVStorageItem
```

👆是 YYCache 的主要结构，通常一个缓存是由内存缓存和磁盘缓存组成，内存缓存提供容量小但高速的存取功能，磁盘缓存提供大容量但相对低速的持久化存储。

## YYCache


```objc
@interface YYCache : NSObject

@property (copy, readonly) NSString *name;
@property (strong, readonly) YYMemoryCache *memoryCache;
@property (strong, readonly) YYDiskCache *diskCache;

- (BOOL)containsObjectForKey:(NSString *)key;
- (nullable id<NSCoding>)objectForKey:(NSString *)key;
- (void)setObject:(nullable id<NSCoding>)object forKey:(NSString *)key;
- (void)removeObjectForKey:(NSString *)key;

@end

```

我将 YYCache 里的代码精简了一下，总的来说就这些增删改查的方法， 里面有 YYMemoryCache 与 YYDiskCache，并且对外提供了一些接口。这些接口基本都是基于 Key 和 Value 设计的，和字典的操作类似。

## YYMemoryCache 

```objc
@interface YYMemoryCache : NSObject

#pragma mark - Attribute
/**
    cache 的名称，默认为nil
 */
@property (nullable, copy) NSString *name;

/**
    memory 中的消息总数
 */
@property (readonly) NSUInteger totalCount;

/**
    memory 中的消息总开销
 */
@property (readonly) NSUInteger totalCost;


#pragma mark - Limit

/**
 消息池子 cache 中存储的最大数量
 默认的值为 NSUIntegerMax 表示无限制
 如果超过此限制，则稍后在后台线程将清除一些对象
 */
@property NSUInteger countLimit;

/**
 消息池子cache中容许的最大开销
 默认的值为 NSUIntegerMax 表示无限制
 如果超过此限制，则稍后在后台线程将清除一些对象
 */
@property NSUInteger costLimit;

/**
 消息池子cache中容许的时间限制
 默认的值为 DBL_MAX 表示无限制
 如果超过此限制，则稍后在后台线程将清除一些对象
 */
@property NSTimeInterval ageLimit;

/**
 自动检测容器限制 默认时间 5.0s
 cache 消息池子持有 Timer，以确保 cache 是否达到上限 如果达到上限则进行削减
 */
@property NSTimeInterval autoTrimInterval;

/**
 如果是 YES 则收到内存报警时会删除所有的 cache 消息对象
 默认值是 YES
 */
@property BOOL shouldRemoveAllObjectsOnMemoryWarning;

/**
 如果是 YES 则收到app进入后台时会删除所有的cache消息对象
 默认值是 YES
 */
@property BOOL shouldRemoveAllObjectsWhenEnteringBackground;

/** 
 app 收到报警时执行的 block 默认为 nil
 */
@property (nullable, copy) void(^didReceiveMemoryWarningBlock)(YYMemoryCache *cache);

/** 
 app 收到进入后台时执行的 block 默认为 nil
 */
@property (nullable, copy) void(^didEnterBackgroundBlock)(YYMemoryCache *cache);

/**
 键值对是否在主线程删除 默认值为NO.
 仅仅当键值对中包含 UIView、CALayer 等非线程安全对象时，将值设为YES
 */
@property BOOL releaseOnMainThread;

/**
 键值对异步的释放 默认值为 YES
 避免堵塞访问方法 否则将在 removeObjectForKey: 等方法中释放 默认是 YES
 */
@property BOOL releaseAsynchronously;


#pragma mark - Access Methods

/** 
 判断消息池子是否包含指定key的消息
 key 消息对象关联的key. 如果是nil则返回NO
 是否包含指定key的消息
 */
- (BOOL)containsObjectForKey:(id)key;

/**
 获取与key关联的消息对象
 key 关联消息对象的 key,如果是 nil 则返回 nil
 返回与 key 关联的消息对象, 如果未找到则返回 nil
 */
- (nullable id)objectForKey:(id)key;

/**
 根据指定的 key 存储消息对象
 message 需要存储到池子的对象. 如果是nil则调用 `removeMessageForKey`.
 key 存储对象关联的key. 如果是nil则不执行任何操作
 与NSMutableDictionary相比, cache池子不会拷贝容器中的键值对
 */
- (void)setObject:(nullable id)object forKey:(id)key;

/**
 根据指定的key和开销cost存储消息
 object 需要存储到池子的对象. 如果是nil则调用 `removeObjectForKey`.
 key 存储对象关联的key. 如果是nil则不执行任何操作
 cost 关联键值对的开销
 与NSMutableDictionary相比, cache池子不会拷贝容器中的键值对
 */
- (void)setObject:(nullable id)object forKey:(id)key withCost:(NSUInteger)cost;

/**
 根据指定的key删除消息
 key 需要删除的object的key. 如果是nil则不执行任何操作
 */
- (void)removeObjectForKey:(id)key;

/**
 删除所有的消息
 */
- (void)removeAllObjects;


#pragma mark - Trim

/**
 用 LRU 算法删除对象，直到 totalCount <= count
 */
- (void)trimToCount:(NSUInteger)count;

/**
 用 LRU 算法删除对象，直到 totalCost <= cost
 */
- (void)trimToCost:(NSUInteger)cost;

/**
 用 LRU 算法删除对象，直到所有到期对象全部被删除
 */
- (void)trimToAge:(NSTimeInterval)age;

@end
```

上面是 YYMemoryCache.h 的主要属性和接口，加上了注释。

### _YYLinkedMapNode 和 _YYLinkedMap

YYMemoryCache 内部其实是通过 **_YYLinkedMapNode** 和 **_YYLinkedMap** 这两个对象操作缓存的。


```objc
/**
 _YYLinkedMap 中的一个节点。
 通常情况下我们不应该使用这个类。
 */
@interface _YYLinkedMapNode : NSObject {
    @package
    __unsafe_unretained _YYLinkedMapNode *_prev; // retained by dic 前一个消息 && 被字典保留
    __unsafe_unretained _YYLinkedMapNode *_next; // retained by dic 后一个消息 && 被字典保留
    id _key;    /// 消息的key
    id _value;  /// 消息
    NSUInteger _cost;   /// 消息开销
    NSTimeInterval _time;   /// 消息时间
}
@end

/**
 YYMemoryCache 内的一个链表。
 _YYLinkedMap 不是一个线程安全的类，而且它也不对参数做校验。
 通常情况下我们不应该使用这个类。
 */
@interface _YYLinkedMap : NSObject {
    @package
    CFMutableDictionaryRef _dic; // do not set object directly 保存消息的字典，外部不要直接设置
    NSUInteger _totalCost;  // 消息总开销
    NSUInteger _totalCount; // 消息总量
    _YYLinkedMapNode *_head; // MRU, do not change it directly MRU最近最常使用, 外部不要直接修改
    _YYLinkedMapNode *_tail; // LRU, do not change it directly LRU最近最少使用, 外部不要直接修改
    BOOL _releaseOnMainThread;  // 是否在主线程release
    BOOL _releaseAsynchronously;    // 是否异步release
}
```

可以看出来 _YYLinkedMapNode 是双向链表， _YYLinkedMap 是双向链表的节点。

_YYLinkedMapNode 记录着它的前一个节点 **_prev** 和 后一个节点 **_next**，并且记录着缓存信息的 **_key** 和 **_value**，这样一个节点就保存缓存数据，可以理解为一个节点就是一个缓存对象。

_YYLinkedMap 使用 **CFMutableDictionaryRef _dic** 字典存储 _YYLinkedMapNode。这样即强引用了节点，又能够利用字典的 Hash 快速定位用户要访问的缓存对象，并且当需要是否手动释放字典的时候，能够通过 CFRelease 手动释放。

### 线程安全

```objc
@implementation YYMemoryCache {
    pthread_mutex_t _lock; // 线程锁，保证线程安全
    _YYLinkedMap *_lru;	// YYMemoryCache 通过操作Map来管理缓存
    dispatch_queue_t _queue;	// 串行队列，用于后台 trim（清扫工作）
}
```

在 YYMemoryCache 中作者是使用 `pthread_mutex` 来保证线程安全的，但是最开始的版本并不是用 pthread_mutex ，而是使用自旋锁 **OSSpinLock**，可以查看[YYCache 设计思路](https://blog.ibireme.com/2015/10/26/yycache/)得知改动的原因。

### LRU

LRU(least-recently-used) 算法翻译过来是”最近最少使用“，顾名思义这种缓存替换策略是基于用户最近访问过的缓存对象而建立。

- 从代码实现上看缓存替换策略的核心思想在于：LRU 认为用户最新使用（访问）过的缓存对象为高频缓存对象，即用户很可能还会再次使用（访问）该缓存对象；而反之，用户很久之前使用（访问）过的缓存对象（期间一直没有再次访问）为低频缓存对象，即用户很可能不会再去使用（访问）该缓存对象，通常在资源不足时会先去释放低频缓存对象。

### _YYLinkedMapNode 和 _YYLinkedMap 使用 LRU

从双向链表可以知道，链表存在两个节点：

* 头结点：用户最近使用的数据，MRU
* 尾节点：用户很久之前使用的数据，LRU


```objc
- (id)objectForKey:(id)key {
    if (!key) return nil;
    pthread_mutex_lock(&_lock);
    // 找到节点
    _YYLinkedMapNode *node = CFDictionaryGetValue(_lru->_dic, (__bridge const void *)(key));
    if (node) {
    		// 更新节点的时间戳
        node->_time = CACurrentMediaTime();
        // 将节点移动到头结点
        [_lru bringNodeToHead:node];
    }
    pthread_mutex_unlock(&_lock);
    return node ? node->_value : nil;
}
```

每次根据 Key 获取某个节点的时候，都会更新节点的时间戳并移动到头结点。


```objc
- (void)setObject:(id)object forKey:(id)key withCost:(NSUInteger)cost {
    if (!key) return;
    if (!object) {
        [self removeObjectForKey:key];
        return;
    }
    pthread_mutex_lock(&_lock);
    _YYLinkedMapNode *node = CFDictionaryGetValue(_lru->_dic, (__bridge const void *)(key));
    NSTimeInterval now = CACurrentMediaTime();
    // 判断节点是否存在
    if (node) {
    		// 存在更新节点，并移动至头节点
        _lru->_totalCost -= node->_cost;
        _lru->_totalCost += cost;
        node->_cost = cost;
        node->_time = now;
        node->_value = object;
        [_lru bringNodeToHead:node];
    } else {
        // 不存在，创建一个新的节点，插入头节点
        node = [_YYLinkedMapNode new];
        node->_cost = cost;
        node->_time = now;
        node->_key = key;
        node->_value = object;
        [_lru insertNodeAtHead:node];
    }
    if (_lru->_totalCost > _costLimit) {
        dispatch_async(_queue, ^{
            [self trimToCost:_costLimit];
        });
    }
    if (_lru->_totalCount > _countLimit) {
        _YYLinkedMapNode *node = [_lru removeTailNode];
        if (_lru->_releaseAsynchronously) {
            dispatch_queue_t queue = _lru->_releaseOnMainThread ? dispatch_get_main_queue() : YYMemoryCacheGetReleaseQueue();
            dispatch_async(queue, ^{
                [node class]; //hold and release in queue
            });
        } else if (_lru->_releaseOnMainThread && !pthread_main_np()) {
            dispatch_async(dispatch_get_main_queue(), ^{
                [node class]; //hold and release in queue
            });
        }
    }
    pthread_mutex_unlock(&_lock);
}
```

当设置新节点时，判断节点是否存在，存在更新节点，并移动至头节点；不存在，创建一个新的节点，插入头节点。


```objc
/**
 消息池子按照数量限制清扫
 */
- (void)_trimToCount:(NSUInteger)countLimit {
    BOOL finish = NO;
    pthread_mutex_lock(&_lock);
    if (countLimit == 0) {
        [_lru removeAll];
        finish = YES;
    } else if (_lru->_totalCount <= countLimit) {
        finish = YES;
    }
    pthread_mutex_unlock(&_lock);
    if (finish) return;
    
    NSMutableArray *holder = [NSMutableArray new];
    while (!finish) {
        if (pthread_mutex_trylock(&_lock) == 0) {
            if (_lru->_totalCount > countLimit) {
            	  // 当缓存数量超出限制的时候，先从尾节点（LRU）开始清除，释放资源
                _YYLinkedMapNode *node = [_lru removeTailNode];
                if (node) [holder addObject:node];
            } else {
                finish = YES;
            }
            pthread_mutex_unlock(&_lock);
        } else {
            // 使用 usleep 以微秒为单位挂起线程，在短时间间隔挂起线程
            // 对比 sleep 用 usleep 能更好的利用 CPU 时间
            usleep(10 * 1000); //10 ms
        }
    }
    if (holder.count) {
        // 判断是否在主线程释放对象
        dispatch_queue_t queue = _lru->_releaseOnMainThread ? dispatch_get_main_queue() : YYMemoryCacheGetReleaseQueue();
        dispatch_async(queue, ^{
            [holder count]; // release in queue
        });
    }
}
```

清除的代码其实不是太难，YYCache 从 count 、cost 和 age 三个维度去做清除工作，具体请去查阅源码。


在代码底部判断是否在主线程释放资源，是作者另一篇文章[iOS 保持界面流畅的技巧](https://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/)中提到：

> 对象的销毁虽然消耗资源不多，但累积起来也是不容忽视的。通常当容器类持有大量对象时，其销毁时的资源消耗就非常明显。同样的，如果对象可以放到后台线程去释放，那就挪到后台线程去。这里有个小 Tip：把对象捕获到 block 中，然后扔到后台队列去随便发送个消息以避免编译器警告，就可以让对象在后台线程销毁了。


```objc
NSArray *tmp = self.array;
self.array = nil;
dispatch_async(queue, ^{
    [tmp class];
});
```

## YYDiskCache


```objc
/**
 YYDiskCache 是一个线程安全的磁盘缓存，用于存储由 SQLite 和文件系统支持的键值对（类似于 NSURLCache 的磁盘缓存）。

YYDiskCache 具有以下功能：

* 它使用 LRU(least-recently-used) 来删除对象。
* 支持按 cost，count 和 age 进行控制。
* 它可以被配置为当没有可用的磁盘空间时自动驱逐缓存对象。
* 它可以自动抉择每个缓存对象的存储类型（sqlite/file）以便提供更好的性能表现。

你可以编译最新版本的 sqlite 并忽略 iOS 系统中的 libsqlite3.dylib 来获得 2x〜4x 的速度提升
 */
@interface YYDiskCache : NSObject

#pragma mark - Attribute
/**
 The name of the cache. Default is nil.
 磁盘cache的名称
 */
@property (nullable, copy) NSString *name;

/**
 The path of the cache (read-only).
 磁盘cache的文件路径
 */
@property (readonly) NSString *path;

/**
 如果存入的消息超过此值，则消息会存入文件file，否则存入sqlite
 0 意味着所有的消息会存入不同的文件file, NSUIntegerMax 意味着所有的消息会存入 sqlite.
 默认的值为 20480 (20KB).
 */
@property (readonly) NSUInteger inlineThreshold;

/**
 如果block为nil 则会使用NSKeyedArchiver归档消息 使用此block以支持未遵循`NSCoding` 协议的对象存储
 默认值为nil
 */
@property (nullable, copy) NSData *(^customArchiveBlock)(id object);

/**
 block不为nil则使用自定义的解归档方法替代 NSKeyedUnarchiver. 使用此block以支持未遵循`NSCoding` 协议的对象
 默认值为nil
 */
@property (nullable, copy) id (^customUnarchiveBlock)(NSData *data);

/**
 当需要写文件时, block 会生成文件名和一个key，如果block是nil 则cache使用MD5生成默认的文件名
 默认值为nil
 */
@property (nullable, copy) NSString *(^customFileNameBlock)(NSString *key);



#pragma mark - Limit

/**
 消息池子cache中存储的最大数量
 默认的值为 NSUIntegerMax 表示无限制
 它并不是一个严格的限制 - 如果缓存超过限制，那么一些缓存对象就会在后台队列中被回收。
 */
@property NSUInteger countLimit;

/**
 消息池子cache中容许的最大开销
 默认的值为 NSUIntegerMax 表示无限制
 它并不是一个严格的限制 - 如果缓存超过限制，那么一些缓存对象就会在后台队列中被回收。
 */
@property NSUInteger costLimit;

/**
 消息池子cache中容许的时间限制
 默认的值为 DBL_MAX 表示无限制
 它并不是一个严格的限制 - 如果缓存超过限制，那么一些缓存对象就会在后台队列中被回收。
 */
@property NSTimeInterval ageLimit;

/**
 cache保证的最小磁盘disk空闲
 默认值为 0, 意味着无限制
 如果disk空闲容量小于此值，将移除对象释放内存
 */
@property NSUInteger freeDiskSpaceLimit;

/**
 自动检测容器限制 默认时间60.0s
 cache消息池子持有Timer,以确保cache是否达到上限 如果达到上限则进行削减
 */
@property NSTimeInterval autoTrimInterval;

/**
 设置`YES` 容许错误log
 */
@property BOOL errorLogsEnabled;

#pragma mark - Initializer

- (instancetype)init UNAVAILABLE_ATTRIBUTE;
+ (instancetype)new UNAVAILABLE_ATTRIBUTE;

/**
 根据path实例化磁盘cache对象
 path cache写入消息的全路径 实例化后，不要在此路径读写数据
 返回 cache 对象, 如果发生错误返回nil
 如果path已经存在内存中，则会直接返回cache对象 取代创建对象
 */
- (nullable instancetype)initWithPath:(NSString *)path;

/**
 推荐的实例化方法
 path cache写入消息的全路径 实例化后，不要在此路径读写数据
 threshold  存入数据尺寸的限制. 如果存入sqlite数据字节数超过此值 则会写入文件,
 0 意味着所有的消息会存入不同的文件file, NSUIntegerMax 意味着所有的消息会存入 sqlite 推荐值为20480
 返回 cache 对象, 如果发生错误返回nil
 如果path已经存在内存中，则会直接返回cache对象 取代创建对象
 */
- (nullable instancetype)initWithPath:(NSString *)path
                      inlineThreshold:(NSUInteger)threshold NS_DESIGNATED_INITIALIZER;


#pragma mark - Access Methods

/**
 返回一个boolean 表示给定的key是否存在disk的cache中 此方法会堵塞直到返回
 key 标识消息对象的key 如果为nil 则返回NO
 返回key是否存在cache中
 */
- (BOOL)containsObjectForKey:(NSString *)key;

/**
 返回一个boolean 表示给定的key是否存在disk的cache中 此方法会立即返回，并在后台线程中执行，直到执行完成调用block回调
 key   标识消息对象的key 如果为nil 则返回NO
 block 在后台线程执行完成后的回调block
 */
- (void)containsObjectForKey:(NSString *)key withBlock:(void(^)(NSString *key, BOOL contains))block;

/**
 返回指定key对应的消息 此方法会堵塞直到返回
 key 标识消息对象的key 如果为nil 则返回nil
 返回key对应的, 如果未找到，则返回nil
 */
- (nullable id<NSCoding>)objectForKey:(NSString *)key;

/**
 返回指定key对应的消息  此方法会立即返回，并在后台线程中执行，直到执行完成调用block回调
 key 标识消息对象的key 如果为nil 则返回nil
 block 在后台线程执行完成后的回调block
 */
- (void)objectForKey:(NSString *)key withBlock:(void(^)(NSString *key, id<NSCoding> _Nullable object))block;

/**
 将消息和对应的key值存入cache中 此方法会堵塞直到写入数据完成
 object 存入cache中的消息对象. 如果是nil则会调用`removeObjectForKey:`.
 key    和消息对象关联的key. 如果为nil则不会操作
 */
- (void)setObject:(nullable id<NSCoding>)object forKey:(NSString *)key;

/**
 将消息和对应的key值存入cache中 此方法会立即返回，并在后台线程中执行，直到执行完成调用block回调
 object 存入cache中的消息对象. 如果是nil则会调用`removeObjectForKey:`.
 key    和消息对象关联的key. 如果为nil则不会操作
 block  在后台执行完后的回调block
 */
- (void)setObject:(nullable id<NSCoding>)object forKey:(NSString *)key withBlock:(void(^)(void))block;

/**
 删除cache中指定key对应的消息 此方法会堵塞直到文件删除完成
 key 标识删除对象的key 如果为nil则不会操作
 */
- (void)removeObjectForKey:(NSString *)key;

/**
 删除cache中指定key对应的消息 此方法会立即返回，并在后台线程中执行，直到执行完成调用block回调
 key 标识删除对象的key 如果为nil则不会操作
 block  在后台执行完后的回调block
 */
- (void)removeObjectForKey:(NSString *)key withBlock:(void(^)(NSString *key))block;

/**
 删除cache中所有的对象 此方法会堵塞直到cache清除完成
 */
- (void)removeAllObjects;

/**
 删除cache中所有的对象 此方法会立即返回，并在后台线程中执行，直到执行完成调用block回调
 block  在后台执行完后的回调block
 */
- (void)removeAllObjectsWithBlock:(void(^)(void))block;

/**
 删除cache中所有的对象 此方法会立即返回，并在后台线程中执行，直到执行完成调用block回调
 不要在block中对该对象发送消息
 progress 删除过程中执行, nil的话忽略
 end      删除完成后执行, nil的话忽略
 */
- (void)removeAllObjectsWithProgressBlock:(nullable void(^)(int removedCount, int totalCount))progress
                                 endBlock:(nullable void(^)(BOOL error))end;


/**
 返回cache中的消息总数量 此方法会堵塞直到读取完成
 返回消息总数
 */
- (NSInteger)totalCount;

/**
 获取cache中的消息总数量 此方法会立即返回，并在后台线程中执行，直到执行完成调用block回调
 block  在后台执行完后的回调block
 */
- (void)totalCountWithBlock:(void(^)(NSInteger totalCount))block;

/**
 返回cache中的消息总开销（字节） 此方法会堵塞直到读取完成
 返回消息总开销（字节）
 */
- (NSInteger)totalCost;

/**
 返回cache中的消息总开销（字节）此方法会立即返回，并在后台线程中执行，直到执行完成调用block回调
 block  在后台执行完后的回调block
 */
- (void)totalCostWithBlock:(void(^)(NSInteger totalCost))block;


#pragma mark - Trim

/**
 一旦 `totalCount` 高于总数限制，则删除消息 将LRU对象放入缓存区 此方法会堵塞直到完成
 count  清除消息后容许的消息总数量
 */
- (void)trimToCount:(NSUInteger)count;

/**
 一旦 `totalCount` 高于总数限制，则删除消息 将LRU对象放入缓存区 此方法会立即返回，并在后台线程中执行，直到执行完成调用block回调
 count  清除消息后容许的消息总数量
 block 完成后的回调
 */
- (void)trimToCount:(NSUInteger)count withBlock:(void(^)(void))block;

/**
 一旦 `totalCount` 高于总开销限制，则删除消息 将LRU对象放入缓存区 此方法会堵塞直到完成
 count  清除消息后容许的消息总开销
 */
- (void)trimToCost:(NSUInteger)cost;

/**
 一旦 `totalCount` 高于总开销限制，则删除消息 将LRU对象放入缓存区 此方法会立即返回，并在后台线程中执行，直到执行完成调用block回调
 count  清除消息后容许的消息总开销
 block 完成后的回调
 */
- (void)trimToCost:(NSUInteger)cost withBlock:(void(^)(void))block;

/**
 按照时间限制削减 （LRU对象进入缓冲区）此方法会堵塞
 age  最大的时间 seconds.
 */
- (void)trimToAge:(NSTimeInterval)age;

/**
 一旦 按照时间限制削减 将LRU对象放入缓存区 此方法会立即返回，并在后台线程中执行，直到执行完成调用block回调
 age  最大的时间 seconds.
 block 完成后的回调
 */
- (void)trimToAge:(NSTimeInterval)age withBlock:(void(^)(void))block;


#pragma mark - Extended Data

/**
 获取消息的拓展数据
 详见'setExtendedData:toObject:'
 object 消息对象
 拓展数据
 */
+ (nullable NSData *)getExtendedDataFromObject:(id)object;

/**
 设置消息的拓展数据
 当保存消息到cache之前可以设置消息的拓展数据 拓展数据会同样存入cache中 你可以使用"getExtendedDataFromObject:"获取拓展数据
 extendedData 拓展数据 (如果是nil 则删除数据)
 object       对应的消息
 */
+ (void)setExtendedData:(nullable NSData *)extendedData toObject:(id)object;

@end
```

👆是 YYDiskCache 的接口和属性，我都加上了注释。

YYDiskCache 是分成 sqlite 和 file 存储的，作者设计的时候是根据文件大小来划分存储方式：

* sqlite: 对于小数据（例如 NSNumber）的存取效率明显高于 file。
* file: 对于较大数据（例如高质量图片）的存取效率优于 sqlite。

YYDiskCache 使用两个相互配合的方式提高存储性能。

### _YYDiskCacheGetGlobal 和 _YYDiskCacheSetGlobal

```objc
- (instancetype)initWithPath:(NSString *)path
             inlineThreshold:(NSUInteger)threshold {
    self = [super init];
    if (!self) return nil;
    
    YYDiskCache *globalCache = _YYDiskCacheGetGlobal(path);
    ``````
}
```

根据公开初始化方法初始化 YYDiskCache 时，发现内部是调用静态方法去创建实例：

```objc
/**
 Map表保存cache实例，管理所有根据 path 创建的 YYDiskCache 实例
 */
static NSMapTable *_globalInstances;
/**
 线程信号
 */
static dispatch_semaphore_t _globalInstancesLock;

/**
 静态变量实例化
 */
static void _YYDiskCacheInitGlobal() {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        // 初始化信号量
        _globalInstancesLock = dispatch_semaphore_create(1);
        // 创建 NSMapTable，Key 强引用，Value 弱引用
        _globalInstances = [[NSMapTable alloc] initWithKeyOptions:NSPointerFunctionsStrongMemory valueOptions:NSPointerFunctionsWeakMemory capacity:0];
    });
}

/**
 获取 NSMapTable 中的 YYDiskCache 实例
 */
static YYDiskCache *_YYDiskCacheGetGlobal(NSString *path) {
    if (path.length == 0) return nil;
    _YYDiskCacheInitGlobal();
    dispatch_semaphore_wait(_globalInstancesLock, DISPATCH_TIME_FOREVER);
    // 通过 NSMapTable 获取 YYDiskCache
    id cache = [_globalInstances objectForKey:path];
    dispatch_semaphore_signal(_globalInstancesLock);
    return cache;
}

/**
 设置 NSMapTable 中的 YYDiskCache 实例，key 值为 cache 路径
 */
static void _YYDiskCacheSetGlobal(YYDiskCache *cache) {
    if (cache.path.length == 0) return;
    _YYDiskCacheInitGlobal();
    dispatch_semaphore_wait(_globalInstancesLock, DISPATCH_TIME_FOREVER);
    // 通过 NSMapTable 设置 YYDiskCache
    [_globalInstances setObject:cache forKey:cache.path];
    dispatch_semaphore_signal(_globalInstancesLock);
}
```

👆代码里面创建 YYDiskCache 的时候使用了 **NSMapTable** 来保存实例对象，并且创建的过程都有加上**信号锁**。

它是 iOS 6 才引入的数据结构集合，用法类似 NSDictionary，但是**它可以对 Value 弱引用**。关于 NSMapTable 更多的语义和使用参考[《iOS中的NSHashTable和NSMapTable》](https://www.jianshu.com/p/dcd222900fa9) 和 [《NSMapTable 官方文档》](https://developer.apple.com/documentation/foundation/nsmaptable?language=objc)。

每当一个 YYDiskCache 被初始化时，其实会先到 NSMapTable 中获取对应 path 的 YYDiskCache 实例，如果获取不到才会去真正的初始化一个 YYDiskCache 实例，并且将其引用在 NSMapTable 中，这样做提升不少性能。


> Note：dispatch_semaphore 是信号量，但当信号总量设为 1 时也可以当作锁来。在没有等待情况出现时，它的性能比 pthread_mutex 还要高，但一旦有等待情况出现时，性能就会下降许多。相对于 OSSpinLock 来说，它的优势在于等待时不会消耗 CPU 资源。对磁盘缓存来说，它比较合适。


### YYKVStorageItem 和 YYKVStorage

由于 YYDiskCache 有时候是操作 sqlite ，有时候是操作 file，所以使用 YYKVStorage 来统一管理缓存对象（sqlite/file），YYKVStorage 其实就是对应着 _YYLinkedMap，YYKVStorageItem 对应 _YYLinkedMapNode。

```objc
/**
 YYKVStorageItem 用来存储键值对数据及拓展数据，通常不应该直接使用它
 */
@interface YYKVStorageItem : NSObject

/**
 消息key值
 */
@property (nonatomic, strong) NSString *key;                ///< key

/**
 消息data数据
 */
@property (nonatomic, strong) NSData *value;                ///< value

/**
 消息文件名
 */
@property (nullable, nonatomic, strong) NSString *filename; ///< filename (nil if inline)

/**
 消息大小（字节）
 */
@property (nonatomic) int size;                             ///< value's size in bytes

/**
 消息修改时间
 */
@property (nonatomic) int modTime;                          ///< modification unix timestamp

/**
 消息导入时间
 */
@property (nonatomic) int accessTime;                       ///< last access unix timestamp

/**
 拓展数据
 */
@property (nullable, nonatomic, strong) NSData *extendedData; ///< extended data (nil if no extended data)
@end

/**
 消息存储类型，表示存储“YYKVStorageItem.value”的位置。
 
 一般而言,数据存入sqlite比写文件更快，但读取数据的性能依赖数据大小 以iPhone 6 64G为例
 数据超过20KB，则从文件读取数据比sqlite读取更快
 存储较小的数据使用 YYKVStorageTypeSQLite 获得更好的性能
 如果存储较大的数据 如图片数据, 使用 YYKVStorageTypeFile 获取更好的性能
 使用 LSMessageDiskStorageTypeMixed 将针对每一个item采用不同的存储方式
 详见 http://www.sqlite.org/intern-v-extern-blob.html
 
 */
typedef NS_ENUM(NSUInteger, YYKVStorageType) {
    
    /// The `value` is stored as a file in file system.
    /// 消息存入文件
    YYKVStorageTypeFile = 0,
    
    /// The `value` is stored in sqlite with blob type.
    /// 消息存入sqlite，采用blob的类型
    YYKVStorageTypeSQLite = 1,
    
    /// The `value` is stored in file system or sqlite based on your choice.
    /// 根据选择选取存入方式
    YYKVStorageTypeMixed = 2,
};



/** 
 消息写入file/sqlite的管理类
 @discussion 键值对的方式将消息存入文件和sqlite 使用`initWithPath:type:`进行初始化
 初始化后 不要再对生成的path进行读写操作 使用最新的sqlite版本获取2-4倍的速度提升
 产生的实例并不是线程安全的，应该在同一时间在同一的线程使用，数据较大时，应该对数据进行拆分成多个片段进行存储
 */
@interface YYKVStorage : NSObject

#pragma mark - Attribute

/**
 消息存入的路径
 */
@property (nonatomic, readonly) NSString *path;        ///< The path of this storage.

/**
 消息存储类型
 */
@property (nonatomic, readonly) YYKVStorageType type;  ///< The type of this storage.

/**
 是否打印log
 */
@property (nonatomic) BOOL errorLogsEnabled;           ///< Set `YES` to enable error logs for debug.

#pragma mark - 初始化
- (instancetype)init UNAVAILABLE_ATTRIBUTE;
+ (instancetype)new UNAVAILABLE_ATTRIBUTE;

/**
 推荐的实例化方法
 path  写数据的路径. 如果路径存在，则会在此路径读写数据 否则建立一个新路径
 type  存储类型  一旦设置后不要修改
 返回一个存储管理实例, 发生错误返回nil
 多个实例操作同一个路径 会导致错误
 */
- (nullable instancetype)initWithPath:(NSString *)path type:(YYKVStorageType)type NS_DESIGNATED_INITIALIZER;


#pragma mark - 保存消息

/**
 保存item key值存在时更新item
 @discussion 此方法会将 item.key, item.value, item.filename 和
 item.extendedData 写入文件或sqlite, 其他属性会忽略. item.key
 和 item.value 不应该为空 (nil || length == 0).
 item  消息item
 返回是否成功
 */
- (BOOL)saveItem:(YYKVStorageItem *)item;

/**
 保存item key值存在时更新item
 此方法会保存键值对到 sqlite. 如果存储类型为 YYKVStorageTypeFile , 此方法会失败
 key   key值不能为空
 value value不能为空
 返回是否成功
 */
- (BOOL)saveItemWithKey:(NSString *)key value:(NSData *)value;

/**
 保存item key值存在时更新item
 如果写入类型为LSMessageDiskStorageTypeFile,filename 不能为空
 如果写入类型为LSMessageDiskStorageTypeSQLite, filename 会被忽略
 如果写入类型为LSMessageDiskStorageTypeMixed, 如果filename不为空 则value会被存入文件 否则存入sqlite
 key           key值不能为空
 value         value不能为空
 filename      文件名
 extendedData  item的拓展数据 如果是nil则忽略
 Whether succeed.
 */
- (BOOL)saveItemWithKey:(NSString *)key
                  value:(NSData *)value
               filename:(nullable NSString *)filename
           extendedData:(nullable NSData *)extendedData;

#pragma mark - 删除消息

/** 
 根据key值删除item
 keys 特定的key值
 返回是否删除成功
 */
- (BOOL)removeItemForKey:(NSString *)key;

/**
 根据keys数组删除items
 keys keys数组
 返回是否删除成功
 */
- (BOOL)removeItemForKeys:(NSArray<NSString *> *)keys;

/**
 根据消息value的开销限制删除items
 size 消息value的最大限制
 返回是否删除成功
 */
- (BOOL)removeItemsLargerThanSize:(int)size;

/**
 删除比指定时间更早存入的消息
 time  指定的时间
 返回是否删除成功
 */
- (BOOL)removeItemsEarlierThanTime:(int)time;

/**
 根据消息开销限制删除items (LRU对象优先删除)
 maxCount 最大的消息开销
 返回是否删除成功
 */
- (BOOL)removeItemsToFitSize:(int)maxSize;

/**
 根据消息数量限制删除items (LRU对象优先删除)
 maxCount 最大的消息数量
 返回是否删除成功
 */
- (BOOL)removeItemsToFitCount:(int)maxCount;

/**
 在后台队列中，删除所有的item
 @discussion 此方法会删除 files 和 sqlite database 进入回收站 并在后台清除回收站数据
 比`removeAllItemsWithProgressBlock:endBlock:`方法更快
 @return 返回是否删除成功
 */
- (BOOL)removeAllItems;

/**
 删除所有的item
 @warning 在block中不要对该实例发送消息
 progress 删除时执行的block，nil则不执行
 end      删除结束执行的block，nil则不执行
 */
- (void)removeAllItemsWithProgressBlock:(nullable void(^)(int removedCount, int totalCount))progress
                               endBlock:(nullable void(^)(BOOL error))end;


#pragma mark - 获取消息

/**
 根据key获取item
 key  特定的key值
 返回item, 发送错误返回nil
 */
- (nullable YYKVStorageItem *)getItemForKey:(NSString *)key;

/**
 根据key获取item的信息（value会被忽略）
 key  特定的key值
 返回item的信息, 发送错误返回nil
 */
- (nullable YYKVStorageItem *)getItemInfoForKey:(NSString *)key;

/**
 根据key获取item的value
 key  特定的key值
 返回item的value, 发送错误返回nil
 */
- (nullable NSData *)getItemValueForKey:(NSString *)key;

/**
 根据key的数组获取item的信息
 keys  key值的数组
 包含`YYKVStorageItem`的数组 发生错误返回nil
 */
- (nullable NSArray<YYKVStorageItem *> *)getItemForKeys:(NSArray<NSString *> *)keys;

/**
 根据key的数组获取item的信息（value会被忽略）
 keys  key值的数组
 包含`LSMessageDiskStorageItem`的数组 发生错误返回nil
 */
- (nullable NSArray<YYKVStorageItem *> *)getItemInfoForKeys:(NSArray<NSString *> *)keys;

/**
 根据一个key值数组获取item和key的字典
 keys  key值的数组
 返回一个字典 key->item对应 发生错误返回nil
 */
- (nullable NSDictionary<NSString *, NSData *> *)getItemValueForKeys:(NSArray<NSString *> *)keys;

#pragma mark - 获取存储属性

/**
 根据key值查找item是否存在
 key  特定的key
 返回item是否存在
 */
- (BOOL)itemExistsForKey:(NSString *)key;

/**
 获取item的总数
 返回总数，如果发生错误返回-1
 */
- (int)getItemsCount;

/**
 获取items的总大小（字节）
 返回总大小，如果发生错误返回-1
 */
- (int)getItemsSize;

@end
```

当 YYDiskCache 存储对象的时候，会判断存储数据库的文件大小最大阈值，超过了会生成文件名，写入文件，然后将文件名存储到 sqlite 中。

```objc
/// YYDiskCache.m

- (void)setObject:(id<NSCoding>)object forKey:(NSString *)key {
    if (!key) return;
    if (!object) {
        [self removeObjectForKey:key];
        return;
    }
    
    NSData *extendedData = [YYDiskCache getExtendedDataFromObject:object];
    NSData *value = nil;
    if (_customArchiveBlock) {
        value = _customArchiveBlock(object);
    } else {
        @try {
            value = [NSKeyedArchiver archivedDataWithRootObject:object];
        }
        @catch (NSException *exception) {
            // nothing to do...
        }
    }
    if (!value) return;
    NSString *filename = nil;
    if (_kv.type != YYKVStorageTypeSQLite) {
        // 如果超过数据库写入大小限制，生成文件名
        if (value.length > _inlineThreshold) {
            filename = [self _filenameForKey:key];
        }
    }
    
    Lock();
    [_kv saveItemWithKey:key value:value filename:filename extendedData:extendedData];
    Unlock();
}

/// YYKVStorage.m
/**
 保存item key值存在时更新item
 */
- (BOOL)saveItemWithKey:(NSString *)key value:(NSData *)value filename:(NSString *)filename extendedData:(NSData *)extendedData {
    // 没有 Key，也没有 Value 直接返回 NO
    if (key.length == 0 || value.length == 0) return NO;
    // 存文件，但是没有文件名，也直接返回 NO
    if (_type == YYKVStorageTypeFile && filename.length == 0) {
        return NO;
    }
    
    if (filename.length) {
        // 写文件失败，返回 NO
        if (![self _fileWriteWithName:filename data:value]) {
            return NO;
        }
        // 将文件名写入数据库，之后方便根据 Key 去查找文件
        if (![self _dbSaveWithKey:key value:value fileName:filename extendedData:extendedData]) {
            // 如果写入数据库失败，把之前写入的文件删除
            [self _fileDeleteWithName:filename];
            return NO;
        }
        return YES;
    } else {
        if (_type != YYKVStorageTypeSQLite) {
            NSString *filename = [self _dbGetFilenameWithKey:key];
            if (filename) {
                [self _fileDeleteWithName:filename];
            }
        }
        return [self _dbSaveWithKey:key value:value fileName:nil extendedData:extendedData];
    }
}

```

### YYKVStorage 性能优化细节


```objc

CFMutableDictionaryRef _dbStmtCache;

/**
 db设置sqlite3_stmt
 */
- (sqlite3_stmt *)_dbPrepareStmt:(NSString *)sql {
    if (![self _dbCheck] || sql.length == 0 || !_dbStmtCache) return NULL;
    // 先尝试从 _dbStmtCache 根据入参 sql 取出已缓存 sqlite3_stmt
    sqlite3_stmt *stmt = (sqlite3_stmt *)CFDictionaryGetValue(_dbStmtCache, (__bridge const void *)(sql));
    if (!stmt) {
        // 如果没有缓存再从新生成一个 sqlite3_stmt
        int result = sqlite3_prepare_v2(_db, sql.UTF8String, -1, &stmt, NULL);
        // 生成结果异常则根据错误日志开启标识打印日志
        if (result != SQLITE_OK) {
            if (_errorLogsEnabled) NSLog(@"%s line:%d sqlite stmt prepare error (%d): %s", __FUNCTION__, __LINE__, result, sqlite3_errmsg(_db));
            return NULL;
        }
        // 生成成功则放入 _dbStmtCache 缓存
        CFDictionarySetValue(_dbStmtCache, (__bridge const void *)(sql), stmt);
    } else {
        sqlite3_reset(stmt);
    }
    return stmt;
}
```

每次操作 sqlite 的时候，都有调用 _dbPrepareStmt 方法获取 sqlite3_stmt 缓存，sqlite3_stmt 保存在 _dbStmtCache 字典中，每次都先从字典里面获取缓存，这样不需要重复生成 sqlite3_stmt。

> sqlite3_stmt: 该对象的实例表示已经编译成二进制形式并准备执行的单个 SQL 语句。


## 总结

YYCache 的设计相当清晰，功能相当强大，具备了优秀缓存的能力：

* 内存缓存和磁盘缓存
* 线程安全
* 缓存控制
* 缓存替换策略
* 性能

### 内存缓存和磁盘缓存

内存缓存 YYMemoryCache 与磁盘缓存 YYDiskCache 相互配合组成的，内存缓存提供容量小但高速的存取功能，磁盘缓存提供大容量但低速的持久化存储。这样的设计支持用户在缓存不同对象时都能够有很好的体验。

在 YYCache 中使用接口访问缓存对象时，会先去尝试从内存缓存 YYMemoryCache 中访问，如果访问不到（没有使用该 key 缓存过对象或者该对象已经从容量有限的 YYMemoryCache 中淘汰掉）才会去从 YYDiskCache 访问，如果访问到（表示之前确实使用该 key 缓存过对象，该对象已经从容量有限的 YYMemoryCache 中淘汰掉成立）会先在 YYMemoryCache 中更新一次该缓存对象的访问信息之后才返回给接口。

### 线程安全

YYMemoryCache 使用了 pthread_mutex 线程锁来确保线程安全，而 YYDiskCache 则选择了更适合它的 dispatch_semaphore

### 缓存控制

提供了 cost、count、age 三个维度去控制缓存，满足绝大多数的需求。

### 缓存替换策略

使用了 LRU(least-recently-used) 策略去提高缓存效率。

### 性能

从上面的分析就可以看出来了：

* 异步释放缓存对象
* 锁的选择
* 使用 NSMapTable 单例管理的 YYDiskCache
* YYKVStorage 中的 _dbStmtCache
* 使用 CoreFoundation 来换取手动释放内存提高效率

[YYCache](https://github.com/piglikeYoung/YYCache) 这个是我 fork 的库，加了一些代码注释，可以参考下。


