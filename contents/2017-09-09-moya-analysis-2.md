
## 前言

上一篇文章[《Moya 源码分析（一）》](http://piglikeyoung.com/2017/08/27/moya-analysis-1/)分析了 **TargetType**，本文将分析 Moya 的核心 **Provider**

## MoyaProvider

MoyaProvider是Moya的基础，它是你API的端点的管理者。Moya的所有功能都是通过MoyaProvider来使用。

### 定义

MoyaProvider 的定义：

```swift
open class MoyaProvider<Target: TargetType>: MoyaProviderType
```

它使用了泛型，接收一个遵守 **TargetType** 的类型，自己遵守了 **MoyaProviderType**协议。

`Provider真正做的事情可以用一个流来表示：Target -> Endpoint -> Request`

使用Demo的例子来说就是，它将 **GitHubAPI** 转换成 **Endpoint**，再将其转换成 **NSRURLRequest**，最后将这个 **NSRURLRequest** 交给 **Alamofire** 去网络请求。

### 属性

#### EndpointClosure

```swift
/// Closure that defines the endpoints for the provider.
public typealias EndpointClosure = (Target) -> Endpoint<Target>
```

**EndpointClosure** 属性是一个闭包，用于让我们对 **Moya** 生成的 **Endpoint** 进行一些我们自己的定制然后返回一个 **Endpoint** 类，例如：我们想增加一个新的HttpHeader：

```swift
let endpointClosure = { (target: MyTarget) -> Endpoint<MyTarget> in
    let defaultEndpoint = MoyaProvider.defaultEndpointMapping(for: target)
    return defaultEndpoint.adding(newHTTPHeaderFields: ["APP_NAME": "MY_APP"])
}
let provider = MoyaProvider<GitHub>(endpointClosure: endpointClosure)
```

这个闭包输入一个 **target**，返回 **Endpoint**。就是前面说的 `Target -> Endpoint的转换`

> Endpoint 稍后就分析!

#### RequestClosure

```swift
/// Closure that resolves an `Endpoint` into a `RequestResult`.
public typealias RequestClosure = (Endpoint<Target>, @escaping RequestResultClosure) -> Void
```

这个闭包就是实现将`Endpoint -> NSURLRequest`，Moya也提供了一个默认实现：

```swift
/// MoyaProvider+Defaults.swift
public final class func defaultRequestMapping(for endpoint: Endpoint<Target>, closure: RequestResultClosure) {
    do {
        let urlRequest = try endpoint.urlRequest()
        closure(.success(urlRequest))
    } catch MoyaError.requestMapping(let url) {
        closure(.failure(MoyaError.requestMapping(url)))
    } catch MoyaError.parameterEncoding(let error) {
        closure(.failure(MoyaError.parameterEncoding(error)))
    } catch {
        closure(.failure(MoyaError.underlying(error, nil)))
    }
}
```

默认实现也只是简单地调用endpoint.urlRequest取得一个NSURLRequest实例。然后调用了closure。然而，你可以在这里修改这个请求Request, 事实上这也是Moya给你的最后的机会。举个例子, 你想禁用所有的cookie，并且设置超时时间等。那么你可以实现这样的闭包：

```swift
let requestClosure = { (endpoint: Endpoint<GitHub>, done: MoyaProvider.RequestResultClosure) in
    // 可以在这里修改request
    var request: URLRequest = endpoint.urlRequest
    request.httpShouldHandleCookies = false
    request.timeoutInterval = 20 

    done(.success(request))
}

provider = MoyaProvider(requestClosure: requestClosure)
```

#### RequestResultClosure

创建 requestClosure 时，用到了闭包 RequestResultClosure，它的参数类型是 `Result`，这是使用了 **[Result框架](https://github.com/antitypical/Result)**，**用来将throw的方式换成Result<data,error>的方式返回**。

```swift
/// Closure that decides if and what request should be performed.
public typealias RequestResultClosure = (Result<URLRequest, MoyaError>) -> Void
```

从上面可以看出，**EndpointClosure** 和 **RequestClosure** 实现了 Target -> Endpoint -> NSRequest的转换流。

#### StubClosure

```swift
/// Closure that decides if/how a request should be stubbed.
public typealias StubClosure = (Target) -> Moya.StubBehavior
```

这个闭包比较简单，返回一个 **Moya.StubBehavior** 的枚举值。它就是让你告诉Moya你是否使用Stub返回数据或者怎样使用Stub返回数据

```swift
public enum StubBehavior {

    /// 不使用Stub返回数据
    case never

    /// 立即使用Stub返回数据
    case immediate

    /// 一段时间间隔后使用Stub返回的数据
    case delayed(seconds: TimeInterval)
}
```

Never  表明不使用Stub来返回模拟的网络数据， Immediate表示马上返回Stub的数据， Delayed  是在几秒后返回。Moya 默认是不使用Stub来测试。

在 API 中我们给 sampleData 赋值后，这个属性是返回的Stub数据。

```swift
extension AccountAPI: TargetType {
    ...
    var sampleData: NSData {
        switch self {
        case .Login:
            return "{'code': 400, 'Token':'123455'}".dataUsingEncoding(NSUTF8StringEncoding)!
        case .Register(let userName, let passwd):
            return "找不到数据"
        }
    }
}

let endPointAction = { (target: TargetType) -> Endpoint<AccountAPI> in
    let url = target.baseURL.URLByAppendingPathComponent(target.path).absoluteString

    switch target {
    case .Login:
        return Endpoint(URL: url, sampleResponseClosure: {.NetworkResponse(200, target.sampleData)}, method: target.method, parameters: target.parameters)
    case .Register:
        return Endpoint(URL: url, sampleResponseClosure: {.NetworkResponse(404, target.sampleData)}, method: target.method, parameters: target.parameters)
    }
}

let stubAction: (type: AccountAPI) -> Moya.StubBehavior  = { type in
    switch type {
    case .Login:
        return Moya.StubBehavior.Immediate
    case .Register:
        return Moya.StubBehavior.Delayed(seconds: 3)
    }
}
let loginAPIProvider = MoyaProvider<AccountAPI>(
    endpointClosure: endPointAction,
    stubClosure: stubAction
)

loginAPIProvider.request(AccountAPI.Login(userName: "user", passwd: "123456")) { (result) in
    switch result {
    case .Success(let respones) :
        print(respones)

    case .Failure(_) :
        print("We got an error")
    }
    print(result)
}
```

这样就实现了一个 Stub。
Login 和 Register 都使用了Stub返回的数据。

> 注意：Moya中Provider对象在销毁的时候会去Cancel网络请求。为了得到正确的结果，你必须保证在网络请求的时候你的Provider不会被释放。否者你会得到下面的错误 “But don’t forget to keep a reference for it in property. If it gets deallocated you’ll see -999 “cancelled” error on response” 。通常为了避免这种情况，你可以将Provider实例设置为类成员变量，或者shared实例


#### Manager

Moya 并不是一个网络请求的三方库，它只是一个抽象的网络层。它对其他网络库的进行了桥接，真正进行网络请求是别人的网络库（比如默认的Alamofire.Manager） 

Moya 为此做了几件事情：

首先抽象了一个RequestType协议，利用这个协议将Alamofire隐藏了起来，让Provider类依赖于这个协议，而不是具体细节。

```swift
/// Plugin.swift
public protocol RequestType {

    // Note:
    //
    // We use this protocol instead of the Alamofire request to avoid leaking that abstraction.
    // A plugin should not know about Alamofire at all.

    /// Retrieve an `NSURLRequest` representation.
    var request: URLRequest? { get }

    /// Authenticates the request with a username and password.
    func authenticate(user: String, password: String, persistence: URLCredential.Persistence) -> Self

    /// Authenticates the request with an `NSURLCredential` instance.
    func authenticate(usingCredential credential: URLCredential) -> Self
}
```

然后让Moya.Manager == Alamofire.Manager，并且让Alamofire.Manager也实现RequestType协议

```swift
/// Moya+Alamofire.swift
public typealias Manager = Alamofire.SessionManager

/// Choice of parameter encoding.
public typealias ParameterEncoding = Alamofire.ParameterEncoding

/// 让Alamofire.Request也实现 RequestType协议
internal typealias Request = Alamofire.Request
extension Request: RequestType { }
```

上面几步，就完成了Alamofire的封装、桥接。正因为桥接封装了Alamofire, 因此Moya的request,最终一定会调用Alamofire的request。

```swift
/// Moya+Internal.swift
func sendRequest(_ target: Target, request: URLRequest, callbackQueue: DispatchQueue?, progress: Moya.ProgressBlock?, completion: @escaping Moya.Completion) -> CancellableToken {
        let initialRequest = manager.request(request as URLRequestConvertible)
        let alamoRequest = target.validate ? initialRequest.validate() : initialRequest
        return sendAlamofireRequest(alamoRequest, target: target, callbackQueue: callbackQueue, progress: progress, completion: completion)
}
```

在 Moya 的 **sendRequest** 方法中就是用 `manager` 去发送请求（默认是 Alamofire）

如果你想自定义你自己的 Manager, 你可以传入你自己的 Manager 到 Privoder。之后所有的请求都会经过你的这个 Manager 。

```swift
let policies: [String: ServerTrustPolicy] = [
    "example.com": .PinPublicKeys(
        publicKeys: ServerTrustPolicy.publicKeysInBundle(),
        validateCertificateChain: true,
        validateHost: true
    )
]

let manager = Manager(
    configuration: NSURLSessionConfiguration.defaultSessionConfiguration(),
    serverTrustPolicyManager: ServerTrustPolicyManager(policies: policies)
)

let provider = MoyaProvider<MyTarget>(manager: manager)
```

### requestNormal

```swift
func requestNormal(_ target: Target, callbackQueue: DispatchQueue?, progress: Moya.ProgressBlock?, completion: @escaping Moya.Completion) -> Cancellable {
    /// 生成端点
    let endpoint = self.endpoint(target)
    /// 生成测试表现
    let stubBehavior = self.stubClosure(target)
    /// 生成取消请求的Token
    let cancellableToken = CancellableWrapper()

    // Allow plugins to modify response
    /// 通过自定义的插件完善请求回调的闭包
    let pluginsWithCompletion: Moya.Completion = { result in
        let processedResult = self.plugins.reduce(result) { $1.process($0, target: target) }
        completion(processedResult)
    }

    /// 是否最终运行中的请求
    if trackInflights {
        /// 进入原子锁
        objc_sync_enter(self)
        /// 保存请求Endpoint
        var inflightCompletionBlocks = self.inflightRequests[endpoint]
        inflightCompletionBlocks?.append(pluginsWithCompletion)
        self.inflightRequests[endpoint] = inflightCompletionBlocks
        /// 退出原子锁
        objc_sync_exit(self)

        /// 如果有正在运行中的CompletionBlock则取消本次请求，反之则记录本次请求
        if inflightCompletionBlocks != nil {
            return cancellableToken
        } else {
            objc_sync_enter(self)
            self.inflightRequests[endpoint] = [pluginsWithCompletion]
            objc_sync_exit(self)
        }
    }

    /// 准备请求工作
    let performNetworking = { (requestResult: Result<URLRequest, MoyaError>) in
        /// 如果本次请求被取消了则返回
        if cancellableToken.isCancelled {
            self.cancelCompletion(pluginsWithCompletion, target: target)
            return
        }

        var request: URLRequest!
        /// 模式匹配取出request
        switch requestResult {
        case .success(let urlRequest):
            request = urlRequest
        case .failure(let error):
            pluginsWithCompletion(.failure(error))
            return
        }

        // Allow plugins to modify request
        // 通过自定义插件完善请求的闭包
        let preparedRequest = self.plugins.reduce(request) { $1.prepare($0, target: target) }

        let networkCompletion: Moya.Completion = { result in
          if self.trackInflights {
            self.inflightRequests[endpoint]?.forEach { $0(result) }

            objc_sync_enter(self)
            self.inflightRequests.removeValue(forKey: endpoint)
            objc_sync_exit(self)
          } else {
            pluginsWithCompletion(result)
          }
        }

        cancellableToken.innerCancellable = self.performRequest(target, request: preparedRequest, callbackQueue: callbackQueue, progress: progress, completion: networkCompletion, endpoint: endpoint, stubBehavior: stubBehavior)
    }

    requestClosure(endpoint, performNetworking)

    return cancellableToken
}
```

## Endpoint（端点）

通过 MoyaProvider 产生了一个 API Endpoint，然后通过这个 **Endpoint** 发起请求。


```swift
open class Endpoint<Target> {
    public typealias SampleResponseClosure = () -> EndpointSampleResponse

    /// A string representation of the URL for the request.
    open let url: String

    /// A closure responsible for returning an `EndpointSampleResponse`.
    open let sampleResponseClosure: SampleResponseClosure

    /// The HTTP method for the request.
    open let method: Moya.Method

    /// The `Task` for the request.
    open let task: Task

    /// The HTTP header fields for the request.
    open let httpHeaderFields: [String: String]?

    public init(url: String,
                sampleResponseClosure: @escaping SampleResponseClosure,
                method: Moya.Method,
                task: Task,
                httpHeaderFields: [String: String]?) {

        self.url = url
        self.sampleResponseClosure = sampleResponseClosure
        self.method = method
        self.task = task
        self.httpHeaderFields = httpHeaderFields
    }

    /// Convenience method for creating a new `Endpoint` with the same properties as the receiver, but with added HTTP header fields.
    open func adding(newHTTPHeaderFields: [String: String]) -> Endpoint<Target> {
        return Endpoint(url: url, sampleResponseClosure: sampleResponseClosure, method: method, task: task, httpHeaderFields: add(httpHeaderFields: newHTTPHeaderFields))
    }

    /// Convenience method for creating a new `Endpoint` with the same properties as the receiver, but with replaced `task` parameter.
    open func replacing(task: Task) -> Endpoint<Target> {
        return Endpoint(url: url, sampleResponseClosure: sampleResponseClosure, method: method, task: task, httpHeaderFields: httpHeaderFields)
    }

    fileprivate func add(httpHeaderFields headers: [String: String]?) -> [String: String]? {
        guard let unwrappedHeaders = headers, unwrappedHeaders.isEmpty == false else {
            return self.httpHeaderFields
        }

        var newHTTPHeaderFields = self.httpHeaderFields ?? [:]
        unwrappedHeaders.forEach { key, value in
            newHTTPHeaderFields[key] = value
        }
        return newHTTPHeaderFields
    }
}
```

Endpoint 包含着我们这次请求的所有信息。

### Endpoint.urlRequest

```swift
extension Endpoint {
    /// Returns the `Endpoint` converted to a `URLRequest` if valid. Throws an error otherwise.
    public func urlRequest() throws -> URLRequest {
        guard let requestURL = Foundation.URL(string: url) else {
            throw MoyaError.requestMapping(url)
        }

        var request = URLRequest(url: requestURL)
        request.httpMethod = method.rawValue
        request.allHTTPHeaderFields = httpHeaderFields

        switch task {
        case .requestPlain, .uploadFile, .uploadMultipart, .downloadDestination:
            return request
        case .requestData(let data):
            request.httpBody = data
            return request
        case let .requestJSONEncodable(encodable):
            return try request.encoded(encodable: encodable)
        case let .requestParameters(parameters, parameterEncoding):
            return try request.encoded(parameters: parameters, parameterEncoding: parameterEncoding)
        case let .uploadCompositeMultipart(_, urlParameters):
            let parameterEncoding = URLEncoding(destination: .queryString)
            return try request.encoded(parameters: urlParameters, parameterEncoding: parameterEncoding)
        case let .downloadParameters(parameters, parameterEncoding, _):
            return try request.encoded(parameters: parameters, parameterEncoding: parameterEncoding)
        case let .requestCompositeData(bodyData: bodyData, urlParameters: urlParameters):
            request.httpBody = bodyData
            let parameterEncoding = URLEncoding(destination: .queryString)
            return try request.encoded(parameters: urlParameters, parameterEncoding: parameterEncoding)
        case let .requestCompositeParameters(bodyParameters: bodyParameters, bodyEncoding: bodyParameterEncoding, urlParameters: urlParameters):
            if bodyParameterEncoding is URLEncoding { fatalError("URLEncoding is disallowed as bodyEncoding.") }
            let bodyfulRequest = try request.encoded(parameters: bodyParameters, parameterEncoding: bodyParameterEncoding)
            let urlEncoding = URLEncoding(destination: .queryString)
            return try bodyfulRequest.encoded(parameters: urlParameters, parameterEncoding: urlEncoding)
        }
    }
}
```

这段代码可以知道 通过 **urlRequest()** 将 `Endpoint` 转换为一个 `URLRequest`

### Endpoint 比较

```swift
extension Endpoint: Equatable, Hashable {
    public var hashValue: Int {
        let request = try? urlRequest()
        return request?.hashValue ?? url.hashValue
    }

    /// Note: If both Endpoints fail to produce a URLRequest the comparison will
    /// fall back to comparing each Endpoint's hashValue.
    public static func == <T>(lhs: Endpoint<T>, rhs: Endpoint<T>) -> Bool {
        let lhsRequest = try? lhs.urlRequest()
        let rhsRequest = try? rhs.urlRequest()
        if lhsRequest != nil, rhsRequest == nil { return false }
        if lhsRequest == nil, rhsRequest != nil { return false }
        if lhsRequest == nil, rhsRequest == nil { return lhs.hashValue == rhs.hashValue }
        return (lhsRequest == rhsRequest)
    }
}
```

遵守 **Equatable** 和 **Hashable** 协议并实现响应方法即可为自己的类提供==比较方法.这个对比方法用来追踪已经在请求中的request。排除多次无用请求。

Moya 提供了一个默认的 **EndpointClosure** 的函数，来实现这个Target到Endpoint的转换：

```swift
public final class func defaultEndpointMapping(for target: Target) -> Endpoint<Target> {
    return Endpoint(
        url: URL(target: target).absoluteString,
        sampleResponseClosure: { .networkResponse(200, target.sampleData) },
        method: target.method,
        task: target.task,
        httpHeaderFields: target.headers
    )
}
```

### SampleResponseClosure

Endpoint 有一个叫做 SampleResponseClosure 的 Enum，用来返回定制的测试networkResponse。比如特定的StautsCode之类的。

```swift
/// Used for stubbing responses.
public enum EndpointSampleResponse {

    /// The network returned a response, including status code and data.
    case networkResponse(Int, Data)

    /// The network returned response which can be fully customized.
    case response(HTTPURLResponse, Data)

    /// The network failed to send the request, or failed to retrieve a response (eg a timeout).
    case networkError(NSError)
}
```

## 总结

1. **endpointClosure**、**requestClosure**、**stubClosure**，这3个Closure是让我们定制请求、响应和进行测试时的回调，非常有用。

2. **Manager** 是真正用来网络请求的类，Moya 自己并不提供 Manager 类，Moya只是对其他网络请求类进行了简单的桥接。这么做是为了让调用方可以轻易地定制、更换网络请求的库。比如你不想用Alamofire，可以十分简单的换成其他库。

3. **PluginType** 数组。Moya 提供了一个插件机制，使我们可以建立自己的插件类来做一些额外的事情。比如写Log，显示“菊花”等。抽离出Plugin层的目的，就是让Provider职责单一，满足开闭原则。把和自己网络无关的行为抽离。避免各种业务揉在一起不利于扩展。


