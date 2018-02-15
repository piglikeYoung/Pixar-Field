
## 前言

本篇文章是对 [王巍 (@onevcat)](https://onevcat.com/) 的著作[《Swifter - Swift 必备 Tips》](https://onev.cat/publication/swifter/)的个人阅读总结和代码片段记录。


## delegate

Cocoa 开发中协议-委托是种常用的设计模式，可以说完全离不开它。

👇这段代码在 Swift 中怎么也不让编译通过：

```swift
protocol MyClassDelegate {
    func method()
}

class MyClass {
    weak var delegate: MyClassDelegate?
}

class ViewController: UIViewController, MyClassDelegate {
    // ...
    var someInstance: MyClass!

    override func viewDidLoad() {
        super.viewDidLoad()

        someInstance = MyClass()
        someInstance.delegate = self
    }

    func method() {
        print("Do something")
    }
}

// weak var delegate: MyClassDelegate? 编译错误
// 'weak' cannot be applied to non-class type 'MyClassDelegate 
```

这是因为 Swift 的 **protocol** 是可以被除了 **class** 以外的其他类型遵守的，而对于像 **struct** 或者 **enum** 这样的类型，本身就不能通过引用计数来管理内存，所以也不可能用 **weak** 这样的 ARC 的概念来进行修饰。

想要在 Swift 中使用 weak delegate，我们就需要将 protocol 限制在 class 内。

一种做法是将 protocol 声明为 Objective-C 的，通过在 protocol 前面加上 **@objc** 关键字来达到，Objective-C 的 protocol 都只有类能够实现，因此使用 weak 来修饰就合理了：

```swift
@objc protocol MyClassDelegate {
    func method()
}
```

另一种可能更好的办法是在 protocol 声明的名字后面加上 **class**，这可以为编译器显示地指明这个protocol 只能由 **class** 来实现。

```swift
protocol MyClassDelegate: class {
    func method()
}
```
与添加 **@objc** 相比，后一种方法更能表现出问题的实质，同时也避免了过多的不必要的 Objective-C 兼容，可以说是一种更好的解决方式。

## Associated Object

得益于 Objective-C 的运行时和 Key-Value Coding 的特性，我们可以在运行时向一个对象添加值存储。**而在使用 Category 扩展现有的类的功能的时候，直接添加实例变量这种行为是不被允许的，这时候一般就使用 property 配合 Associated Object 的方式，将一个对象 “关联” 到已有的要扩展的对象上。**进行关联后，在对这个目标对象访问的时候，从外界看来，就似乎是直接在通过属性访问对象的实例变量一样，可以非常方便。

在 Swift 上这样的方法依旧有效，只不过写法上有些不一样：

```swift
// MyClass.swift
class MyClass {
}

// MyClassExtension.swift
private var key: Void?

extension MyClass {
    var title: String? {
        get {
            return objc_getAssociatedObject(self, &key) as? String
        }

        set {
            objc_setAssociatedObject(self,
                &key, newValue,
                .OBJC_ASSOCIATION_RETAIN_NONATOMIC)
        }
    }
}

// 测试
func printTitle(_ input: MyClass) {
	  if let title = input.title {
        print("Title: \(title)")
    } else {
        print("没有设置")
    }
}

let a = MyClass()
printTitle(a)
a.title = "Swifter.tips"
printTitle(a)

// 输出：
// 没有设置
// Title: Swifter.tips”
```

## Lock

只要涉及多线程并发，肯定会遇到锁。

在 Cocoa 和 Objective-C 中加锁的方式有很多，但是其中在日常开发最常用的应该是 **@synchronized**，这个关键字可以用来修饰一个变量，并为其自动加上和解除互斥锁。举个例子：

```objc
- (void)myMethod:(id)anObj {
    @synchronized(anObj) {
        // 在括号内持有 anObj 锁
    }
}
```
👆这个方法虽然简单好用，但是它在 Swift 中已经（或者暂时）不存在了。其实 **@synchronized** 在幕后做的事情是调用了 **objc_sync** 中的 **objc_sync_enter** 和 **objc_sync_exit** 方法，并且加入了一些异常判断。因此，在 Swift 中，如果我们忽略掉那些异常，可以这么写：

```swift
func myMethod(anObj: AnyObject!) {
    objc_sync_enter(anObj)

    // 在 enter 和 exit 之间持有 anObj 锁

    objc_sync_exit(anObj)
} 
```

更进一步，如果我们喜欢以前的那种形式，甚至可以写一个全局的方法，接受一个闭包，来将 **objc_sync_enter** 和 **objc_sync_exit** 封装起来：

```swift
func synchronized(_ lock: AnyObject, closure: () -> ()) {
    objc_sync_enter(lock)
    closure()
    objc_sync_exit(lock)
}
```

结合尾随闭包，使用起来就和 Objective-C 中很像了：

```swift
func myMethod(anObj: AnyObject!) {
    synchronized(anObj){
    		// 在括号内持有 anObj 锁
    }
} 
```


```swift
// 一个实际的线程安全的 setter 例子
class Obj {
    var _str = "123"
    var str: String {
        get {
            return _str
        }
        set {
            synchronized(self) {
                _str = newValue
            }
        }
    // 下略
    }
}
```

## 随机数生成

在 Objective-C 中生成随机数一般使用 **arc4random**，在 Swift 中也可以使用。它会返回一个任意整数，我们想要在某个范围里的数的话，可以做模运算（%）取余数。


```swift
/// 这是错误代码
let diceFaceCount = 6
let randomRoll = Int(arc4random()) % diceFaceCount + 1
print(randomRoll)
```
因为 **arc4random** 返回的值不论在什么平台上都是一个 `UInt32`，于是在 32 位的平台上就有一半几率在进行 Int 转换时越界，时不时的崩溃也就不足为奇了。（iPhone 5 和前任都是32位的 CPU，之后的是64位，Swift 的 Int 和 CPU 架构有关，Int32 或者 Int64）。

相对安全改良版本：

```swift
func arc4random_uniform(_: UInt32) -> UInt32
```

使用：

```swift
let diceFaceCount: UInt32 = 6
let randomRoll = Int(arc4random_uniform(diceFaceCount)) + 1
print(randomRoll)
```

最佳实践是创建个 **Range** 的随机数的方法：

```swift
func random(in range: Range<Int>) -> Int {
    let count = UInt32(range.upperBound - range.lowerBound)
    return Int(arc4random_uniform(count)) + range.lowerBound
}

for _ in 0...100 {
    let range = Range<Int>(1...6)
    print(random(in: range))
}
```

## 断言

断言的另一个优点是它是一个开发时的特性，只有在 Debug 编译的时候有效，而在运行时是不被编译执行的，因此断言并不会消耗运行时的性能。这就不用每次代码发布时，一一找出去除。

在 Release 版本强制启用断言或者在 Debug 版本强制禁用断言配置，请参考原文，不推荐这样修改。

如果我们需要在 Release 发布时在无法继续时将程序强行终止的话，应该选择使用 `fatalError`。

## Playground 延时运行

为了让 playground 具有延迟运行的能力，我们需要这么做：

```swift
import PlaygroundSupport

PlaygroundPage.current.needsIndefiniteExecution = true
```
默认情况下它会在顶层代码最后一句运行后20秒的时候停止执行。这个时间长度满足大部分需求了，如果你想要改变这个时间的话，可以通过 `Alt + Cmd + 回车` 来打开辅助编译器，在那里修改。

## Log 输出

在 Objective-C 中，我们常用 **NSLog** 来打印输出控制台，通常只会在 **Debug** 版本打印，**Release** 版本为了性能会屏蔽掉（因为也没人看）：

```objc
#ifdef DEBUG
    #define DPrintf(fmt, ...)  printf("📍%s + %d🎈 %s\n", __PRETTY_FUNCTION__, __LINE__, [[NSString stringWithFormat:fmt, ##__VA_ARGS__]UTF8String])
#else
    #define DPrintf(fmt, ...)
#endif
```

在 Swift 中，编译器也为我们准备了几个很有用的编译符号：

* \#file (String) 包含这个符号的文件的路径
* \#line (Int) 符号出现处的行号
* \#column (Int) 符号出现处的列
* \#function (String) 包含这个符号的方法名

使用：

```swift
func printLog<T>(_ message: T,
                    file: String = #file,
                  method: String = #function,
                    line: Int = #line)
{
    print("\((file as NSString).lastPathComponent)[\(line)], \(method): \(message)")
}
```

当需要在 Release 版本关闭输出时：

```swift
func printLog<T>(_ message: T,
                    file: String = #file,
                  method: String = #function,
                    line: Int = #line)
{
    #if DEBUG
    print("\((file as NSString).lastPathComponent)[\(line)], \(method): \(message)")
    #endif
}
```

新版本的 LLVM 编译器在遇到这个空方法时，甚至会直接整个方法去掉，完全不去调用它，从而实现零成本。

## 属性访问控制

Swift 由低到高提供了 **private**，**fileprivate**，**internal**，**public** 和 **open** 五种访问控制的权限。默认是 **internal** 权限。

private 让代码只能在当前作用域或者同一文件中同一类型的作用域中被使用，fileprivate 表示代码可以在当前文件中被访问，而不做类型限定。

public 和 open 的区别在于，只有被 open 标记的内容才能在别的框架中被继承或者重写。

在开发中的时候，我们给某属性加上了 private 修饰符了，但是希望在类型之外也能够读取到这个类型，同时为了保证类型的封装和安全，只能在类型内部对其进行改变和设置：

```swift
class MyClass {
    private(set) var name: String?
}
```

这样 set 被限制为 private，保证外部只能访问不能修改。

这种写法没有对读取做限制，相当于使用了默认的 internal 权限。如果我们希望在别的 module 中也能访问这个属性，同时又保持只在当前作用域可以设置的话，我们需要将 get 的访问权限提高为 public。属性的访问控制可以通过两次的访问权限指定来实现，具体来说，将刚才的声明变为：

```swift
public class MyClass {
    public private(set) var name: String?
}
```
> MyClass 也要加 public 是因为其他 module 连 MyClass 都不能访问的话，怎么访问 name

## 小结

以上代码片段全部出自 [王巍 (@onevcat)](https://onevcat.com/) 的著作[《Swifter - Swift 必备 Tips》](https://onev.cat/publication/swifter/)，仅供参考查阅使用，更加详细的内容请参考原著，请支持正版。


