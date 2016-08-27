#UIAppearance简单使用
iOS5及其以后提供了一个比较强大的工具**UIAppearance**，我们通过UIAppearance设置一些UI的全局效果，这样就可以很方便的实现UI的自定义效果又能最简单的实现统一界面风格，它提供如下两个方法。
####appearance
```
+ (id)appearance
```
这个方法是获取全局配置，比如你设置UINavBar的tintColor，你可以这样写：
```
[[UINavigationBar appearance] setTintColor:myColor];
```

####appearanceWhenContainedIn
```
+ (id)appearanceWhenContainedIn:(Class <>)ContainerClass,...
```

这个方法可设置某个类的样式改变：例如：设置UIBarButtonItem 在UINavigationBar、UIPopoverController、UITabbar中的效果。就可以这样写
```
[[UIBarButtonItem appearanceWhenContainedIn:[UINavigationBar class], [UIPopoverController class],[UITabbar class] nil] setTintColor:myPopoverNavBarColor];
```

**注意：使用appearance设置UI效果最好采用全局的设置，在所有界面初始化前开始设置，否则可能失效**

##使用
####修改UINavigationBar样式
```Objective-C
UINavigationBar * appearance = [UINavigationBar appearance];
[appearance setBackgroundImage:[UIImage imageNamed:@"navBg.png"] forBarMetrics:UIBarMetricsDefault];
```

####修改UITabBar样式
```Objective-C
UITabBar *appearance = [UITabBar appearance];
//设置背景图片
[appearance setBackgroundImage:[UIImage imageNamed:@"tabbarBg.png"]];
//设置选择item下划线的背景图片
UIImage * selectionIndicatorImage =[[UIImage imageNamed:@"tabbar_slider"]resizableImageWithCapInsets:UIEdgeInsetsMake(4, 0, 0, 0)] ;
[appearance setSelectionIndicatorImage:selectionIndicatorImage];
```
**如果想深入了解UIAppearance，可以详细阅读我翻译的这篇文章[UIAppearance入门教程](https://github.com/piglikeYoung/Study-notes/blob/master/contents/%E6%8B%BF%E6%9D%A5%E4%B8%BB%E4%B9%89/UIAppearance%E5%85%A5%E9%97%A8%E6%95%99%E7%A8%8B/UIAppearance%E5%85%A5%E9%97%A8%E6%95%99%E7%A8%8B.md)**





