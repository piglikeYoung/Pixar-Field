
## 前言
前面的文字已经分析了 **AFURLSessionManager** 和 **AFNetworkReachabilityManager**，一个是 AFN 的核心，另一个是 AFN 非常有用的工具类。

这里我们接着了解 AFURLSessionManager 中使用到的**对发出请求和接收响应时**进行序列化的两个模块：
* AFURLResponseSerialization
* AFURLRequestSerialization

前者是处理响应的模块，将请求返回的数据解析成对用的格式。后者主要是修改请求的头部（主要是 HTTP 请求）。

AFURLResponseSerialization 主要使用在 AFURLSessionManager 中，而 AFURLRequestSerialization 主要用于 AFHTTPSessionManager 中，因为它主要用于**修改 HTTP 头部**。

## AFURLResponseSerialization

AFURLResponseSerialization 其实并不是一个类，它是个协议，它只有一个需要实现的方法：

```objc
@protocol AFURLResponseSerialization <NSObject, NSSecureCoding, NSCopying>
- (nullable id)responseObjectForResponse:(nullable NSURLResponse *)response
                           data:(nullable NSData *)data
                          error:(NSError * _Nullable __autoreleasing *)error NS_SWIFT_NOTHROW;
@end
```

遵循这个协议的同时也需要遵循 NSObject, NSSecureCoding 和 NSCopying 这三个协议，实现安全编码、拷贝以及Objective-C 对象的基本行为。

先了解下 AFURLResponseSerialization 的模型结构：

