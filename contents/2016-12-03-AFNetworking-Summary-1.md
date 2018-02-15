
## 前言

先借用大神的话
> 我花费了大量的时间阅读和学习各种开源的代码、研究其中的实现原理、尝试自己实现相关技术、尝试在工作中使用，这使得我在 iOS 开发技术上进步很快。 -- ibireme

我准备开启一个系列文章，会对 **AFNetworking** 的源码进行分析，了解它是如何构建，平时我们是如何使用它发送HTTP请求。

`本系列的AFNetworking版本为：3.1.0`

## 概述

[AFNetworking](https://github.com/AFNetworking/AFNetworking) 是 iOS 和 MacOS 开发中不可或缺的网络请求库，是由无数开源爱好者共同贡献的code形成的开源库。

这是我对 AFNetworking 工程目录架构的归纳，之后会一一展开。

![Snip20170107_1](http://p44bkxib3.bkt.clouddn.com/Snip20170107_1.png)

这篇文章，先介绍**AFNetworking** 的使用。

## NSURLSession

NSURLSession 是 iOS 7之后苹果推出提供网络请求，下载内容的类，提供了丰富的API，并且支持后台下载。

简单的使用：

```objc
NSURLRequest *request = [[NSURLRequest alloc] initWithURL:[[NSURL alloc] initWithString:@"https://github.com"]];
NSURLSession *session = [NSURLSession sharedSession];
NSURLSessionDataTask *task = [session dataTaskWithRequest:request
                                       completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
                                           NSString *dataString = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
                                           NSLog(@"%@", dataString);
                                       }];
[task resume];
```
1. 实例化一个 **NSURLRequest/NSMutableURLRequest**，赋值 URL
2. 通过 **- sharedSession** 获取 NSURLSession 单例
3. 然后调用 **- dataTaskWithRequest:completionHandler:** 方法，会返回一个 **NSURLSessionDataTask** 对象
4. data task 调用 **- resume** 开始执行任务
5. 请求结束后 **completionHandler** 中会返回请求结果 NSData

## AFNetworking

AFNetworking 是对 NSURLSession 的封装，让请求更加简单好用：

```objc
AFHTTPSessionManager *manager = [[AFHTTPSessionManager alloc] initWithBaseURL:[[NSURL alloc] initWithString:@"hostname"]];
[manager GET:@"relative_url" parameters:nil progress:nil
    success:^(NSURLSessionDataTask * _Nonnull task, id  _Nullable responseObject) {
        NSLog(@"%@" ,responseObject);
    } failure:^(NSURLSessionDataTask * _Nullable task, NSError * _Nonnull error) {
        NSLog(@"%@", error);
    }];
```
> iOS 9之后，苹果默认支持 https，如果请求的链接是 http 的，需要在 info.plist 中加入兼容的键值对，才能发出请求

![Snip20170107_3](http://p44bkxib3.bkt.clouddn.com/Snip20170107_3.png)


AFHTTPSessionManager 的 初始化方法 **- initWithBaseURL:**，观察它的调用栈：

```objc
- [AFHTTPSessionManager initWithBaseURL:]
    - [AFHTTPSessionManager initWithBaseURL:sessionConfiguration:]
        - [AFURLSessionManager initWithSessionConfiguration:]
            - [NSURLSession sessionWithConfiguration:delegate:delegateQueue:]
            - [AFJSONResponseSerializer serializer] // 负责序列化响应
            - [AFSecurityPolicy defaultPolicy] // 负责身份认证
            - [AFNetworkReachabilityManager sharedManager] // 查看网络连接情况
        - [AFHTTPRequestSerializer serializer] // 负责序列化请求
        - [AFJSONResponseSerializer serializer] // 负责序列化响应
```
观察调用栈可以看出：
* AFHTTPSessionManager 是 AFURLSessionManager 的子类（它在父类的基础上封装 http 的常用请求：GET，POST，PUT，PATCH，HEAD，DELETE等等）。
* AFURLSessionManager 初始化 **NSURLSession** 实例，同时创建了 **AFJSONResponseSerializer** 来系列化响应，还有 **AFSecurityPolicy** 和 **AFNetworkReachabilityManager** 来保证请求安全和监控网络连接状态
* AFHTTPSessionManager 还有自己的 **AFHTTPRequestSerializer** 和 **AFJSONResponseSerializer** 来序列化请求和响应

接下来观察**GET:parameters:process:success:failure:**的调用栈：

```objc
- [AFHTTPSessionManager GET:parameters:process:success:failure:]
    - [AFHTTPSessionManager dataTaskWithHTTPMethod:parameters:uploadProgress:downloadProgress:success:failure:] // 返回 NSURLSessionDataTask #1
        - [AFHTTPRequestSerializer requestWithMethod:URLString:parameters:error:] // 返回 NSMutableURLRequest
        - [AFURLSessionManager dataTaskWithRequest:uploadProgress:downloadProgress:completionHandler:] // 返回 NSURLSessionDataTask #2
            - [NSURLSession dataTaskWithRequest:] // 返回 NSURLSessionDataTask #3
            - [AFURLSessionManager addDelegateForDataTask:uploadProgress:downloadProgress:completionHandler:]
                - [AFURLSessionManagerTaskDelegate init]
                - [AFURLSessionManager setDelegate:forTask:]
    - [NSURLSessionDataTask resume]
```
* \#1，\#2，\#3 都是通过 NSURLSession 生成的 data task，返回的 data task 调用 **- resume** 方法执行请求。
* 在**addDelegateForDataTask:uploadProgress:downloadProgress:completionHandler:** 方法中，将某些执行的block回调告诉**AFURLSessionManagerTaskDelegate**，比如上传的**uploadProgress**，下载的**downloadProgress**，所有的请求完成的**completionHandler**

## 总结

AFNetworking 其实就是对 NSURLSession 的高度封装提供简单易用的API给开发者使用。

本章只是简单的对 NSURLSession -> AFURLSessionManager -> AFHTTPSessionManager 的简单概述，下一章将会深入了解。

## 参考链接
* [AFNetworking](https://github.com/AFNetworking/AFNetworking)
* [AFNetworking 概述（一）](https://github.com/Draveness/iOS-Source-Code-Analyze/blob/master/contents/AFNetworking/AFNetworking%20%E6%A6%82%E8%BF%B0%EF%BC%88%E4%B8%80%EF%BC%89.md)


