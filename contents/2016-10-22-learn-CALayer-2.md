
## 前言
上一篇文章学习了**CALayer**和它的两个子类**CAScrollLayer**和**CATextLayer**。本章继续学习别的Layer。

## AVPlayerLayer
AVPlayerLayer是AVFoundation的一个Layer，它有个AVPlayer来播放 AV media 文件（AVPlayerItems）。

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self setUpPlayerLayer];
    
    // 3
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(playerDidReachEndNotificationHandler:) name:AVPlayerItemDidPlayToEndTimeNotification object:nil];
}

- (void)setUpPlayerLayer {
    // 1
    NSURL *url = [[NSBundle mainBundle] URLForResource:@"colorfulStreak" withExtension:@"m4v"];
    AVPlayer *player = [AVPlayer playerWithURL:url];
    player.actionAtItemEnd = AVPlayerActionAtItemEndNone;
    // 2
    self.playerLayer = [AVPlayerLayer layer];
    self.playerLayer.frame = CGRectMake(0, 0, 300, 180);
    self.playerLayer.player = player;
    [self.viewForPlayerLayer.layer addSublayer:self.playerLayer];
}

- (void)playerDidReachEndNotificationHandler:(NSNotification *)notification {
    AVPlayerItem *playerItem = self.playerLayer.player.currentItem;
    if (playerItem) {
        [playerItem seekToTime:kCMTimeZero];
    }
}
```

分析：
1. 根据播放文件路径创建 player ，告诉player播放完成后什么也不做
2. 创建 playerLayer
3. 注册 AVPlayer 的播放一个文件结束时的通知。（记得在controller销毁时移除监听）

`AVPlayerLayer` 有两个属性：
* **videoGravity**：设置视频的显示方式
* **readyForDisplay**：是否视频加载完成就立刻播放

`AVPlayer` 有几个特别的属性和方法：
* **rate**：表示0到1的播放速率，0表示暂停，1表示正常速度播放（1X）。换句话说，调用 **pause** 方法等同于 设置 Rate 为 0；调用 **play** 方法等同于 设置 rate 为 1。

那么设置快进，慢动作或者反向播放，可以直接 rate 的值来实现，设置超过 1 的值，就是要求视频快进播放，比如设置为 2，就是两倍的速度播放。设置为负数时，就是反向播放。

在设置 rate 的时候，应该先验证视频能否按照想要设置速率来播放，在 `AVPlayerItem` 中提供了检验的方法：
* **canPlayFastForward** rate 大于 1
* **canPlaySlowForward** rate 在 0 和 1 之间
* **canPlayReverse** rate 是 -1
* **canPlaySlowReverse** rate 在 -1 和 0 之间
* **canPlayFastReverse** rate 小于 -1

## CAGradientLayer
当你使用2种以上颜色时，CAGradientLayer 能够让它很容易应用到背景上，你需要配置一组从 **startPoint** 到 **endPoint** 对应的CGColors。

**startPoint** 和 **endPoint** 不是明确的点，它们是坐标系的边界反映。x的值是1，意味着该点位于layer的右边缘，y的值是1，意味着该点在layer的底部边缘。

CAGradientLayer 有个 **type** 属性，只有**kCAGradientLayerAxial** 一个选项，表示线性过渡颜色数组的值。

如下图所示，线A从 **startPoint** 到 **endPoint** 是渐变方向，垂直于线A的线B，沿着线B的颜色都是一样的。
![AxialGradientLayerType](http://p44bkxib3.bkt.clouddn.com/AxialGradientLayerType.gif)

另外，你还可以通过 **locations** 属性来控制颜色数组的显示，分配给每个颜色的显示范围。如果没有设置 **locations** 这个值，每个颜色都是平均显示，均匀分布。如果设置了，它的数量必须和颜色的数量对应，否则会发生错误。

```objc
- (void)setUpGradientLayer {
    self.gradientLayer = [CAGradientLayer layer];
    self.gradientLayer.frame = CGRectMake(0, 0, 200, 200);
    // colors 接收 CGColorRef 的对象数组，用于定义每个 gradient 的结束位置
    self.gradientLayer.colors = self.colors;
    // gradient 渐变的起始锚点
    self.gradientLayer.startPoint = CGPointMake(0.5f, 0.0f);
    // gradient 渐变的结束锚点
    self.gradientLayer.endPoint = CGPointMake(0.5f, 1.0f);
    // 渐变的结束位置，与colors一一对应
    self.gradientLayer.locations = self.locations;
    
    [self.viewForGradientLayer.layer addSublayer:self.gradientLayer];
}
```

显示效果如下图：
![CAGradientLayer-500x500](http://p44bkxib3.bkt.clouddn.com/CAGradientLayer-500x500.png)


## CAReplicatorLayer
CAReplicatorLayer 可以将自己的sublayers复制指定的次数。
每个复制的layer都可以拥有自己的颜色和属性，CAReplicatorLayer可以延迟sublayers的动画效果显示，还支持 Depth（景深）效果产生3D效果。 

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    
    // 创建CAReplicatorLayer，它是个容器层，你添加内容到它上面，它会复制添加的内容。
    _replicatorLayer = [CAReplicatorLayer layer];
    [self setUpReplicatorLayer];
    [_viewForReplicatorLayer.layer addSublayer:_replicatorLayer];
    
    // 需要被复制的layer
    _instanceLayer = [CALayer layer];
    [self setUpInstanceLayer];
    [_replicatorLayer addSublayer:_instanceLayer];
    
    [self setUpLayerFadeAnimation];
    [self instanceDelaySliderChanged:_instanceDelaySlider];
    [self updateLayerSizeSliderValueLabel];
    [self updateInstanceCountSliderValueLabel];
    [self updateInstanceDelaySliderValueLabel];
}

- (void)setUpReplicatorLayer {
    _replicatorLayer.frame = CGRectMake(0, 0, 250, 250);
    CGFloat count = self.instanceCountSlider.value;
    // 复制多少份
    _replicatorLayer.instanceCount = count;
    // 是否开启三维空间效果
    _replicatorLayer.preservesDepth = NO;
    _replicatorLayer.instanceColor = [[UIColor whiteColor] CGColor];
    // 颜色值递减
    _replicatorLayer.instanceRedOffset = [self offsetValueForSwitch:_offsetRedSwitch];
    _replicatorLayer.instanceGreenOffset = [self offsetValueForSwitch:_offsetGreenSwitch];
    _replicatorLayer.instanceBlueOffset = [self offsetValueForSwitch:_offsetBlueSwitch];
    _replicatorLayer.instanceAlphaOffset = [self offsetValueForSwitch:_offsetAlphaSwitch];
    CGFloat angle = (M_PI * 2.0) / count;
    // 复制图层在被创建时产生的和上一个复制图层的位移(位移的锚点时CAReplicatorlayer的中心点)
    _replicatorLayer.instanceTransform = CATransform3DMakeRotation(angle, 0.0, 0.0, 1.0);
}
```

