
## 前言
终于放假了，回到家，准备过年，乘着年前把 AFNetworking 源码阅读的最后一篇po出来。

这一篇分析下 AFSecurityPolicy 文件，看看AFNetworking的网络安全策略，特别的是苹果强制要求2017年全部APP支持 HTTPS 后（由于各种原因推迟了），不了解 HTTPS 的可以看下

* [《写给 iOS 开发者看的 HTTPS 指南》](https://autolayout.club/2016/12/22/%E5%86%99%E7%BB%99-iOS-%E5%BC%80%E5%8F%91%E8%80%85%E7%9C%8B%E7%9A%84-HTTPS-%E6%8C%87%E5%8D%97/?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io) 
* [《iOS安全系列之一：HTTPS》](http://oncenote.com/2014/10/21/Security-1-HTTPS/)

AFSecurityPolicy 的主要作用就是验证 HTTPS 请求的证书是否有效，如果 app 中有敏感的信息或者资金流动的信息，一定要使用 HTTPS 来保证交易的信息安全。

## AFSSLPinningMode

首先一眼看到的是 **AFSSLPinningMode** ，验证服务器是否被信任的三种模式：

```objc
typedef NS_ENUM(NSUInteger, AFSSLPinningMode) {
    AFSSLPinningModeNone,
    AFSSLPinningModePublicKey,
    AFSSLPinningModeCertificate,
};
```

* AFSSLPinningModeNone: 表示不做SSL pinning，只跟浏览器一样在系统的信任机构列表里验证服务端返回的证书。若证书是信任机构签发的就会通过，若是自己服务器生成的证书，这里是不会通过的。
* AFSSLPinningModePublicKey: 用证书绑定方式验证，客户端要有服务端的证书拷贝，只是验证时只验证证书里的公钥，不验证证书的有效期等信息。只要公钥是正确的，就能保证通信不会被窃听，因为中间人没有私钥，无法解开通过公钥加密的数据。
* AFSSLPinningModeCertificate: 也是用证书绑定方式验证证书，需要客户端保存有服务端的证书拷贝，这里验证分两步，第一步验证证书的域名/有效期等信息，第二步是对比服务端返回的证书跟客户端返回的是否一致。

## 初始化
使用前，需要先初始化使用哪种验证方式：

```objc
+ (instancetype)policyWithPinningMode:(AFSSLPinningMode)pinningMode {
    return [self policyWithPinningMode:pinningMode withPinnedCertificates:[self defaultPinnedCertificates]];
}

+ (instancetype)policyWithPinningMode:(AFSSLPinningMode)pinningMode withPinnedCertificates:(NSSet *)pinnedCertificates {
    AFSecurityPolicy *securityPolicy = [[self alloc] init];
    securityPolicy.SSLPinningMode = pinningMode;

    [securityPolicy setPinnedCertificates:pinnedCertificates];

    return securityPolicy;
}

```

从证书里面取出公钥，保存在 **pinnedPublicKeys**
```objc
- (void)setPinnedCertificates:(NSSet *)pinnedCertificates {
    _pinnedCertificates = pinnedCertificates;

    if (self.pinnedCertificates) {
        NSMutableSet *mutablePinnedPublicKeys = [NSMutableSet setWithCapacity:[self.pinnedCertificates count]];
        for (NSData *certificate in self.pinnedCertificates) {
            id publicKey = AFPublicKeyForCertificate(certificate);
            if (!publicKey) {
                continue;
            }
            [mutablePinnedPublicKeys addObject:publicKey];
        }
        self.pinnedPublicKeys = [NSSet setWithSet:mutablePinnedPublicKeys];
    } else {
        self.pinnedPublicKeys = nil;
    }
}
```

## AFPublicKeyForCertificate

### 取出公钥

取出公钥的时候，使用了 **AFPublicKeyForCertificate** 函数：

```objc
static id AFPublicKeyForCertificate(NSData *certificate) {
    // 初始化一些临时变量
    id allowedPublicKey = nil;
    SecCertificateRef allowedCertificate;
    SecPolicyRef policy = nil;
    SecTrustRef allowedTrust = nil;
    SecTrustResultType result;
    
    // 因为此处传入的certificate参数是NSData类型的，所以需要使用SecCertificateCreateWithData来将NSData对象转化为SecCertificateRef对象
    allowedCertificate = SecCertificateCreateWithData(NULL, (__bridge CFDataRef)certificate);
    __Require_Quiet(allowedCertificate != NULL, _out);

    // 创建符合X509标准的 SecPolicyRef，然后和证书创建的 SecPolicyRef 比较，确认证书是否值得信任
    policy = SecPolicyCreateBasicX509();
    __Require_noErr_Quiet(SecTrustCreateWithCertificates(allowedCertificate, policy, &allowedTrust), _out);
    __Require_noErr_Quiet(SecTrustEvaluate(allowedTrust, &result), _out);
    
    // 获取证书的公钥，__bridge_transfer 会将结果桥接成 NSObject 对象，然后将 SecTrustCopyPublicKey 返回的指针释放。
    allowedPublicKey = (__bridge_transfer id)SecTrustCopyPublicKey(allowedTrust);

// 释放各种指针
_out:
    if (allowedTrust) {
        CFRelease(allowedTrust);
    }

    if (policy) {
        CFRelease(policy);
    }

    if (allowedCertificate) {
        CFRelease(allowedCertificate);
    }

    return allowedPublicKey;
}
```

### __Require_Quiet

取出公钥的时候用到一个宏，**__Require_Quiet**，它会判断 allowedCertificate != NULL 是否成立，如果 allowedCertificate 为空就会跳到 _out 标签处继续执行：

```objc
#ifndef __Require_Quiet
	#define __Require_Quiet(assertion, exceptionLabel)                            \
	  do                                                                          \
	  {                                                                           \
		  if ( __builtin_expect(!(assertion), 0) )                                \
		  {                                                                       \
			  goto exceptionLabel;                                                \
		  }                                                                       \
	  } while ( 0 )
#endif
```

**__Require_noErr_Quiet** 也是差不多的原理。

## evaluateServerTrust

初始化完成后，就要验证服务器是否能够信任了，调用 **- [AFSecurityPolicy evaluateServerTrust:forDomain:]** 方法

```objc
- (BOOL)evaluateServerTrust:(SecTrustRef)serverTrust
                  forDomain:(NSString *)domain
{
    #1: 不能隐式地信任自己签发的证书

    #2: 设置 policies

    #3: 验证证书是否有效

    #4: 根据 SSLPinningMode 对服务端进行验证
        
    return NO;
}

```

### 不能隐式地信任自己签发的证书

```objc
if (domain && self.allowInvalidCertificates && self.validatesDomainName && (self.SSLPinningMode == AFSSLPinningModeNone || [self.pinnedCertificates count] == 0)) {
        NSLog(@"In order to validate a domain name for self signed certificates, you MUST use pinning.");
        return NO;
    }
```

如果不提供证书或者不验证证书，并且设置 allowInvalidCertificates = YES，满足了所有条件，说明这次验证是不安全的，直接返回 NO。

### 设置 policies 

```objc
NSMutableArray *policies = [NSMutableArray array];
    if (self.validatesDomainName) {
        [policies addObject:(__bridge_transfer id)SecPolicyCreateSSL(true, (__bridge CFStringRef)domain)];
    } else {
        [policies addObject:(__bridge_transfer id)SecPolicyCreateBasicX509()];
    }
```

如果要验证的域名存在，使用域名创建 SecPolicyRef，否则会创建一个符合 X509 标准的默认 SecPolicyRef 对象。

### 验证证书的有效性

```objc
SecTrustSetPolicies(serverTrust, (__bridge CFArrayRef)policies);

if (self.SSLPinningMode == AFSSLPinningModeNone) {
   return self.allowInvalidCertificates || AFServerTrustIsValid(serverTrust);
} else if (!AFServerTrustIsValid(serverTrust) && !self.allowInvalidCertificates) {
   return NO;
}
```

* 如果 SSLPinningMode == AFSSLPinningModeNone 。如果允许无效证书，直接return YES。不允许，跟浏览器一样在系统的信任机构列表里验证服务端返回的证书。

* 如果 SSLPinningMode != AFSSLPinningModeNone，如果服务器信任无效，并且不允许无效证书，return NO。

### 根据 SSLPinningMode 对服务器信任进行验证


```objc
switch (self.SSLPinningMode) {
   case AFSSLPinningModeNone:
   default:
       return NO;
   case AFSSLPinningModeCertificate: {
       
   }
   case AFSSLPinningModePublicKey: {

   }
}
```

**AFSSLPinningModeNone** 直接返回 NO

**AFSSLPinningModeCertificate**

```objc
NSMutableArray *pinnedCertificates = [NSMutableArray array];
// 遍历获取证书
for (NSData *certificateData in self.pinnedCertificates) {
 [pinnedCertificates addObject:(__bridge_transfer id)SecCertificateCreateWithData(NULL, (__bridge CFDataRef)certificateData)];
}
// 为服务器信任设置证书
SecTrustSetAnchorCertificates(serverTrust, (__bridge CFArrayRef)pinnedCertificates);

// 服务器是否能信任
if (!AFServerTrustIsValid(serverTrust)) {
 return NO;
}

// 获取服务器信任的全部证书
NSArray *serverCertificates = AFCertificateTrustChainForServerTrust(serverTrust);

// 遍历，如果有相同的，return YES
for (NSData *trustChainCertificate in [serverCertificates reverseObjectEnumerator]) {
 if ([self.pinnedCertificates containsObject:trustChainCertificate]) {
     return YES;
 }
}
  
return NO;
```

**AFSSLPinningModePublicKey** 

```objc
NSUInteger trustedPublicKeyCount = 0;
NSArray *publicKeys = AFPublicKeyTrustChainForServerTrust(serverTrust);

for (id trustChainPublicKey in publicKeys) {
 for (id pinnedPublicKey in self.pinnedPublicKeys) {
     if (AFSecKeyIsEqualToKey((__bridge SecKeyRef)trustChainPublicKey, (__bridge SecKeyRef)pinnedPublicKey)) {
         trustedPublicKeyCount += 1;
     }
 }
}
return trustedPublicKeyCount > 0;
```
这里只验证公钥，先从服务器信任中获取公钥，如果 pinnedPublicKeys 中的公钥与服务器信任中的公钥相同的数量大于 0，就会 return YES。

## 在 AFURLSessionManager 的使用

在 AFURLSessionManager 中使用 **- (BOOL)evaluateServerTrust:forDomain:** 方法的有两处地方 

* **- URLSession:didReceiveChallenge:completionHandler:**
* **- URLSession:task:didReceiveChallenge:completionHandler:**


```objc
if ([self.securityPolicy evaluateServerTrust:challenge.protectionSpace.serverTrust forDomain:challenge.protectionSpace.host]) {
 disposition = NSURLSessionAuthChallengeUseCredential;
 credential = [NSURLCredential credentialForTrust:challenge.protectionSpace.serverTrust];
} else {
 disposition = NSURLSessionAuthChallengeCancelAuthenticationChallenge;
}
```

NSURLAuthenticationChallenge 表示一个认证的挑战，提供了关于这次认证的全部信息。它有一个非常重要的属性 protectionSpace，这里保存了需要认证的保护空间, 每一个 NSURLProtectionSpace 对象都保存了主机地址，端口和认证方法等重要信息。

调用认证方法后，根据认证的结果，会在 completionHandler 中传入不同的 disposition 和 credential 参数。

## 总结

AFSecurityPolicy 作为一个 AFNetworking 的工具类验证证书的有效性，给 APP 网络安全把关。iOS 全面使用 HTTPS 只是时间早晚的问题。

## 参考链接
* [AFNetworking](https://github.com/AFNetworking/AFNetworking)
* [AFNetworking源码阅读（六）](http://www.cnblogs.com/polobymulberry/p/5174298.html)
* [验证 HTTPS 请求的证书（五）](https://github.com/Draveness/iOS-Source-Code-Analyze/blob/master/contents/AFNetworking/%E9%AA%8C%E8%AF%81%20HTTPS%20%E8%AF%B7%E6%B1%82%E7%9A%84%E8%AF%81%E4%B9%A6%EF%BC%88%E4%BA%94%EF%BC%89.md)



