
## 前言
每种开发语言都有判断对象不存在的时候，最基础的C语言是通过“0”来判断“不存在”，而用 **NULL** 作为指针空值。在 OC 中有`nil` / `Nil` / `NULL` / `NSNull`。下面我们分别来看看这几种空值的区别。

## nil
Objective-C在C的表达不存在的基础上增加了nil。nil是一个指向不存在的对象指针。

在 objc.h 中的定义：

```objc
#ifndef nil
# if __has_feature(cxx_nullptr)
#   define nil nullptr
# else
#   define nil __DARWIN_NULL
# endif
#endif
```
判断是否有 **cxx_nullptr** 特性，如果有， nil 定义为 **nullptr**，否则定义为 **__DARWIN_NULL**。

在 _types.h 中的定义：

```objc
#define __DARWIN_NULL ((void *)0)
```

本质上 nil 是 **(void *)0**。

### 使用场景
nil 一般用于指向Objective-C对象，表示对象为空，例如：

```objc
NSObject *obj = nil;
id onObj = nil;
if (obj2){...}
```

nil 最显著的行为是，它虽然为零，仍然可以有消息发送给它。在别的语言里，比如Java，调用空对象的方法会崩溃，但是在Objective-C中，即使nil为“零”，你也可以调用它的方法，这样可以简化非空判断：

```objc
// 举个例子，这个表达...
if (name != nil && [name isEqualToString:@"Steve"]) { ... }

// ...可以被简化为：
if ([name isEqualToString:@"steve"]) { ... }
```

## Nil
在 objc.h 中定义：

```objc
#ifndef Nil
# if __has_feature(cxx_nullptr)
#   define Nil nullptr
# else
#   define Nil __DARWIN_NULL
# endif
#endif
```
与 `nil` 基本一致，本质上也是 **(void *)0**。

### 使用场景
Nil 是 Objective-C 类类型的书面空值，对应 Class 类型对象。

```objc
Class someClass = Nil;
Class anotherClass = [NSString class];
```

## NULL
在 _null.h 中的定义：

```objc
#ifndef NULL 
#define NULL  __DARWIN_NULL
#endif  /* NULL */
```

那么本质上 NULL 也是 **(void *)0**。

### 使用场景
NULL 一般用于表示 C 指针空值，例如：

```objc
int *pointerToInt = NULL;
char *pointerToChar = NULL;
struct TreeNode *rootNode = NULL;
```

## NSNull
看到 **NS** 就可以判断它是  Objective-C 的一个类，代表空值的类。它定义在 NSNull.h 文件里：

```objc
#import <Foundation/NSObject.h>

NS_ASSUME_NONNULL_BEGIN

@interface NSNull : NSObject <NSCopying, NSSecureCoding>

+ (NSNull *)null;

@end

NS_ASSUME_NONNULL_END
```

NSNull 只有一个单例方法 +[NSnull null]，一般用于在集合对象中保存一个空的占位对象。

### 使用场景
在 Foundation 集合对象（NSArray、NSDictionary、NSSet 等）中， nil 通常被用于表示集合对象结束的标志，因此无法用 nil 来存储一个空值，所以一般用 [NSNull null] 空对象来存储。另外，在 NSDictionary 的 -objectForKey: 方法中，如果当前字典中 key 对应的值不存在时，该方法会返回 nil，表明当前 key 在字典中未添加，但是如果我们想明确表示某一 key 已经在字典中添加，但是它没有值，这时候就可以用 [NSNull null] 来赋值表示。

```objc
// 当 NSArray 里遇到 nil 时，就说明这个数组对象的元素截止了，即 NSArray 只关注 nil 之前的对象，nil 之后的对象会被抛弃。
NSArray *array = [NSArray arrayWithObjects:@"one", @"two", nil];
 
// 错误的使用
NSMutableDictionary *dict = [NSMutableDictionary dictionary];
[dict setObject:nil forKey:@"someKey"];
 
// 正确的使用
NSMutableDictionary *dict = [NSMutableDictionary dictionary];
[dict setObject:[NSNull null] forKey:@"someKey"];
```

## NIL 或 NSNil

**Objective-C 中不存在这两个符号！！！**

## 总结
从上述分析我们可知，不管是 NULL、nil 还是 Nil，它们本质上是一样的，都是 (void *)0，只是写法不同。这样做的意义是为了区分不同的数据类型，虽然它们值相同，但我们需要理解它们之间的字面意义并用于不同场景，让代码更加明确，增加可读性。

| 标志 | 值 | 含义 |
| :------------ | :--------------- | :----- |
| NULL | (void *)0 | C指针的字面零值 |
| nil | (id)0 | Objective-C对象的字面零值 |
| Nil | (Class)0 | Objective-C类的字面零值 |
| NSNull | [NSNull null] | 用来表示零值的单独的对象 |

## 参考链接
* [nil / Nil / NULL / NSNull](http://nshipster.cn/nil/)
* [Objective C 中的nil，Nil，NULL和NSNull理解](http://magicalboy.com/null-value-in-objective-c/)
* [nil/Nil/NULL/NSNull的区别](http://blog.csdn.net/wzzvictory/article/details/18413519)
* [Objective-C 中 NULL、nil、Nil、NSNull 的定义及不同](https://kangzubin.cn/null-and-nil-in-objective-c/#more)


