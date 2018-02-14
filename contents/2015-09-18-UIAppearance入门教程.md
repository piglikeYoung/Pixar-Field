原文：[《UIAppearance Tutorial: Getting Started》](http://www.raywenderlich.com/108766/uiappearance-tutorial)

译者注：这篇文章是使用UIAppearance定义UI界面的入门教程

## 序言
虽然iOS的拟物设计已经成为了过去式，但是这并不代表你的iOS App的控件的外观被限制在系统自有的那几个。
当然，你可以重新定义自己的控件和App的外观，在这个时候，Apple推荐开发者使用标准的UIKit控件并利用iOS提供的各种自定义的方法来达到你的效果。这样做的原因是因为UIKit控件运行效率非常高， 另外你的这些自定义的控件也会在未来的版本中表现更好的兼容性。
在这个UIAppearance的教程中，你会用一些基本的UI自定义技术美化一款寻找宠物的App使它由平常变得与众不同。

## 开篇
[点击这里](http://cdn1.raywenderlich.com/wp-content/uploads/2015/06/Pet_Finder_Starter.zip)下载本教程的所用到的源码。这个App使用了很多标准UIKit控件，配色是默认的vanilla。

打开这个项目，然后看看它的结构，编译运行后，宠物查找器的主界面是这样子的：
![plain-600x500](http://p44bkxib3.bkt.clouddn.com/plain-600x500.png)


APP有一个导航栏和一个标签栏。主界面显示了宠物列表；点击一个宠物会显示他的详情。同时这里有一个查找界面。一个可以让用户选择主题的界面好像是个不错的主意，我们可以从这里开始。

## 支持主题
很多Apps并不能让用户选择主题，在大多数时候也并不推荐开发者去发布一个具有主题选择功能的App。如果你的app显示的内容没有控制好，你会很快发现你的App中的主题相互冲突。但是，你也许在开发阶段会想对不同
的主题进行测试来看看哪个更适合你的App,或者做一个A/B 测试来看Beta版本的用户更喜欢哪种主题。

在这个UIAppearance 教程中，你会创建几个不同的主题，然后可以看看哪个更加符合你的审美。

选择 File\New\File…然后选择 iOS\Source\Swift File。 点击Next然后将文件命名为Theme。最后点击Next并点击Create，Xcode会自动打开你创建的新文件，这个文件里只有一行代码。

删除这行代码并且用如下代码替换：
```swift
import UIKit
 
enum Theme : Int {
  case Default, Dark, Graphical
 
  var mainColor: UIColor {
    switch self {
    case .Default:
      return UIColor(red: 87.0/255.0, green: 188.0/255.0, blue: 95.0/255.0, alpha: 1.0)
    case .Dark:
      return UIColor(red: 242.0/255.0, green: 101.0/255.0, blue: 34.0/255.0, alpha: 1.0)
    case .Graphical:
      return UIColor(red: 10.0/255.0, green: 10.0/255.0, blue: 10.0/255.0, alpha: 1.0)
    }
  }
}
```
这段代码为你的App添加了一个包含不同的主题的枚举值。从现在起，所有的主题只有特定一个mainColor

然后，添加下面的struct:
```swift
struct ThemeManager {
 
}
```

这段代码让你可以在App中使用主题。它暂时是空的，但是马上会对他进行修改。

然后，将下面这行代码添加到enum的声明前。
```swift
let SelectedThemeKey = "SelectedTheme"
```

现在，给ThemeManager添加如下方法：
```swift
static func currentTheme() -> Theme {
  if let storedTheme = NSUserDefaults.standardUserDefaults().valueForKey(SelectedThemeKey)?.integerValue {
    return Theme(rawValue: storedTheme)!
  } else {
    return .Default
  }
}
```
这儿也没有太过复杂的代码：这是你将用到的控制app的样式的主方法。它使用NSUserDefaults来持久化存储你的当前主题，每次启动App时它都会被读取。

要测试这个是否可用，可以打开AppDelegate.swift然后将下面代码添加到 application(_:didFinishLaunchingWithOptions):
```swift
println(ThemeManager.currentTheme().mainColor)
```

Build and run。你应该会在控制台看到下面的输出结果：
```swift
UIDeviceRGBColorSpace 0.94902 0.396078 0.133333 1
```
那么从现在起，你已经有三个主题，并且你可以通过ThemeManager对他们进行管理。现在是时候在App中使用他们了。

## 主题添加到控件
回到Theme.swift，添加下面的方法到ThemeManager:
```swift
static func applyTheme(theme: Theme) {
  // 1 
  NSUserDefaults.standardUserDefaults().setValue(theme.rawValue, forKey: SelectedThemeKey)
  NSUserDefaults.standardUserDefaults().synchronize()
 
  // 2
  let sharedApplication = UIApplication.sharedApplication()
  sharedApplication.delegate?.window??.tintColor = theme.mainColor
}
```
这里我们过一遍上面的代码：

1.先使用NSUserDefaults持久化存储选中的主题

2.然后获取你选择的主题并且将主题的main color赋给 application’s window的tintColor 属性。关于tintColor稍后会详细讲解

然后你需要做的就是调用这个方法，在AppDelegate.swift中调用。
将之前加的 println()语句替换成下面的代码：
```swift
let theme = ThemeManager.currentTheme()
ThemeManager.applyTheme(theme)
```

Build and run。你会发现你的App明显看起来偏绿色调多了：

![theme_applied1](http://p44bkxib3.bkt.clouddn.com/theme_applied1.png)

在App中随便点点；会发现各种地方都已经是这种颜色了。但是你并没有在controller或者view上面做任何修改，这个绿魔法到底是什么呢？:]

## 使用 Tint Colors
从iOS7开始，UIView 就已经暴露了tintColor属性，一般这个属性被用在app的主色调选择器和某些可交互的界面元素的交互状态。
如果你为某个view指定一个tint，那么这个tint将会自动应用到这个view的所有的子view中，因为UIWindow继承于UIView，你可以通过设定window的tintColor来定义整个app的色彩。


点击左上角的齿轮图标；一个有分段控制的 table view将会显示出来，但是当你选择不同的主题并提交时，没有东西会发生改变，是时候修改它了。
打开SettingsTableViewController.swift 然后添加下面的代码到 applyTheme()，就在dismiss()方法上面；
```swift
if let selectedTheme = Theme(rawValue: themeSelector.selectedSegmentIndex) {
  ThemeManager.applyTheme(selectedTheme)
}
```
在这你就可以调用你在 ThemeManager中加入的方法了，这个方法会通过设置UIWindow的tintColor属性来设定主题的主色调。

下一步，添加下面这些代码到viewDidLoad()方法的底部，这样做了之后，themeSelector就会在view controller第一次加载时，选中之前的被存储到 NSUserDefaults中的主题
```swift
themeSelector.selectedSegmentIndex = ThemeManager.currentTheme().rawValue
```

Build and run.点击设置按键，选择Dark主题然后提交修改。App的主色调将瞬间会从绿色变为橙色。
![theme_applied2](http://p44bkxib3.bkt.clouddn.com/theme_applied2.png)

眼神锐利的读者或许已经注意到这些颜色就是定义在ThemeType的mainColor()


稍等，当你选择Dark，APP不像夜间的颜色，为了达到夜间的效果，你还需要再自定义些东西。

## 自定义导航条
打开Theme.swift， 添加两个方法到Theme:
```swift
var barStyle: UIBarStyle {
  switch self {
  case .Default, .Graphical:
    return .Default
  case .Dark:
    return .Black
  }
}
 
var navigationBackgroundImage: UIImage? {
  return self == .Graphical ? UIImage(named: "navBackground") : nil
}
```
这两个方法分别是返回navigation bar的样式和背景图片给每个主题。


下一步，添加下面几行代码到applyTheme():最后
```swift
UINavigationBar.appearance().barStyle = theme.barStyle
UINavigationBar.appearance().setBackgroundImage(theme.navigationBackgroundImage, forBarMetrics: .Default)
```

好的 — 为什么是在这里设置UINavigationBar的样式，而不是在你创建UINavigationBar对象的时候设置？
UIKit有一个非正式的协议叫做UIAppearance，UIAppearance包含了很多控件信息。当你在UIKit的类中调用appearance()，它将返回一个UIAppearance的代理，当你通过这个代理设置控件的属性，所有同类型的控件属性都会被设置为一样的值。你就可以很方便的在一个地方设置控件的属性，不需要在每个控件初始化的时候都设置一遍。

Build and run。选择夜间主题，导航条会看起来更像夜间:

![theme_applied3](http://p44bkxib3.bkt.clouddn.com/theme_applied3.png)

这样看起来是不是更加好看了，但是你还要做些工作。
 
下一步，你将自定义返回的按钮，iOS使用的是默认chevron符号，但是你可以写些更有趣的东西:]

## 自定义导航条的返回按钮
返回按钮是所有主题共用的，所以你只需在Themes.swift的applyTheme()里添加下列代码：
```swift
UINavigationBar.appearance().backIndicatorImage = UIImage(named: "backArrow")
UINavigationBar.appearance().backIndicatorTransitionMaskImage = UIImage(named: "backArrowMask")
```
这就简单的给UINavigationBar的返回按钮设置了返回图片和transition mask图片

Build and run. 点击宠物就能看到设置好的返回按钮:

![back_button](http://p44bkxib3.bkt.clouddn.com/back_button.png)

打开存放图片的Images.xcassets，找名字叫‘backArrow’的图片，你会发现图片是黑色的，但是在APP里图片颜色随着主题颜色的改变而改变。

iOS如何改变按钮图片的颜色，还有为什么只有返回按钮的图片颜色改变，不是所有的图片都改变呢？<br/>
事实上，iOS的图片有三个渲染模式：
* Original：总是使用和原来相同的颜色。
* Template：忽略图片颜色，只使用图片作为模板。在这种模式下，iOS只使用图片形状，图片的颜色是根据屏幕上的tint color进行渲染。所以当一个控件有一个tint color，iOS将使用你提供的图片的形状，颜色渲染成和tint color一样的颜色。
* Automatic：(图片默认的模式)系统根据你在哪里使用图片，决定使用‘Original’还是‘Template’模式渲染图片。比如back indicators, navigation control bar button items 和 tab bar 的图片, iOS 将忽略默认图片的颜色，除非你自己改变渲染模式。

回过头来看APP，点击某个宠物或者Adopt进入下一层。仔细观察navigation bar返回按钮的动画，你看到问题了吗？

![mask1](http://p44bkxib3.bkt.clouddn.com/mask1.gif)

当Back文字从左往右移动时，它转过了左箭头图片，看起来很不优雅：<br/>
修复它，你要改变transition mask 的图片<br/>
更新Themes.swift里applyTheme()的代码：
```swift
UINavigationBar.appearance().backIndicatorTransitionMaskImage = UIImage(named: "backArrow")
```
Build and run. 再次点击某个宠物或者Adopt进入下一层. 这次动画看起来好多了:<br/>
![mask2](http://p44bkxib3.bkt.clouddn.com/mask2.gif)

文字不再切断并且在左箭头图片的后面通过。所以到底发生了什么？<br/>
使用整张非透明图片和有透明的图片作为返回按钮的transition mask会产生完全不一样效果：使用整张非透明图片，文字会从左到右在左箭头图片后面通过，左箭头后面也是可显示的范围。
在最初的实现，你提供了一种图像覆盖整个表面，文字可见通过左箭头。但你现在使用的左箭头图片本身作为mask，但文字消失在mask的右边缘，而不是在左箭头的右边。

看看‘backarrowmaskfixed’图片，返回按钮箭头和文字如何完美的结合在一起:<br/>
![indicator_mask](http://p44bkxib3.bkt.clouddn.com/indicator_mask.png)

黑色的形状是左箭头图片，红色的形状是mask。这样文本可以从红色区域显示，在其他地方就隐藏。

再次改变applyTheme()方法的最后一行，这次使用升级版的mask：
```swift
UINavigationBar.appearance().backIndicatorTransitionMaskImage = UIImage(named: "backArrowMaskFixed")
```
Build and run.  点击某个宠物或者Adopt进入下一层. 你将会看到文字在图片下面消失，这就是你预期要实现的效果:<br/>
![mask3](http://p44bkxib3.bkt.clouddn.com/mask3.gif)

## 自定义Tab Bar
仍然在Theme.swift, 添加代码到 Theme:
```swift
var tabBarBackgroundImage: UIImage? {
  return self == .Graphical ? UIImage(named: "tabBarBackground") : nil
}
 
var backgroundColor: UIColor {
  switch self {
  case .Default, .Graphical:
    return UIColor(white: 0.9, alpha: 1.0)
  case .Dark:
    return UIColor(white: 0.8, alpha: 1.0)
  }
}
 
var secondaryColor: UIColor {
  switch self {
  case .Default:
    return UIColor(red: 242.0/255.0, green: 101.0/255.0, blue: 34.0/255.0, alpha: 1.0)
  case .Dark:
    return UIColor(red: 34.0/255.0, green: 128.0/255.0, blue: 66.0/255.0, alpha: 1.0)
  case .Graphical:
    return UIColor(red: 140.0/255.0, green: 50.0/255.0, blue: 48.0/255.0, alpha: 1.0)
  }
}
```

这是设置tab bar在不同主题下面的背景图片和背景颜色。

使用这些样式，需要添加代码到applyTheme()
```swift
UITabBar.appearance().barStyle = theme.barStyle
UITabBar.appearance().backgroundImage = theme.tabBarBackgroundImage
 
let tabIndicator = UIImage(named: "tabBarSelectionIndicator")?.imageWithRenderingMode(.AlwaysTemplate)
let tabResizableIndicator = tabIndicator?.resizableImageWithCapInsets(
    UIEdgeInsets(top: 0, left: 2.0, bottom: 0, right: 2.0))
UITabBar.appearance().selectionIndicatorImage = tabResizableIndicator
```

设置tab bar的样式和背景图片和之前设置UINavigationBar基本相同<br/>
最后三行代码是给tab bar设置indicator图片，indicator图片的渲染模式设置为.AlwaysTemplate，系统就会自动忽略图片的颜色，并且图片是张拉伸后的图片<br/>

Build and run. 你会看到最新样式的tab bar:

![theme_applied4](http://p44bkxib3.bkt.clouddn.com/theme_applied4.png)

夜间模式真正变得越来越好看了! :]<br/>
indicator 图片总是 6 points 高度 and 49 points 宽度, iOS 在运行时将它拉伸.<br/>
下一节将介绍可调整大小的图像和它们是如何工作的。

## 自定义Segmented Control
添加下列代码到Theme.swift的applyTheme()底部
```swift
let controlBackground = UIImage(named: "controlBackground")?
   .imageWithRenderingMode(.AlwaysTemplate)
     .resizableImageWithCapInsets(UIEdgeInsets(top: 3, left: 3, bottom: 3, right: 3))
let controlSelectedBackground = UIImage(named: "controlSelectedBackground")?
   .imageWithRenderingMode(.AlwaysTemplate)
     .resizableImageWithCapInsets(UIEdgeInsets(top: 3, left: 3, bottom: 3, right: 3))
 
UISegmentedControl.appearance().setBackgroundImage(controlBackground, forState: .Normal,
    barMetrics: .Default)
UISegmentedControl.appearance().setBackgroundImage(controlSelectedBackground, forState: .Selected,
    barMetrics: .Default)
```
为了理解代码，先看看controlBackground图片。图片非常小，但iOS会非常智能的裁剪和拉伸图片。

切片的意思是什么？请看放大后的模型：<br/>
![slicing](http://p44bkxib3.bkt.clouddn.com/slicing.png)


There are four 3×3 squares, one in each corner. These squares are left untouched when resizing the image, but the gray pixels get stretched horizontally and vertically as required.
In your image, all the pixels are black and assume the tint color of the control. You instruct iOS how to stretch the image using UIEdgeInsets() and passed 3 for the top, left, bottom and right parameters since your corners are 3×3.

Build and run. 点击Gear icon ，看到UISegmentedControl 新的样式:<br/>
![segmented](http://p44bkxib3.bkt.clouddn.com/segmented.png)


圆润的边角已经被你的3×3方角取代。

## 自定义 Steppers, Sliders, 和 Switches
改变stepper的颜色，添加下面的代码到Theme.swift的applyTheme()
```swift
UIStepper.appearance().setBackgroundImage(controlBackground, forState: .Normal)
UIStepper.appearance().setBackgroundImage(controlBackground, forState: .Disabled)
UIStepper.appearance().setBackgroundImage(controlBackground, forState: .Highlighted)
UIStepper.appearance().setDecrementImage(UIImage(named: "fewerPaws"), forState: .Normal)
UIStepper.appearance().setIncrementImage(UIImage(named: "morePaws"), forState: .Normal)
```
你已经给UISegmentedControl添加了图片。
这不仅改变了颜色的UISegmentedControl，还改变了无聊的 + 和 - 符号。

Build and run. 打开 Search 看看改变:<br/>

![stepper](http://p44bkxib3.bkt.clouddn.com/stepper.png)

UISlider 和 UISwitch 添加相应的主题。<br/>
添加下列代码到applyTheme():
```swift
UISlider.appearance().setThumbImage(UIImage(named: "sliderThumb"), forState: .Normal)
UISlider.appearance().setMaximumTrackImage(UIImage(named: "maximumTrack")?
    .resizableImageWithCapInsets(UIEdgeInsets(top: 0, left: 0.0, bottom: 0, right: 6.0)), 
      forState: .Normal)
UISlider.appearance().setMinimumTrackImage(UIImage(named: "minimumTrack")?
    .imageWithRenderingMode(.AlwaysTemplate)
      .resizableImageWithCapInsets(UIEdgeInsets(top: 0, left: 6.0, bottom: 0, right: 0)), 
        forState: .Normal)
 
UISwitch.appearance().onTintColor = theme.mainColor.colorWithAlphaComponent(0.3)
UISwitch.appearance().thumbTintColor = theme.mainColor
```

UISlider 有三个主要自定义的点: the slider’s thumb, the minimum track and the maximum track.

the maximum track没有更改渲染模式，the minimum track修改渲染模式，然后颜色和主题一致

UISwitch的thumbTintColor和主题颜色一致，onTintColor的颜色更浅一些，能够和thumbTintColor的颜色产生对比

Build and run. 点击 Search ：<br/>

![slider-switch](http://p44bkxib3.bkt.clouddn.com/slider-switch.png)

正如你看到的，appearance代理将会改变所有同类型控件的属性，但是有时候你不想全局修改控件，有些地方你想单独修改某个控件！

## 自定义单个控件
打开 SearchTableViewController.swift， 添加 下列代码到 viewDidLoad():
```swift
speciesSelector.setImage(UIImage(named: "dog"), forSegmentAtIndex: 0)
speciesSelector.setImage(UIImage(named: "cat"), forSegmentAtIndex: 1)
```
分别给segment设置图片。
Build and run. 打开 Search :<br/>
![species](http://p44bkxib3.bkt.clouddn.com/species.png)

iOS自动翻转segment的选中色，这是因为图片自动使用Template模式渲染图片。

打开 PetTableViewController.swift 添加代码到 viewWillAppear():
```swift
view.backgroundColor = ThemeManager.currentTheme().backgroundColor    
tableView.separatorColor = ThemeManager.currentTheme().secondaryColor
```
下一步，添加下列代码到tableView(_:cellForRowAtIndexPath:) 方法，在return之前
```swift
cell.textLabel!.font = UIFont(name: "Zapfino", size: 14.0)
```
改变cell textLabel 的字体

Build and run. 比较APP修改之前，之后的样子：<br/>

![theme_applied5-580x500](http://p44bkxib3.bkt.clouddn.com/theme_applied5-580x500.png)

下图显示之前和之后的结果搜索界面，我想你会同意，新版本更和谐和有趣。<br/>

![theme_applied6-580x500](http://p44bkxib3.bkt.clouddn.com/theme_applied6-580x500.png)

## 总结
你可以下载完成的项目从本教程的所有调整在[这里](http://cdn1.raywenderlich.com/wp-content/uploads/2015/06/Pet-Finder_Finished.zip)

