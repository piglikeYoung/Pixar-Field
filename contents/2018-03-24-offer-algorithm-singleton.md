
## 实现单例模式

设计一个类，只能生成该类的一个实例。

在 Objective-C 中单例的公认的写法类似下面这样：

```Objc
@implementation MyManager
+ (id)sharedManager {
    static MyManager * staticInstance = nil;
    static dispatch_once_t onceToken;

    dispatch_once(&onceToken, ^{
        staticInstance = [[self alloc] init];
    });
    return staticInstance;
}
@end
```

在 Swift 1.2 之后可以推荐写：

```Swift
class MyManager  {
    static let shared = MyManager()
    private init() {}
}
```

