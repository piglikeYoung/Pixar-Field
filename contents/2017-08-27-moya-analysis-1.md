
## 前言

由于现在业务需要，新项目将使用 Swift 语言进行开发，需要进行技术预言，APP 的开发最重要的部分就是网络请求了。iOS 中可以使用 **URLSession** 进行网络请求，但是方便起见，我通常会选择使用 [**Alamofire**](https://github.com/Alamofire/Alamofire) 这样的著名第三方库。

而 [**Moya**](https://github.com/Moya/Moya) 又是基于 **Alamofire** 的更高一层的网络请求封装抽象层，我们可以直接操作 **Moya**，然后 **Moya** 去管理请求，而不用直接和 **Alamofire** 进行接触。

> 本文不会详细指导如何使用 Moya ，如果你还没使用过 Moya 建议你去下载Demo看看，或者去查看它的温度。

## 结构

本文是基于 **Moya 10.0.1**  进行分析的。
![WechatIMG32](http://p44bkxib3.bkt.clouddn.com/WechatIMG32.jpeg)

上图是 Moya 的文件结构，接下来将对其主要部分进行分析。

## Target

发出网络请求之前，一般是先建立一个 **Enum** 的 **Target**，它定义了网络请求相关行为，**Target** 必须实现 **TargetType** 协议。

### TargetType.swift
```swift
public protocol TargetType {

    /// The target's base `URL`.
    var baseURL: URL { get }

    /// The path to be appended to `baseURL` to form the full `URL`.
    var path: String { get }

    /// The HTTP method used in the request.
    var method: Moya.Method { get }

    /// Provides stub data for use in testing.
    /// 用于单元测试的测试数据，只会在单元测试文件中有作用
    var sampleData: Data { get }

    /// The type of HTTP task to be performed.
    /// request、upload、download
    var task: Task { get }

    /// A Boolean value determining whether the embedded target performs Alamofire validation. Defaults to `false`.
    /// 是否执行Alamofire校验 ，一般有手机号、邮箱号校验之类的
    var validate: Bool { get }

    /// The headers to be used in the request.
    /// 请求头
    var headers: [String: String]? { get }
}

public extension TargetType {
    /// Defaults to `false`.
    /// 默认不执行Alamofire校验
    var validate: Bool {
        return false
    }
}
```

协议很简单，都是网络请求的基础参数，其中有两个有意思的参数：**SampleData** 和 **validate**

**SampleData**，在我们进行单元测试的时候自动返回设置的测试数据，这样在服务器接口没有完成的情况下也能调用网络请求。

**validate**，以前我们想要验证参数的合法性，可能需要这么做：
* 在发送请求前先校验，不合法 return 并弹出提示框
* 写一个 Block：(参数)->(校验结果) 当做一个参数传进 Request 方法中

而在 Moya 中，简单很多：

```swift
public var validate: Bool {
    switch self {
    case .login(params):
        let phoneNumber = params.phoneNumber
        return phoneNumber.count == 11
    default:
        return false
    }
}
```

#### 通过Enum来管理APIs

可能你们有疑问，返回的值都是固定的，是不是每个API都要创建一个实例遵守 **TargetType** 协议，然后创建一个基类 baseAPI 让其他API继承减少重复代码？

**肯定是不需要基类的**，可以仿照 **TargetType** 返回默认值的扩展，给每个参数返回默认值，这样之后每个API实现了协议，没有赋值的均使用默认值：

```Swift
public extension TargetType {
    
    // 服务器地址
    public var baseURL: URL {
        return ServerHelper.getServerBasePath()
    }
    
    public var path: String {
        return ""
    }
    
    // 请求类型
    public var method: Moya.Method {
        return .post
    }
    
    // 是否执行Alamofire验证
    public var validate: Bool {
        return false
    }

    public var sampleData: Data {
        return "{}".data(using: String.Encoding.utf8)!
    }

    public var headers: [String: String]? {
        return nil
    }
}
```

Demo 中提供了一个GithubAPI的模块，通过使用Enum的关联值来表示各个API的具体参数：

```swift
public enum GitHub {
    case zen
    case userProfile(String)
    case userRepositories(String)
}
```

这么简便的使用方式，都需要归功于 Swift 强大的 Enum，如果不是很清楚可以参考这篇文章[《Swift 中枚举高级用法及实践》](http://swift.gg/2015/11/20/advanced-practical-enum-examples/)

## MultiTarget.swift

Moya 中还有个 **MultiTarget.swift** 的文件，它是个遵守了 **TargetType** 协议的 Enum，也重写了协议的每个属性，但是它只有一个Case：

```swift
/// The embedded `TargetType`.
case target(TargetType)

/// Initializes a `MultiTarget`.
public init(_ target: TargetType) {
    self = MultiTarget.target(target)
}
    
/// The embedded `TargetType`.
public var target: TargetType {
    switch self {
    case .target(let t): return t
    }
}
```
它的初始化就是传入一个API枚举参数，比如 **GitHub.userRepositories(username)**。

之前我们都是一个 **target** 对应一个 **Provider**。有时我们程序会根据业务逻辑拆分成多个 **target**，这样 **target** 可能就会有很多个，如果有多少个 **target** 我们就创建多少个 **Provider**，会让应用程序的逻辑复杂化。特别是当它们使用同样的 **plugins** 或 **closures** 时，又要做一些额外的工作去维护。

### Provider 定义
之前一一对应的Privider：

```Swift
let giphyProvider = MoyaProvider<Giphy>()

let gitHubProvider = MoyaProvider<GitHub>()
```

现在使用多个 **target** 的 Privider：

```swift
let provider = MoyaProvider<MultiTarget>()
```

### Provider 使用
之前一一对应的Privider：

```swift
giphyProvider.request(.upload) { result in
    // do something with `result`
}
 
gitHubProvider.request(.zen) { result in
    // do something with `result`
}
```

现在使用多个 **target** 的 Privider：

```swift
provider.request(MultiTarget(Giphy.upload)) { result in
	
}

provider.request(MultiTarget(GitHub.zen)) { result in
            
}
```

## 参考文档
* [Swift 中枚举高级用法及实践](http://swift.gg/2015/11/20/advanced-practical-enum-examples/)
* [Swift - 网络抽象层库Moya的使用详解7(多个target使用同一个Provider)](http://www.hangge.com/blog/cache/detail_1817.html)

