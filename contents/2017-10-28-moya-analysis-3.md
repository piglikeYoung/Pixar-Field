
## 前言

前两篇文章已经将 Moya 的主体思想`Target -> Endpoint -> Request`大致分析了下，本章将说说 Moya 的 **Plugin** 插件机制。

## Plugin

Moya 提供还提供插件机制，你可以自定义各种插件，所有插件必须满足 PluginType 协议：

```swift
public protocol PluginType {
    /// Called to modify a request before sending.
    func prepare(_ request: URLRequest, target: TargetType) -> URLRequest

    /// Called immediately before a request is sent over the network (or stubbed).
    func willSend(_ request: RequestType, target: TargetType)

    /// Called after a response has been received, but before the MoyaProvider has invoked its completion handler.
    func didReceive(_ result: Result<Moya.Response, MoyaError>, target: TargetType)

    /// Called to modify a result before completion.
    func process(_ result: Result<Moya.Response, MoyaError>, target: TargetType) -> Result<Moya.Response, MoyaError>
}

public extension PluginType {
    func prepare(_ request: URLRequest, target: TargetType) -> URLRequest { return request }
    func willSend(_ request: RequestType, target: TargetType) { }
    func didReceive(_ result: Result<Moya.Response, MoyaError>, target: TargetType) { }
    func process(_ result: Result<Moya.Response, MoyaError>, target: TargetType) -> Result<Moya.Response, MoyaError> { return result }
}
```

协议里定义一个网络请求流程里可以处理的事情：
* prepare 准备发起请求
* willSend 开始发起请求
* didReceive 收到请求响应
* process 处理请求结果

Moya 默认提供了四个插件：

AccessTokenAuthorizable插件 (AccessTokenAuthorizable.swift)，HTTP认证的插件。
CredentialsPlugin 插件，URL证书插件。
Logging插件(NetworkLoggerPlugin.swift)，在调试时，输入网络请求的调试信息到控制台
Network Activity Indicator插件（NetworkActivityPlugin.swift）。可以用这个插件来显示网络菊花。


如果要创建自定义插件，请参考[《创建自定义插件》](http://www.hangge.com/blog/cache/detail_1818.html)


