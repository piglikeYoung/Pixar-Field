
## 前言

本篇文章是对 [王巍 (@onevcat)](https://onevcat.com/) 的著作[《Swifter - Swift 必备 Tips》](https://onev.cat/publication/swifter/)的个人阅读总结和代码片段记录。


## 正则表达式

Swift 至今为止并没有在语言层面上支持正则表达式。我们可以使用 Cocoa 中的 **NSRegularExpression** 来做正则匹配。

一个最简单的实现：

```swift
struct RegexHelper {
    let regex: NSRegularExpression
    
    init(_ pattern: String) throws {
        try regex = NSRegularExpression(pattern: pattern, options: .caseInsensitive)
    }
    
    func match(_ input: String) -> Bool {
        let matches = regex.matches(in: input, options: [], range: NSMakeRange(0, input.utf16.count))
        return matches.count > 0
    }
}
```

使用：

```swift
let mailPattern = "^([a-z0-9_\\.-]+)@([\\da-z\\.-]+)\\.([a-z\\.]{2,6})$"

let matcher: RegexHelper
do {
    matcher = try RegexHelper(mailPattern)
}

let maybeMailAddress = "onev@onevcat.com"

if matcher.match(maybeMailAddress) {
    print("有效的邮箱地址")
}
```

> 一个很棒的正则表达式文章[《正则表达式 30 分钟入门教程》](https://deerchao.net/tutorials/regex/regex.htm)，还有[《8个常用正则表达式》](https://code.tutsplus.com/tutorials/8-regular-expressions-you-should-know--net-6149)

现在有了方便的封装，使用 **=~** 来实现：

```swift
precedencegroup MatchPrecedence {
    associativity: none
    higherThan: DefaultPrecedence
}

infix operator =~: MatchPrecedence

func =~(lhs: String, rhs: String) -> Bool {
    do {
        return try RegexHelper(rhs).match(lhs)
    } catch _ {
        return false
    }
}
```

使用：

```swift
if "onev@onevcat.com" =~
"^([a-z0-9_\\.-]+)@([\\da-z\\.-]+)\\.([a-z\\.]{2,6})$" {
    print("有效的邮箱地址")
}
```

## 单例

在 Objective-C 中单例的公认写法类似这样：

```objc
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

Swift 1.2 之后支持了 class 的 static let 和 static var 这样的存储变量后，有个推荐的写法：

```swift
class MyManager  {
    static let shared = MyManager()
    private init() {}
}
```

私有初始化方法，是让项目其他地方不能够通过 **init** 来生成自己的 **MyManager** 实例，保证了类型单例的唯一性。如果你需要的是类似 **default** 的形式的单例（也就是说这个类的使用者可以创建自己的实例）的话，可以去掉这个私有的 **init** 方法。

## 条件编译

Swift 中没有宏定义的概念，因此我们不能使用 **#ifdef** 的方法来检查某个符号是否经过宏定义。为了控制编译流程和内容，Swift 还是提供了几种简答的机制来根据需求定制编译内容。


```
#if <condition>

#elseif <condition>

#else

#endif
```

使用例子：

```
@IBAction func someButtonPressed(sender: AnyObject!) {
    #if FREE_VERSION
        // 弹出购买提示，导航至商店等
    #else
        // 实际功能
    #endif
}
```

**FREE_VERSION** 这个编译符号代表免费版本，我们需要在项目的编一个选项中进行设置，在项目的 Build Settings 中，找到 Swift Compiler - Custom Flags，并在其中的 Other Swift Flags 加上 **-D FREE_VERSION** 就可以了。

## 编译标记

在 Objective-C 中，我们经常插入 **#param** 来标记代码的间距。Swift 中也有类似的标记：

```swift
// MARK:

// TODO:

// FIXME:
```

## 内存管理，weak 和 unowned

**unowned** 更像以前的 **unsafe_unretained**，而 **weak** 就是以前的 **weak**。用通俗的话说，就是 unowned 设置以后即使它原来引用的内容已经被释放了，它仍然会保持对被已经释放了的对象的一个 "无效的" 引用，它不能是 Optional 值，也不会被指向 nil。如果你尝试调用这个引用的方法或者访问成员属性的话，程序就会崩溃。而 weak 则友好一些，在引用的内容被释放后，标记为 weak 的成员将会自动地变成 nil (因此被标记为@weak 的变量一定需要是 Optional 值)。

关于两者使用的选择，Apple 给我们的建议是如果能够确定在访问时不会已被释放的话，尽量使用 unowned，如果存在被释放的可能，那就选择用 weak。

## 值类型和引用类型

Swift 中的 struct 和 enum 定义的类型是值类型，使用 class 定义的为引用类型。很有意思的是，Swift 中的所有的内建类型都是值类型，不仅包括了传统意义像 Int，Bool 这些，甚至连 String，Array 以及 Dictionary 都是值类型的。

在使用数组和字典时的最佳实践应该是，按照具体的数据规模和操作特点来决定到时是使用值类型的容器还是引用类型的容器：在需要处理大量数据并且频繁操作 (增减) 其中元素时，选择 NSMutableArray 和 NSMutableDictionary 会更好，而对于容器内条目小而容器本身数目多的情况，应该使用 Swift 语言内建的 Array 和 Dictionary。

## String 还是 NSString

没有什么特别需要注意的话，尽量使用原生的 **String** 类型。

具体原因查阅原文。

String 唯一一个比较麻烦的地方在于它和 **Range** 的配合。在 NSString 中，我们在匹配字符串的时候通常使用 NSRange 来表征结果或者作为输入。而在使用 String 的对应的 API 时，NSRange 也会被映射成它在 Swift 中且对应 String 的特殊版本：**Range<String.Index>**。这有时候会让人非常讨厌： 

```swift 
let levels = "ABCDE"

let nsRange = NSMakeRange(1, 4)
// 编译错误
// Cannot convert value of type `NSRanve` to expected argument type 'Range<Index>'
levels.replacingCharacters(in: nsRange, with: "AAAA")

let indexPositionOne = levels.index(levels.startIndex, offsetBy: 1)
let swiftRange = indexPositionOne ..< levels.index(levels.startIndex, offsetBy: 5)
levels.replacingCharacters(in: swiftRange, with: "AAAA")
// 输出：
// AAAAA
```

这样太麻烦了，这种情况下，将 **string** 转为 **NSString** 也许是个不错的选择：

```swift
let nsRange = NSMakeRange(1, 4)
(levels as NSString).replacingCharacters(in: nsRange, with: "AAAA")
```

## GCD 和延时调用

GCD 里面有一个很好用的延时调用：

```swift
let time: TimeInterval = 2.0
DispatchQueue.main.asyncAfter(deadline: DispatchTime.now() + time) {
    print("2秒后输出")
}
```
👆代码非常简单，但是还可以稍微封装下，最好加上取消功能：

```swift
typealias Task = (_ cancel : Bool) -> Void

func delay(_ time: TimeInterval, task: @escaping ()->()) ->  Task? {
    
    func dispatch_later(block: @escaping ()->()) {
        let t = DispatchTime.now() + time
        DispatchQueue.main.asyncAfter(deadline: t, execute: block)
    }
    
    var closure: (()->Void)? = task
    var result: Task?
    
    let delayedClosure: Task = {
        cancel in
        if let internalClosure = closure {
            if (cancel == false) {
                DispatchQueue.main.async(execute: internalClosure)
            }
        }
        closure = nil
        result = nil
    }
    
    result = delayedClosure
    
    dispatch_later {
        if let delayedClosure = result {
            delayedClosure(false)
        }
    }
    
    return result;
}

func cancel(_ task: Task?) {
    task?(true)
}
```

使用：

```swift
delay(2) { print("2 秒后输出") }

let task = delay(5) { print("拨打 110") }

// 仔细想一想..
// 还是取消为妙..
cancel(task)
```

## 调用C动态库

生成 **MD5** 密钥：

```swift
// TargetName-Bridging-Header.h
#import <CommonCrypto/CommonCrypto.h>

// StringMD5.swift
extension String {
    var MD5: String {
        var digest = [UInt8](repeating: 0, count: Int(CC_MD5_DIGEST_LENGTH))
        if let data = data(using: .utf8) {
            data.withUnsafeBytes { (bytes: UnsafePointer<UInt8>) -> Void in
                CC_MD5(bytes, CC_LONG(data.count), &digest)
            }
        }
        
        var digestHex = ""
        for index in 0..<Int(CC_MD5_DIGEST_LENGTH) {
            digestHex += String(format: "%02x", digest[index])
        }
        
        return digestHex
    }
}

// 测试
print("swifter.tips".MD5)

// 输出
// dff88de99ff03d109de22fed4f71a273
```

## 输出格式化

在 **Objective-C** 中经常使用 **NSLog** 配合上 **%@** 来输出打印内容到控制台：

```objc
int a = 3;
float b = 1.234567;
NSString *c = @"Hello";
NSLog(@"int:%d float:%f string:%@",a,b,c);
// 输出：
// int:3 float:1.234567 string:Hello 
```

在 Swift 里，我们一般使用 **Print** 来打印：

```swift
let a = 3;
let b = 1.234567  // 我们在这里不去区分 float 和 Double 了
let c = "Hello"
print("int:\(a) double:\(b) string:\(c)")
// 输出：
// int:3 double:1.234567 string:Hello
```

这时候我们常常遇到有些问题，比如打印👆 **b** 中的小数点后两位，在 Objective-C 中使用 **NSLog** 时可以写成：

```objc
NSLog(@"float:%.2f",b);
// 输出：
// float:1.23
```

Swift 的 **Print** 就没这么幸运了。**String** 的格式化初始化方法可以帮助我们利用格式化的字符串：

```swift
let format = String(format:"%.2f", 1.234567)
print("double:\(format)")
// 输出：
// double:1.23
```

每次这么写也很麻烦。如果大量使用的话，还是写个 Double 的扩展：

```swift
extension Double {
    func format(_ f: String) -> String {
        return String(format: "%\(f)f", self)
    }
}

let f = ".2"
print("double:\(1.234567.format(f))")
```

## 数组 enumerate

使用 **NSArray** 时经常遇到的是同时获取值和下标索引，在 Objective-C 中最方便的方式是使用 **enumerateObjectsUsingBlock:** 方法：

```objc
NSArray *arr = @[@1, @2, @3, @4, @5];
__block NSInteger result = 0;
[arr enumerateObjectsUsingBlock:^(NSNumber *num, NSUInteger idx, BOOL *stop) {
    result += [num integerValue];
    if (idx == 2) {
        *stop = YES;
    }
}];

NSLog(@"%ld", result);
// 输出：6
```

停止遍历需要用到 ***stop** 来标记停止，在 Swift 中，这个 API 的 ***stop** 被转换为对应的 **UnsafeMutablePointer<ObjCBool>**：

```swift
let arr: NSArray = [1,2,3,4,5]
var result = 0
arr.enumerateObjects ({ (num, idx, stop) -> Void in
    result += num as! Int
    if idx == 2 {
        stop.pointee = true
    }
})
print(result)
// 输出：6
```

虽然使用 **enumerateObjectsUsingBlock:** 很方便，但是其实从性能上来说这个方法并不理想（具体见原文章），另外它需要使用 **NSArray** 类型，已经不符合 Swift的编码方式了。

在 Swift 上有个很好的替代：

```swift
var result = 0
for (idx, num) in [1,2,3,4,5].enumerated() {
    result += num
    if idx == 2 {
        break
    }
}
print(result)
```

## 小结

以上代码片段全部出自 [王巍 (@onevcat)](https://onevcat.com/) 的著作[《Swifter - Swift 必备 Tips》](https://onev.cat/publication/swifter/)，仅供参考查阅使用，更加详细的内容请参考原著，请支持正版。


