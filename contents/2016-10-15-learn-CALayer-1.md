
## 前言
我们都知道在iOS中每一个你看到的东西都是由View组成的，比如按钮视图，table视图等等。但是你可能不知道每个View背后都有一个叫做CALayer的类。本系列文章着重学习使用CALayer，CALayer和View的区别请参考：
* [详解CALayer 和 UIView的区别和联系](http://www.jianshu.com/p/079e5cf0f014)
* [View-Layer 协作](https://objccn.io/issue-12-4/)

本章的内容大部分是从[CALayer Tutorial: Getting Started](http://www.raywenderlich.com/90488/calayer-in-ios-with-swift-10-examples)学习得来。

## CALayer
CALayer是接下来要学习到的Layers的基类，包含了很多基础属性，比如
* **contents**：表示Layer呈现的内容，一般会设置为CGImage的对象
* **contentsGravity**：表示内容对齐和填充方式
* **cornerRadius**：表示Layer圆角半径，默认zero
* **masksToBounds**：是否进行bounds的切割，设置圆角属性是会设置为YES

Layer还有很多属性和View相似，比如**frame**，**bounds**等等用于设置Layer的位置。

CALayer还有几个知识点：
* Layers有subLayer，就像Views有subViews
* Layer的properties能够产生隐式动画。当你改变Layer的property，它会默认产生隐式动画。你也可以使用自定义动画。
* Layer比Views更轻量级，更能节约性能。
* Layers有大量有用的properties。

看下面这段代码：
```objc
// 1
_layer = [CALayer new];
_layer.frame = CGRectMake(0, 0, 200, 200);

// 2
_layer.contents = (__bridge id _Nullable)([[UIImage imageNamed:@"star"] CGImage]);
_layer.contentsGravity = kCAGravityCenter;

// 3
_layer.magnificationFilter = kCAFilterLinear;
_layer.geometryFlipped = NO;

// 4
_layer.backgroundColor = [kDefaultBackgroundColor CGColor];

// 5
_layer.cornerRadius = 100.0;
_layer.borderWidth = 12.0;
_layer.borderColor = [[UIColor whiteColor] CGColor];

// 6
_layer.shadowOpacity = 0.75;
_layer.shadowOffset = CGSizeMake(0, 3);
_layer.shadowRadius = 3.0;
```

分析：
1. 创建了一个CALayer对象，设置它的frame
2. 把layer的内容设置为一张图片，图片的格式是Quartz image data（CGImage）
3. 设置内容对齐和填充方式
4. 设置layer的背景色
5. 设置layer的圆角弧度为自己宽度的一半（它就会变成圆形），设置它的边框宽度和颜色
6. 设置layer的阴影透明度，阴影偏移量，阴影圆角半径

最终效果：
![Snip20161029_1](http://p44bkxib3.bkt.clouddn.com/Snip20161029_1.png)

**CALayer有附加属性能够提高性能**：
* **shouldRasterize**：光栅化，设置为YES后CALayer会被光栅化为bitmap，layer的阴影等效果也会被保存到bitmap中，`缓存起来后不需要多次渲染提高了性能`，而对于经常变动的内容，这个时候不要开启，否则会造成性能的浪费。
* **drawsAsynchronously**：异步渲染，这个和shouldRasterize正好相反，默认也是NO，设置为YES能够提高`需要反复渲染`layer内容的性能，比如之后会学习到的**emitter layer**连续渲染颗粒动画。

> 提醒：在设置**shouldrasterize**或**drawsasynchronously**时，应该反复比较设置前后的性能，正确使用时，能够提高性能，误用时，性能可能会急转直下。

> 注意：Layer不属于响应链条的一部分，它无法和View一样直接响应点击和手势。


## CAScrollLayer
CAScrollLayer是CALayer的子类，用于显示Layer的一部分，不能直接响应用户的触摸甚至检查滚动范围。CAScrollLayer的可滚动区域的范围是由它的子层布局来确定的。 CAScrollLayer不提供键盘或鼠标事件处理，也没有提供可见滚动条。 

UIScrollView并没有用CAScrollLayer，事实上，就是简单的通过直接操作图层边界来实现滑动。

CAScrollLayer可以设置它的滚动方向，还可以滚动到特定的点或者区域。

```objc
#import "ScrollView.h"
@implementation ScrollView
// 1
+ (Class)layerClass
{
    return [CAScrollLayer class];
}

- (void)setUp
{
    //enable clipping
    self.layer.masksToBounds = YES;

    //attach pan gesture recognizer
    UIPanGestureRecognizer *recognizer = nil;
    recognizer = [[UIPanGestureRecognizer alloc] initWithTarget:self action:@selector(pan:)];
    [self addGestureRecognizer:recognizer];
}

- (id)initWithFrame:(CGRect)frame
{
    //this is called when view is created in code
    if ((self = [super initWithFrame:frame])) {
        [self setUp];
    }
    return self;
}

- (void)awakeFromNib {
    //this is called when view is created from a nib
    [self setUp];
}

- (void)pan:(UIPanGestureRecognizer *)recognizer
{
    //get the offset by subtracting the pan gesture
    //translation from the current bounds origin
    CGPoint offset = self.bounds.origin;
    offset.x -= [recognizer translationInView:self].x;
    offset.y -= [recognizer translationInView:self].y;

    //scroll the layer
    [(CAScrollLayer *)self.layer scrollToPoint:offset];

    //reset the pan gesture translation
    [recognizer setTranslation:CGPointZero inView:self];
    
    // 结束恢复原点
//    if (sender.state == UIGestureRecognizerStateEnded) {
//        [UIView animateWithDuration:0.3 animations:^{
//            [self.scrollingViewLayer scrollToPoint:CGPointZero];
//        }];
//    }

}
@end
```
分析：
1. 创建一个View的子类，重写**layerClass**方法，返回**CAScrollLayer**。这里修改了创建的Layer类，并添加它作为sublayer。
2. 在手势响应方法中，它会滚到触摸点。注意：**scrollToPoint** 和 **scrollToRect**方法没有隐式动画，需要使用 UIView 动画才会有滚动动画。

这里有一些经验规则时使用（或不使用）CAScrollLayer：
* 如果你想使用轻量级的，只需要通过代码的方式滚动：考虑使用CAScrollLayer
* 如果你想用户能够通过屏幕滚动：最好使用UIScrollView，了解更多可以点击[链接](https://videos.raywenderlich.com/courses/scroll-view-school/lessons/1)
* 如果你要滚动非常巨大的图片：考虑使用CATiledLayer（接下来会学到）

## CATextLayer
CATextLayer能够简单而快速绘制普通文本或者富文本，和UILabel不一样，CATextLayer没有UIFont属性，只有 CTFontRef 或者 CGFontRef。


```objc
- (void)setUpTextLayer {
	// 1
	CFStringRef fontName = (__bridge CFStringRef)@"Noteworthy-Light";
	self.noteworthyLightFont = CTFontCreateWithName(fontName, baseFontSize, nil);
	fontName = (__bridge CFStringRef)@"Helvetica";
	self.helveticaFont = CTFontCreateWithName(fontName, baseFontSize, nil);
	
	// 2
	NSMutableString *string =[NSMutableString string];
	for (NSInteger i = 0; i <= 20; i++) {
		[string appendFormat:@"Lorem ipsum dolor sit amet, consectetur adipiscing elit. Fusce auctor arcu quis velit congue dictum. "];
	}
	// 3
	self.textLayer = ({
	   CATextLayer *layer = [CATextLayer layer];
	   layer.frame = self.viewForTextLayer.bounds;
	   layer.string = string;
	   layer.font = self.helveticaFont;
	   layer.foregroundColor = [[UIColor darkGrayColor] CGColor];
	   layer.wrapped = YES;
	   layer.alignmentMode = kCAAlignmentLeft;
	   layer.truncationMode = kCATruncationEnd;
	   layer.contentsScale = [[UIScreen mainScreen] scale];
	   layer;
	});
}
```
分析：
1. 创建给Layer赋值的字体
2. 创建显示文本
3. 创建CATextLayer并设置属性，contentsScale匹配当前屏幕的发大倍数

不仅仅是CATextLayer，所有的Layer类，默认的放大系数**contentsScale**都是 1。当关联上Views时，layer会自动设置**contentsScale**与当前屏幕的放大系数一致。你手动创建的Layer时，需要设置**contentsScale**，否则它的放大系数和Retina不匹配。




## 参考链接
* [详解CALayer 和 UIView的区别和联系](http://www.jianshu.com/p/079e5cf0f014)
* [View-Layer 协作](https://objccn.io/issue-12-4/)
* [CALyer.h阅读笔记](http://www.360doc.com/content/16/0318/10/31697881_543270664.shtml)
* [iOS-Core-Animation-Advanced-Techniques(七)](http://www.cocoachina.com/ios/20150106/10840.html)
* [WWDC心得与延伸:iOS图形性能](http://www.cocoachina.com/ios/20150429/11712.html)
* [iOS-Core-Animation-Advanced-Techniques](https://github.com/AttackOnDobby/iOS-Core-Animation-Advanced-Techniques)




