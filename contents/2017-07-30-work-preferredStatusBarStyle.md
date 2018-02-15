## 前言

最近对公司 iOS 项目进行版本升级，最低支持 iOS 10，升级后有很多 API 已经过期了，其中就包括 **statusBarStyle**：

```objc
@property(readwrite, nonatomic) UIStatusBarStyle statusBarStyle NS_DEPRECATED_IOS(2_0, 9_0, "Use -[UIViewController preferredStatusBarStyle]") __TVOS_PROHIBITED;
```

根据提示使用 **preferredStatusBarStyle** 方法代替时，发现时而有效时而无效，非常纠结！

```objc
- (UIStatusBarStyle)preferredStatusBarStyle {
    return UIStatusBarStyleLightContent;
}
```

## 解决方法

去网上Google了好久才发现原因：你的ViewController有可能嵌套到了 **UINavigationController** 或者 **UITabBarController** 中。

系统会从 **window** 的 **rootViewController** 开始层层确认 **preferredStatusBarStyle**。

所以需要从父类着手处理，创建创建一个 **UINavigationController** 或者 **UITabBarController** 的子类，然后重写 **childViewControllerForStatusBarStyle** 方法：

```objc
- (UIViewController *)childViewControllerForStatusBarStyle {
    return self.selectedViewController;
}
```

使用子类化的容器后，系统就会根据返回的 ViewController 的 **preferredStatusBarStyle** 方法来处理 status bar 的 style 了。

以此类推 Status Bar Hidden 也是通过这种方式奏效：

```objc
- (UIViewController *)childViewControllerForStatusBarHidden {
    return self.selectedViewController;
}
```

还有个情况，当present一个ViewController时，**preferredStatusBarStyle** 又不工作了，这时候你需要在present前设置：

```objc
vc.modalPresentationStyle = UIModalPresentationCustom;
vc.modalPresentationCapturesStatusBarAppearance = YES;
```
这样present出来的VC的 Status Bar 才生效。

系统还提供了一个刷新 Status Bar 的方法，当需要强制刷新时，可以调用 **setNeedsStatusBarAppearanceUpdate** 来刷新。