分析：
1. 创建 CAReplicatorLayer 对象，设置它的frame
2. 设置 replicator layer 复制sublayers个数和延迟渲染时间，设置为2D效果，(preservesDepth = false) ，背景色为白色
3. 添加 RGBA渐变到每个成功复制的实例上，RGB默认都是0。在本例子中，默认颜色是白色，意味这个RGB都已经设置为1.0
4. 设置渐变路径是围成个圆

更多详情看Demo的代码

最终效果：
![CAReplicatorLayer](http://p44bkxib3.bkt.clouddn.com/CAReplicatorLayer.gif)

使用 CAReplicatorLayer 可以创建很多 cool 的动画，可以参考：
* [使用CAReplicatorLayer创建动画](http://www.ios-animations-by-emails.com/posts/2015-march#tutorial)
* [CALayer-CAReplicatorLayer(复制图层)](http://www.jianshu.com/p/84455b674f55)


## CATiledLayer
有些时候你可能需要绘制一个很大的图片，常见的例子就是一个高像素的照片或者是地球表面的详细地图。iOS应用通畅运行在内存受限的设备上，所以读取整个图片到内存中是不明智的。载入大图可能会相当地慢，那些对你看上去比较方便的做法（在主线程调用UIImage的-imageNamed:方法或者-imageWithContentsOfFile:方法）将会阻塞你的用户界面，至少会引起动画卡顿现象。

CATiledLayer为载入大图造成的性能问题提供了一个解决方案：将大图分解成小片然后将他们单独按需载入。

[详细实践](https://zsisme.gitbooks.io/ios-/content/chapter6/catiledLayer.html)

CATiledLayer 有两个properties：（[详细解释](http://www.cocoachina.com/bbs/read.php?tid-31201.html)）
* **levelsOfDetail** ：表示一共有多少个drawLayer的位置
* **levelsOfDetailBias**：表示比1大的位置里有多少个drawLayer的位置（包括1）

## 参考链接
* [iOS-Core-Animation-Advanced-Techniques(七)](http://www.cocoachina.com/ios/20150106/10840.html)
* [WWDC心得与延伸:iOS图形性能](http://www.cocoachina.com/ios/20150429/11712.html)
* [iOS-Core-Animation-Advanced-Techniques](https://github.com/AttackOnDobby/iOS-Core-Animation-Advanced-Techniques)



