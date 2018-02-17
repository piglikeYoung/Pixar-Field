
## 前言
最近忙着做个国外的项目，国外的就需要做字符串本地化了，这里就简单聊聊如何使用国际化。

字符串本地化有两种方法：`修改代码` 或者 `修改xib和storyboard`，这里主要介绍通过代码的方式实现字符串本地化。

## NSLocalizedString
本地化字符串肯定要用到的`NSLocalizedString`这个宏，它是本地化的核心，它还有三个鲜为人知的变体：

* **NSLocalizedStringFromTable**
* **NSLocalizedStringFromTableInBundle**
* **NSLocalizedStringWithDefaultValue**

这些宏最终都调用 **NSBundle** 的 **localizedStringForKey:value:table:** 方法来完成任务。

使用这些宏有两个好处：一方面相比直接调用 **localizedStringForKey:value:table:** 方法，使用宏让代码简单易懂；另一方面，类似 **genstrings** 这样的工具能够监测到这些宏，从而生成供你翻译使用的字符串文件。这些工具会解析 .c 和 .m 后缀的文件，然后为其中每一个需要进行本地化的字符串都生成对应条目，并写入到生成的 .strings 文件中。

## genstrings的使用
为什么要使用这个工具，直接去选择生成需要的语言就好了啊？

主要是因为当你后面增加一个本地化字符串都得去**Localizable.strings**添加相应的key-value，当数量很多的时候很麻烦，所以苹果提供这个这个工具快捷的生成本地化key-value。


**使用方式也很简单：**
1.先cd到工程所在目录，执行命令创建**en.lproj**文件夹

```bash
mkdir en.lproj
```

2.让**genstrings**去检测自己项目中所有的 .m 后缀文件

```bash
find . -name *.m | xargs genstrings -o en.lproj
```
`-o` 选项指定了生成字符串文件的存放目录，默认情况下文件名是 Localizable.strings。需要注意的是，genstrings 默认会覆盖已存在的同名字符串文件。

`-a` 选项可以让 genstrings 将生成的条目追加到已存在同名文件的末尾，而不会覆盖原文件。

不过一般情况下你也许想将生成文件放到另一个目录中，然后使用你喜欢的合并工具将它们与已有文件合并以保留已翻译好的条目。

字符串文件的格式非常简单，都是键值对的形式：

```objc
/* 就是关于我们啦 */
"About Us" = "About Us";
```

**需要注意一点：**NSLocalizedString(key,comment) 用这个函数时，`key不能是宏定义或者一些动态字符串`，否则用上面的命令会报错。把那些不支持的key去掉后就可以编译生成了。

## 字符串键值最佳实践
使用`NSLocalizedString`宏的时候，第一个参数就是为每个特殊字符串指定的键值**（key）**。
这个**Key**如何定义会比较好，并且能够保证它的唯一性，否则在多语言的时候会出现多种意思。

以英文单词「run」为例，作为名词表示「跑步」，作为动词表示「奔跑」，在翻译的时候要加以区别。而且根据上下文的不同，每种具体的译法在文字上可能还会有细微变化。

一个健身应用在不同的地方用到这个单词的不同意思是很正常的，但是如果你使用下面的方法来进行本地化：

```objc
NSLocalizedString(@"Run", nil)
```

无论第二个参数指定了注释内容还是留空，你在字符串文件中都只有一个「run」的条目。而在德语中，「run」作名词时应该译为「Lauf」，作动词时则应该译为「laufen」，或者在特定情况下译为完全不同的形式比如「loslaufen」和「Los geht’s」。

好的键值应该满足两个条件：

* `键值必须在每个具体的上下文中保持唯一性`
* `如果我们没有翻译特定的那个上下文，那么它们不会被其他情况覆盖到而被翻译`

本文推荐使用如下的命名空间方法：

```objc
NSLocalizedString(@"activity-profile.title.the-run", nil)
NSLocalizedString(@"home.button.start-run", nil)
```

这样的键值可以区分应用中不同地方出现的单词，同时提供具体的上下文，比如是标题中的或者按钮中的。上面的例子里我们为了简便忽略了第二个参数，实际使用中如果键值本身没有提供清晰的上下文说明，你可以将进一步的说明作为第二个参数传入。同时请确保键值中只含有 ASCII 字符。

## 分割字符串文件
当需要本地化的内容很多的时候，我们通常会把字符串归类，比如分为Home，Me等等模块，这时候就可以使用`NSLocalizedStringFromTable`这个宏来分类，它有三个参数key、table 和 comment，其中 **table** 参数表示该字符串对应的一个表格，genstrings 会为表中的每一个条目生成一个以条目名称（假设为 table-item）命名的独立字符串文件 **table-item.strings**。

比如：

```objc
NSLocalizedStringFromTable(@"home.button.start-run", @"Home", @"some comment..")
```

还有个简便的方法来设置Table，让本地化更轻松些：

```objc
static NSString * LocalizedHomeString(NSString *key, NSString *comment) {
    return [[NSBundle mainBundle] localizedStringForKey:key value:key table:@"Home"];
}
```
这样就不用每次设置table，不过为了给所有调用此函数的地方生成字符串文件，你要在执行 genstrings 的时候加上 -s 选项：

```bash
find . -name *.m | xargs genstrings -o en.lproj -s LocalizedHomeString
```
`-s` 这个选项指定了本地化函数的`共同前缀名称`，如果你还定义了 **LocalizedHomeString**FromTable，**LocalizedHomeString**FromTableInBundle， **LocalizedHomeString**WithDefaultValue 等函数，以上命令也会调用它们。

## 格式化字符串
我们经常需要在运行时才能最终确定下来字符串的本地化，格式化字符串可以很容易的实现这一点。

以字符串「Run 1 out of 3 completed.」为例，我们可以这样构造格式化字符串:


```objc
NSString *localizedString = NSLocalizedString(@"activity-profile.label.run %lu out of %lu completed", nil);
self.label.text = [NSString localizedStringWithFormat:localizedString, completedRuns, totalRuns];
```

在翻译的时候经常需要对其中的格式化占位符进行顺序调整以符合语法，幸运的是我们可以在字符串文件中轻松地搞定：


```objc
"activity-profile.label.run %lu out of %lu completed" = "Von %2$lu Läufen hast du %1$lu absolviert";
```

## 结论
字符串本地化还包含了很多功能：单复数与阴阳性，字母大小写，格式化数字，格式化日期，调试本地化字符串等等内容，我在这就使用了常用的部分，更多的内容可以浏览 [《ObjC-字符串本地化》](https://objccn.io/issue-9-3/)

## 参考链接
* [字符串本地化](https://objccn.io/issue-9-3/)
* [使用genstrings和NSLocalizedString实现App文本的本地化](http://www.cnblogs.com/every2003/archive/2012/03/20/2407253.html)
* [iOS国际化和genstrings所有子目录本地化字符串](https://my.oschina.net/u/1049180/blog/215695)

