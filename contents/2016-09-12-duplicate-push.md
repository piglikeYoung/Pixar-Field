
## 前言
相信不少iOS开发者都遇到过点击`tableView(:didSelectRowAtIndexPath:)`push到别的界面，有些用户点击了cell后，可能会触发两次push，弹出两次别的界面。这种情况出现的概率比较小，在***快速双击*** 或者 ***长按cell稍微放开又按下***的情况下比较容易复现。

为此去Google解决方案，找到这篇文章[《防止点击 Cell 时 ViewController 被重复 Push》](https://github.com/nixzhu/dev-blog/blob/master/2016-01-04-duplicate-push.md)，作者[@nixzhu](https://twitter.com/nixzhu)

然后结合我的实际情况作出解决并摘录了一些原文内容。

既然push了两次，那么`tableView(:didSelectRowAtIndexPath:)`一定被调用了两次。但又因为这是一个delegate方法，是由iOS来调用的，怎样才算被“选中”实在难以猜想，何况不同的 iOS 版本还可能有差异。不过我们仍然有办法做处理，毕竟它被调用了两次，就算是用**计数判断**的办法，我们也能防止同一次点击出现重复的 push。真实的问题是这种情况难以调试，因为问题很不容易出现，你怎么知道修改后的代码是工作的呢？

## 解决
原文章就给出了解决方案：

```swift
func tableView(tableView: UITableView, didSelectRowAtIndexPath indexPath: NSIndexPath) {
    defer {
        tableView.deselectRowAtIndexPath(indexPath, animated: true)
    }

    if let navigationController = navigationController {
        guard navigationController.topViewController == self else {
            return
        }
    }

    // TODO: performSegueWithIdentifier...
}
```
这个代码是利用了 push 后，navigationController 的`ViewControllers stack` 会发生改变，那么其`topViewController`自然会发生改变。**所以在 push 之前，我们先保证此时的 topViewController 是当前 ViewController，这样就可避免重复 push，因为你第二次 push ，topViewController就变成了第一次 push 后的 ViewController 就会 return 掉**。

原作者还用了 defer 关键字，也就是无论如何，我们都可以取消 cell 的选中。当然，我们还应该判断 navigationController 的存在性，因为当前的 ViewController 不一定内嵌在某个 UINavigationController 里，segue 也并非只有 push 一种，还有show。

### 如何重现问题？
我自己是快速点击两次cell，原作者是长按 cell，然后稍微减少手指的压力再立即增加压力，触发这个bug。（估计是UI业务逻辑有关吧，如果你没有观察到两次 push 间 viewControllers stack 的改变，那说明你遇到的问题更加诡异，此方法也可能不适用。）

我通过打印的办法确认，在出现重复 push 的情况下，iOS 9.2 会调用两次 `tableView(:didSelectRowAtIndexPath:)`，但在此之间，第一次 performSegue 时，UINavigationController 的 viewControllers stack 就已经被改变了，也就是说，第二次就不会再 push 了。

既然只要 performSegue 就能改变 UINavigationController 的 viewControllers stack，那我们也不需要在 tableView(:didSelectRowAtIndexPath:) 里做处理，直接重载 ViewController 的 `performSegueWithIdentifier(:sender:)` 即可：

```objc
override func performSegueWithIdentifier(identifier: String, sender: AnyObject?) {
    if let navigationController = navigationController {
        guard navigationController.topViewController == self else {
            return
        }
    }

    super.performSegueWithIdentifier(identifier, sender: sender)
}
```

这样代码逻辑会更好一点。进一步，若你的 app 里有许多 ViewController，也许它们都有可能出现重复 push。那为了避免重复代码，可以写一个基类让其他 ViewController 继承。

**注意**，你可能会想说，怎么不在 `shouldPerformSegueWithIdentifier(:sender:)` 里做处理呢？不是更合理吗？

这个方法看起来挺像这么回事，但可惜，`它只会作用于直接在 IB 里连线的 segue，如果你手动 performSegue，它并不会被调用`。这样的逻辑也说得通，毕竟你都手动调用了，哪还有shouldPerformSegue（该不该跳转）的问题呢？

## 我的情况
我的cell点击后的情况比较特殊：点击cell后调用异步网络请求，根据请求返回确定cell是否跳转，这又是个bug，因为请求是异步的，当你第一次请求还没回来没有 push （viewControllers stack未改变），第二次点击请求回来了并且已经 push ，这时第一次请求回来了又 push 一次，这样原作者的办法就无效了。

两种方法解决：

* 点击cell后增加个小菊花loading，让用户在请求回来前不能响应别的事件
* 增加个Bool变量isClickCell，默认是NO。每次点击cell前判断是否为YES，如果是YES直接return，反之isClickCell赋值为YES，然后异步网络请求，网络请求不论成功或者失败都赋值为NO，让用户下次可以点击cell

**这两种解决方法都不算特别突出，如果你有更好的解决办法，可以给我留言，我们交流交流。**

## 参考链接
[《防止点击 Cell 时 ViewController 被重复 Push》](https://github.com/nixzhu/dev-blog/blob/master/2016-01-04-duplicate-push.md)