![Snip20170126_3](http://p44bkxib3.bkt.clouddn.com/Snip20170126_3.png)

* AFHTTPResponseSerializer 是模型中所有Serializer的根类
* 所有Serializer 都需要遵循 AFURLResponseSerialization 协议

## AFHTTPResponseSerializer
AFHTTPResponseSerializer 是所有响应解析的根类，因为它遵循了 AFURLResponseSerialization 协议，所以子类也要遵循。

### 初始化

```objc
+ (instancetype)serializer {
    return [[self alloc] init];
}

- (instancetype)init {
    self = [super init];
    if (!self) {
        return nil;
    }

    self.acceptableStatusCodes = [NSIndexSet indexSetWithIndexesInRange:NSMakeRange(200, 100)];
    self.acceptableContentTypes = nil;

    return self;
}
```

acceptableStatusCodes 设置为从 200 到 299 之间的状态码, 因为只有这些状态码表示获得了有效的响应。

### 验证响应

AFHTTPResponseSerializer 最重要的方法 **- [AFHTTPResponseSerializer validateResponse:data:error:]**


```objc
- (BOOL)validateResponse:(NSHTTPURLResponse *)response
                    data:(NSData *)data
                   error:(NSError * __autoreleasing *)error
{
    BOOL responseIsValid = YES;
    NSError *validationError = nil;

    if (response && [response isKindOfClass:[NSHTTPURLResponse class]]) {
        if (self.acceptableContentTypes && ![self.acceptableContentTypes containsObject:[response MIMEType]] &&
            !([response MIMEType] == nil && [data length] == 0)) {

            #1: 返回的数据类型无效
        }

        if (self.acceptableStatusCodes && ![self.acceptableStatusCodes containsIndex:(NSUInteger)response.statusCode] && [response URL]) {
            #2: 返回的状态码无效
        }
    }

    if (error && !responseIsValid) {
        *error = validationError;
    }

    return responseIsValid;
}
```

```objc
if (self.acceptableContentTypes && ![self.acceptableContentTypes containsObject:[response MIMEType]] &&
            !([response MIMEType] == nil && [data length] == 0)) {

  if ([data length] > 0 && [response URL]) {
      NSMutableDictionary *mutableUserInfo = [@{
                                                NSLocalizedDescriptionKey: [NSString stringWithFormat:NSLocalizedStringFromTable(@"Request failed: unacceptable content-type: %@", @"AFNetworking", nil), [response MIMEType]],
                                                NSURLErrorFailingURLErrorKey:[response URL],
                                                AFNetworkingOperationFailingURLResponseErrorKey: response,
                                              } mutableCopy];
      if (data) {
          mutableUserInfo[AFNetworkingOperationFailingURLResponseDataErrorKey] = data;
      }

      validationError = AFErrorWithUnderlyingError([NSError errorWithDomain:AFURLResponseSerializationErrorDomain code:NSURLErrorCannotDecodeContentData userInfo:mutableUserInfo], validationError);
  }

  responseIsValid = NO;
}
```
* 先判断 acceptableContentTypes 集合里面是否包含响应类型，如果不包含进入 if
* 在 if 里面通过 AFErrorWithUnderlyingError 包装错误信息，最后设置 responseIsValid = NO

第二段代码也是差不多的实现。


### 实现协议
其实就是调用👆的验证方法，然后返回数据
```objc
- (id)responseObjectForResponse:(NSURLResponse *)response
                           data:(NSData *)data
                          error:(NSError *__autoreleasing *)error
{
    [self validateResponse:(NSHTTPURLResponse *)response data:data error:error];

    return data;
}
```

### NSSecureCoding，NSCopying
最后是实现 NSSecureCoding，NSCopying 这两个协议，没什么好说的。

## AFJSONResponseSerializer

现在看下 AFJSONResponseSerializer ，它继承自 AFHTTPResponseSerializer。

### 初始化

调用父类初始化后，增加了 acceptableContentTypes 赋值

```objc
self.acceptableContentTypes = [NSSet setWithObjects:@"application/json", @"text/json", @"text/javascript", nil];
```


### 实现协议

这个子类对 AFURLResponseSerialization 协议实现比父类复杂得多

```objc
- (id)responseObjectForResponse:(NSURLResponse *)response
                           data:(NSData *)data
                          error:(NSError *__autoreleasing *)error
{
	  // 去父类验证响应有效
    if (![self validateResponse:(NSHTTPURLResponse *)response data:data error:error]) {
        if (!error || AFErrorOrUnderlyingErrorHasCodeInDomain(*error, NSURLErrorCannotDecodeContentData, AFURLResponseSerializationErrorDomain)) {
            return nil;
        }
    }

    // Workaround for behavior of Rails to return a single space for `head :ok` (a workaround for a bug in Safari), which is not interpreted as valid input by NSJSONSerialization.
    // See https://github.com/rails/rails/issues/1742
    
    // 解决只有一个空格引起的bug
    BOOL isSpace = [data isEqualToData:[NSData dataWithBytes:" " length:1]];
    
    if (data.length == 0 || isSpace) {
        return nil;
    }
    
    NSError *serializationError = nil;
    
    // 序列化JSON
    id responseObject = [NSJSONSerialization JSONObjectWithData:data options:self.readingOptions error:&serializationError];

    if (!responseObject)
    {
        if (error) {
            *error = AFErrorWithUnderlyingError(serializationError, *error);
        }
        return nil;
    }
    
    // JSON是否移除 ‘NSNull’ 默认是 NO
    if (self.removesKeysWithNullValues) {
        return AFJSONObjectByRemovingKeysWithNullValues(responseObject, self.readingOptions);
    }

    return responseObject;
}
```

移除 JSON 中 null 的函数 AFJSONObjectByRemovingKeysWithNullValues 是一个递归调用的函数：

```objc
static id AFJSONObjectByRemovingKeysWithNullValues(id JSONObject, NSJSONReadingOptions readingOptions) {
    if ([JSONObject isKindOfClass:[NSArray class]]) {
        NSMutableArray *mutableArray = [NSMutableArray arrayWithCapacity:[(NSArray *)JSONObject count]];
        for (id value in (NSArray *)JSONObject) {
            [mutableArray addObject:AFJSONObjectByRemovingKeysWithNullValues(value, readingOptions)];
        }

        return (readingOptions & NSJSONReadingMutableContainers) ? mutableArray : [NSArray arrayWithArray:mutableArray];
    } else if ([JSONObject isKindOfClass:[NSDictionary class]]) {
        NSMutableDictionary *mutableDictionary = [NSMutableDictionary dictionaryWithDictionary:JSONObject];
        for (id <NSCopying> key in [(NSDictionary *)JSONObject allKeys]) {
            id value = (NSDictionary *)JSONObject[key];
            if (!value || [value isEqual:[NSNull null]]) {
                [mutableDictionary removeObjectForKey:key];
            } else if ([value isKindOfClass:[NSArray class]] || [value isKindOfClass:[NSDictionary class]]) {
                mutableDictionary[key] = AFJSONObjectByRemovingKeysWithNullValues(value, readingOptions);
            }
        }

        return (readingOptions & NSJSONReadingMutableContainers) ? mutableDictionary : [NSDictionary dictionaryWithDictionary:mutableDictionary];
    }

    return JSONObject;
}
```
方法中判断类型是 NSArray 还是 NSDictionary，只有 NSDictionary 中存在 ‘null’，所有移除是靠 **[mutableDictionary removeObjectForKey:key];**

## AFURLRequestSerialization

AFURLRequestSerialization 相对复杂些，它的主要工作是对发出的 HTTP 请求进行处理。
这个文件中大部分类都是为 AFHTTPRequestSerializer 服务的：

1. 处理请求参数
2. 设置 HTTP 请求头
3. 设置请求URL的属性
4. 分块上传

> 分块上传相对复杂得多，有兴趣的可以去研究下，这里不进行细讲

### 处理请求参数

设置查询参数主要是通过 **AFQueryStringPair** 完成，它有两个属性 field 和 value 对应HTTP 请求的查询 URL 中的参数。

```objc
@interface AFQueryStringPair : NSObject
@property (readwrite, nonatomic, strong) id field;
@property (readwrite, nonatomic, strong) id value;

- (instancetype)initWithField:(id)field value:(id)value;

- (NSString *)URLEncodedStringValue;
@end
```

其中 **- [AFQueryStringPair URLEncodedStringValue]** 会返回 **key=value** 这种格式，同时使用 **AFPercentEscapedStringFromString** 函数来对 field 和 value 进行处理，将其中的 :#[]@!$&'()*+,;= 等字符转换为百分号表示的形式。

```objc
- (NSString *)URLEncodedStringValue {
    if (!self.value || [self.value isEqual:[NSNull null]]) {
        return AFPercentEscapedStringFromString([self.field description]);
    } else {
        return [NSString stringWithFormat:@"%@=%@", AFPercentEscapedStringFromString([self.field description]), AFPercentEscapedStringFromString([self.value description])];
    }
}
```

**AFQueryStringPairsFromKeyAndValue** 方法，如果 value 是集合类型，它会不断递归调用自己：

```objc
NSArray * AFQueryStringPairsFromKeyAndValue(NSString *key, id value) {
    NSMutableArray *mutableQueryStringComponents = [NSMutableArray array];

    NSSortDescriptor *sortDescriptor = [NSSortDescriptor sortDescriptorWithKey:@"description" ascending:YES selector:@selector(compare:)];

    if ([value isKindOfClass:[NSDictionary class]]) {
        NSDictionary *dictionary = value;
        // Sort dictionary keys to ensure consistent ordering in query string, which is important when deserializing potentially ambiguous sequences, such as an array of dictionaries
        for (id nestedKey in [dictionary.allKeys sortedArrayUsingDescriptors:@[ sortDescriptor ]]) {
            id nestedValue = dictionary[nestedKey];
            if (nestedValue) {
                [mutableQueryStringComponents addObjectsFromArray:AFQueryStringPairsFromKeyAndValue((key ? [NSString stringWithFormat:@"%@[%@]", key, nestedKey] : nestedKey), nestedValue)];
            }
        }
    } else if ([value isKindOfClass:[NSArray class]]) {
        NSArray *array = value;
        for (id nestedValue in array) {
            [mutableQueryStringComponents addObjectsFromArray:AFQueryStringPairsFromKeyAndValue([NSString stringWithFormat:@"%@[]", key], nestedValue)];
        }
    } else if ([value isKindOfClass:[NSSet class]]) {
        NSSet *set = value;
        for (id obj in [set sortedArrayUsingDescriptors:@[ sortDescriptor ]]) {
            [mutableQueryStringComponents addObjectsFromArray:AFQueryStringPairsFromKeyAndValue(key, obj)];
        }
    } else {
        // 不是集合类型，就创建 AFQueryStringPair 对象，添加到数组
        [mutableQueryStringComponents addObject:[[AFQueryStringPair alloc] initWithField:key value:value]];
    }

    return mutableQueryStringComponents;
}
```
最后会返回一个数组，数组里面是一个个 AFQueryStringPair 对象。

得到数组后会调用 **AFQueryStringFromParameters** 使用 **&** 来拼接它们。

```objc
NSString * AFQueryStringFromParameters(NSDictionary *parameters) {
    NSMutableArray *mutablePairs = [NSMutableArray array];
    for (AFQueryStringPair *pair in AFQueryStringPairsFromDictionary(parameters)) {
        [mutablePairs addObject:[pair URLEncodedStringValue]];
    }

    return [mutablePairs componentsJoinedByString:@"&"];
}
```
最终得到：

```
username=haha&password=123456
```

### 设置 HTTP 请求头

AFHTTPRequestSerializer 对外提供了 **- [AFHTTPRequestSerializer setValue:forHTTPHeaderField:]** 方法来设置请求头，内部实现是通过给一个可变字典 **mutableHTTPRequestHeaders** 赋值和取值获取：

```objc
- (void)setValue:(NSString *)value
forHTTPHeaderField:(NSString *)field
{
	[self.mutableHTTPRequestHeaders setValue:value forKey:field];
}

- (NSString *)valueForHTTPHeaderField:(NSString *)field {
    return [self.mutableHTTPRequestHeaders valueForKey:field];
}
```

真正用到的时候又是通过 **HTTPRequestHeaders** 这个方法获取不可变字典：

```objc
- (NSDictionary *)HTTPRequestHeaders {
    return [NSDictionary dictionaryWithDictionary:self.mutableHTTPRequestHeaders];
}

```

我们设置一般的请求头字段时，可以参考 AFHTTPRequestSerializer 初始化的设置

```objc
userAgent = [NSString stringWithFormat:@"%@/%@ (%@; iOS %@; Scale/%0.2f)", [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleExecutableKey] ?: [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleIdentifierKey], [[NSBundle mainBundle] infoDictionary][@"CFBundleShortVersionString"] ?: [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleVersionKey], [[UIDevice currentDevice] model], [[UIDevice currentDevice] systemVersion], [[UIScreen mainScreen] scale]];

[self setValue:userAgent forHTTPHeaderField:@"User-Agent"];
```

验证用户名和密码时，调用 **- [AFHTTPRequestSerializer setAuthorizationHeaderFieldWithUsername:password:]**：

```objc
- (void)setAuthorizationHeaderFieldWithUsername:(NSString *)username
                                       password:(NSString *)password
{
    NSData *basicAuthCredentials = [[NSString stringWithFormat:@"%@:%@", username, password] dataUsingEncoding:NSUTF8StringEncoding];
    NSString *base64AuthCredentials = [basicAuthCredentials base64EncodedStringWithOptions:(NSDataBase64EncodingOptions)0];
    [self setValue:[NSString stringWithFormat:@"Basic %@", base64AuthCredentials] forHTTPHeaderField:@"Authorization"];
}
```

### 设置请求URL的属性

设置完请求头和请求参数，还有一些关于请求的设置：

```objc
/**
 字符串编码，默认 `NSUTF8StringEncoding`.
 */
@property (nonatomic, assign) NSStringEncoding stringEncoding;

/**
 请求能够使用蜂窝无线电，默认 YES.
 */
@property (nonatomic, assign) BOOL allowsCellularAccess;

/**
 缓存策略，默认 `NSURLRequestUseProtocolCachePolicy`.
 */
@property (nonatomic, assign) NSURLRequestCachePolicy cachePolicy;

/**
 请求是否使用默认的cookie处理，默认 `YES`.
 */
@property (nonatomic, assign) BOOL HTTPShouldHandleCookies;

/**
 请求是否使用 管道， 默认 `NO` .
 */
@property (nonatomic, assign) BOOL HTTPShouldUsePipelining;

/**
 网络请求的服务类型，默认 `NSURLNetworkServiceTypeDefault`.
 */
@property (nonatomic, assign) NSURLRequestNetworkServiceType networkServiceType;

/**
 请求超时时间，默认是 60s.
 */
@property (nonatomic, assign) NSTimeInterval timeoutInterval;
```

这些值会通过调用 **AFHTTPRequestSerializerObservedKeyPaths** 返回一个数组：
```objc
static NSArray * AFHTTPRequestSerializerObservedKeyPaths() {
    static NSArray *_AFHTTPRequestSerializerObservedKeyPaths = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _AFHTTPRequestSerializerObservedKeyPaths = @[NSStringFromSelector(@selector(allowsCellularAccess)), NSStringFromSelector(@selector(cachePolicy)), NSStringFromSelector(@selector(HTTPShouldHandleCookies)), NSStringFromSelector(@selector(HTTPShouldUsePipelining)), NSStringFromSelector(@selector(networkServiceType)), NSStringFromSelector(@selector(timeoutInterval))];
    });

    return _AFHTTPRequestSerializerObservedKeyPaths;
}
```

AFURLRequestSerialization 是通过 KVO监听这些值的变化，当这些值被设置时，会触发KVO，然后把新的属性存储在 **mutableObservedChangedKeyPaths** 字典中：

```objc
+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)key {
    // 手动控制通知方式
    if ([AFHTTPRequestSerializerObservedKeyPaths() containsObject:key]) {
        return NO;
    }
    // 默认自动通知
    return [super automaticallyNotifiesObserversForKey:key];
}

- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(__unused id)object
                        change:(NSDictionary *)change
                       context:(void *)context
{
    if (context == AFHTTPRequestSerializerObserverContext) {
        if ([change[NSKeyValueChangeNewKey] isEqual:[NSNull null]]) {
            [self.mutableObservedChangedKeyPaths removeObject:keyPath];
        } else {
            [self.mutableObservedChangedKeyPaths addObject:keyPath];
        }
    }
}
```

最后在生成 NSURLRequest 请求时设置：

```objc
NSMutableURLRequest *mutableRequest = [[NSMutableURLRequest alloc] initWithURL:url];
mutableRequest.HTTPMethod = method;

for (NSString *keyPath in AFHTTPRequestSerializerObservedKeyPaths()) {
   if ([self.mutableObservedChangedKeyPaths containsObject:keyPath]) {
       [mutableRequest setValue:[self valueForKeyPath:keyPath] forKey:keyPath];
   }
}
```


### 初始化

```objc
- (instancetype)init {
    self = [super init];
    if (!self) {
        return nil;
    }

    self.stringEncoding = NSUTF8StringEncoding;

    self.mutableHTTPRequestHeaders = [NSMutableDictionary dictionary];

    #1: 设置 Accept-Language，User-Agent等。忽略

    // HTTP Method Definitions; see http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html
    self.HTTPMethodsEncodingParametersInURI = [NSSet setWithObjects:@"GET", @"HEAD", @"DELETE", nil];

    self.mutableObservedChangedKeyPaths = [NSMutableSet set];
    for (NSString *keyPath in AFHTTPRequestSerializerObservedKeyPaths()) {
        if ([self respondsToSelector:NSSelectorFromString(keyPath)]) {
            [self addObserver:self forKeyPath:keyPath options:NSKeyValueObservingOptionNew context:AFHTTPRequestSerializerObserverContext];
        }
    }

    return self;
}
```
上面提到到 KVO 监听属性变化，就是在初始化的时候做的，确保它们在改变后更新 NSMutableURLRequest 中对应的属性。

初始化完成后，如果调用普通的请求方法，就会进入这个方法：

```objc
- (NSMutableURLRequest *)requestWithMethod:(NSString *)method
                                 URLString:(NSString *)URLString
                                parameters:(id)parameters
                                     error:(NSError *__autoreleasing *)error
{
    // 检测参数
    NSParameterAssert(method);
    NSParameterAssert(URLString);

    // 生成url
    NSURL *url = [NSURL URLWithString:URLString];

    NSParameterAssert(url);

    NSMutableURLRequest *mutableRequest = [[NSMutableURLRequest alloc] initWithURL:url];
    // 设置请求方式
    mutableRequest.HTTPMethod = method;
    // 设置kVO监听的请求属性
    for (NSString *keyPath in AFHTTPRequestSerializerObservedKeyPaths()) {
        if ([self.mutableObservedChangedKeyPaths containsObject:keyPath]) {
            [mutableRequest setValue:[self valueForKeyPath:keyPath] forKey:keyPath];
        }
    }
    // 设置 HTTP 请求头和请求参数
    mutableRequest = [[self requestBySerializingRequest:mutableRequest withParameters:parameters error:error] mutableCopy];

	return mutableRequest;
}
```

调用 **- [AFHTTPRequestSerializer requestBySerializingRequest:withParameters:error:]** 方法，设置 HTTP 请求头和请求参数：
```objc
- (NSURLRequest *)requestBySerializingRequest:(NSURLRequest *)request
                               withParameters:(id)parameters
                                        error:(NSError *__autoreleasing *)error
{
    NSParameterAssert(request);

    NSMutableURLRequest *mutableRequest = [request mutableCopy];

    // 设置请求头
    [self.HTTPRequestHeaders enumerateKeysAndObjectsUsingBlock:^(id field, id value, BOOL * __unused stop) {
        if (![request valueForHTTPHeaderField:field]) {
            [mutableRequest setValue:value forHTTPHeaderField:field];
        }
    }];

    NSString *query = nil;
    if (parameters) {
        if (self.queryStringSerialization) {
            NSError *serializationError;
            query = self.queryStringSerialization(request, parameters, &serializationError);

            if (serializationError) {
                if (error) {
                    *error = serializationError;
                }

                return nil;
            }
        } else {
            switch (self.queryStringSerializationStyle) {
                case AFHTTPRequestQueryStringDefaultStyle:
                    // 把请求参数转换为 &拼接的字符串
                    query = AFQueryStringFromParameters(parameters);
                    break;
            }
        }
    }
    
    // 如果是 `GET`，`HEAD` 和 `DELETE`的请求，请求参数拼接到URL后面
    if ([self.HTTPMethodsEncodingParametersInURI containsObject:[[request HTTPMethod] uppercaseString]]) {
        if (query && query.length > 0) {
            mutableRequest.URL = [NSURL URLWithString:[[mutableRequest.URL absoluteString] stringByAppendingFormat:mutableRequest.URL.query ? @"&%@" : @"?%@", query]];
        }
    } else {
        // #2864: an empty string is a valid x-www-form-urlencoded payload
        if (!query) {
            query = @"";
        }
        if (![mutableRequest valueForHTTPHeaderField:@"Content-Type"]) {
            [mutableRequest setValue:@"application/x-www-form-urlencoded" forHTTPHeaderField:@"Content-Type"];
        }
        // 请求参数放入请求体
        [mutableRequest setHTTPBody:[query dataUsingEncoding:self.stringEncoding]];
    }

    return mutableRequest;
}
```


## 总结
1. AFURLResponseSerialization 负责对返回的数据进行序列化
2. AFURLRequestSerialization 负责生成 NSMutableURLRequest，为请求设置 HTTP 请求头和请求参数，管理发出的请求

## 参考链接
* [AFNetworking](https://github.com/AFNetworking/AFNetworking)
* [AFNetworking源码阅读（六）](http://www.cnblogs.com/polobymulberry/p/5174298.html)
* [处理请求和响应 AFURLSerialization（三）](https://github.com/Draveness/iOS-Source-Code-Analyze/blob/master/contents/AFNetworking/%E5%A4%84%E7%90%86%E8%AF%B7%E6%B1%82%E5%92%8C%E5%93%8D%E5%BA%94%20AFURLSerialization%EF%BC%88%E4%B8%89%EF%BC%89.md#afurlresponseserialization)

