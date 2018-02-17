
## 前言
前两篇介绍CALayer和它的部分子类，本章继续学习其他的子类。

## CAShapeLayer
CAShapeLayer 可以通过设置矢量路径来绘制图片，它使用起来比使用图片快得多。另一个优势是你不再需要提供常规的@2x和@3x尺寸的图片了。

此外，你可以使用各种属性来自定义线条粗细，颜色，划线，连线路径。如果线条相交形成一个封闭的区域，可以设置填充该区域的颜色。

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    
    // 1
    self.color = [UIColor colorWithRed:248/255.0 green:96/255.0 blue:47/255.0 alpha:1.0];
    [self setUpOpenPath];
    [self setUpClosedPath];
    [self setUpShapeLayer];
}

- (void)setUpShapeLayer {
    // 3
    self.shapeLayer = ({
        CAShapeLayer *layer = [CAShapeLayer layer];
        layer.path = self.openPath.CGPath;
        layer.fillColor = nil;
        layer.fillRule = kCAFillRuleNonZero;
        layer.lineCap = kCALineCapButt;
        layer.lineDashPattern = nil;
        layer.lineDashPhase = 0.0;
        layer.lineJoin = kCALineJoinMiter;
        layer.lineWidth = _lineWidthSlider.value;
        layer.miterLimit = 4.0;
        layer.strokeColor = self.color.CGColor;
        [_viewForShapeLayer.layer addSublayer:layer];
        layer;
    });
    
}

- (void)setUpOpenPath {
	// 2
    self.openPath = [UIBezierPath bezierPath];
    [self.openPath moveToPoint:CGPointMake(30, 196)];
    [self.openPath addCurveToPoint:CGPointMake(112.0, 12.5) controlPoint1:CGPointMake(110.56, 13.79) controlPoint2:CGPointMake(112.07, 13.01)];
    [self.openPath addCurveToPoint:CGPointMake(194, 196) controlPoint1:CGPointMake(111.9, 11.81) controlPoint2:CGPointMake(194, 196)];
    [self.openPath addLineToPoint:CGPointMake(30.0, 85.68)];
    [self.openPath addLineToPoint:CGPointMake(194.0, 48.91)];
    [self.openPath addLineToPoint:CGPointMake(30.0, 196)];
}

- (void)setUpClosedPath {
    self.closedPath = [UIBezierPath bezierPath];
    self.closedPath.CGPath = self.openPath.CGPath;
    [self.closedPath closePath];
}
```

分析：
1. 创建 color，path 和 shapeLayer
2. 绘制 shapeLayer 路径。如果你不想这个写绘图代码，你可以使用 PaintCode 软件，导入矢量图（SVG）或者 photoshop（PSD）文件来生成你的代码。
3. 设置 shapeLayer 的属性

**CAShapeLayer** 属性
* **fillRule**：填充规则，有两个选择 **kCAFillRuleNonZero** 和 **kCAFillRuleEvenOdd**

**nonzero**字面意思是“非零”。按该规则，要判断一个点是否在图形内，从该点作任意方向的一条射线，然后检测射线与图形路径的交点情况。从0开始计数，路径从左向右穿过射线则计数加1，从右向左穿过射线则计数减1。得出计数结果后，如果结果是0，则认为点在图形外部，否则认为在内部。下图演示了nonzero规则:

![nonzero](http://p44bkxib3.bkt.clouddn.com/nonzero.png)

**evenodd**字面意思是“奇偶”。按该规则，要判断一个点是否在图形内，从该点作任意方向的一条射线，然后检测射线与图形路径的交点的数量。如果结果是奇数则认为点在内部，是偶数则认为点在外部。下图演示了evenodd 规则
![evenodd](http://p44bkxib3.bkt.clouddn.com/evenodd.png)

* **lineCap**：线端点类型

![lineCap](http://p44bkxib3.bkt.clouddn.com/lineCap.png)

* **lineJoin**：线连接类型

![lineJoin](http://p44bkxib3.bkt.clouddn.com/lineJoin.png)

* **lineDashPattern**：线型模板
这是一个NSNumber的数组，索引从1开始记，奇数位数值表示实线长度，偶数位数值表示空白长度

* **lineDashPhase**：线型模板的起始位置
默认是0，可以理解为第一个空格起始位置 = 实线长度 - lineDashPhase。假设设置为5绘制 5空格，设置lineDashPhase为2，那么第一段绘制为5-2，然后空5绘制5


## CATransformLayer
CATransformLayer 和别的平面的Layer不一样，它可以绘制3D结构。它实际上是其sublayers的容器，每个子图层可以有自己的transforms和opacity变换，但是，它不会渲染自己的属性，比如背景色，边框，只渲染sublayers。

你不能直接点击一个transform layer，因为它没有一个2D坐标空间来映射一个接触点，但是能够点击单个sublayers。

```objc
@interface ViewController ()  
{  
    CGPoint startPoint;  
  
    CATransformLayer *s_Cube;  
  
    float pix, piy;  
}  
  
@property (nonatomic, weak) IBOutlet UIView *containerView;  
  
@end  
  
@implementation ViewController  
  
- (CALayer *)faceWithTransform:(CATransform3D)transform  
{  
    //create cube face layer  
    CALayer *face = [CALayer layer];  
    face.frame = CGRectMake(-50, -50, 100, 100);  
      
    //apply a random color  
    CGFloat red = (rand() / (double)INT_MAX);  
    CGFloat green = (rand() / (double)INT_MAX);  
    CGFloat blue = (rand() / (double)INT_MAX);  
    face.backgroundColor = [UIColor colorWithRed:red  
                                           green:green  
                                            blue:blue  
                                           alpha:1.0].CGColor;  
  
    //apply the transform and return  
    face.transform = transform;  
    return face;  
}  
  
- (CALayer *)cubeWithTransform:(CATransform3D)transform  
{  
    //create cube layer  
    CATransformLayer *cube = [CATransformLayer layer];  
      
    //add cube face 1  
    CATransform3D ct = CATransform3DMakeTranslation(0, 0, 50);  
    [cube addSublayer:[self faceWithTransform:ct]];  
      
    //add cube face 2  
    ct = CATransform3DMakeTranslation(50, 0, 0);  
    ct = CATransform3DRotate(ct, M_PI_2, 0, 1, 0);  
    [cube addSublayer:[self faceWithTransform:ct]];  
      
    //add cube face 3  
    ct = CATransform3DMakeTranslation(0, -50, 0);  
    ct = CATransform3DRotate(ct, M_PI_2, 1, 0, 0);  
    [cube addSublayer:[self faceWithTransform:ct]];  
      
    //add cube face 4  
    ct = CATransform3DMakeTranslation(0, 50, 0);  
    ct = CATransform3DRotate(ct, -M_PI_2, 1, 0, 0);  
    [cube addSublayer:[self faceWithTransform:ct]];  
      
    //add cube face 5  
    ct = CATransform3DMakeTranslation(-50, 0, 0);  
    ct = CATransform3DRotate(ct, -M_PI_2, 0, 1, 0);  
    [cube addSublayer:[self faceWithTransform:ct]];  
      
    //add cube face 6  
    ct = CATransform3DMakeTranslation(0, 0, -50);  
    ct = CATransform3DRotate(ct, M_PI, 0, 1, 0);  
    [cube addSublayer:[self faceWithTransform:ct]];  
      
    //center the cube layer within the container  
    CGSize containerSize = self.containerView.bounds.size;  
    cube.position = CGPointMake(containerSize.width / 2.0,  
                                containerSize.height / 2.0);  
      
    //apply the transform and return  
    cube.transform = transform;  
    return cube;  
}  
  
- (void)viewDidLoad  
{  
    [super viewDidLoad];  
      
    //set up the perspective transform  
    CATransform3D pt = CATransform3DIdentity;  
    pt.m34 = -1.0 / 500.0;  
    self.containerView.layer.sublayerTransform = pt;  
      
    //set up the transform for cube 1 and add it  
    CATransform3D c1t = CATransform3DIdentity;  
    c1t = CATransform3DTranslate(c1t, -100, 0, 0);  
    CALayer *cube1 = [self cubeWithTransform:c1t];  
    s_Cube = (CATransformLayer *)cube1;  
    [self.containerView.layer addSublayer:cube1];  
      
    //set up the transform for cube 2 and add it  
    CATransform3D c2t = CATransform3DIdentity;  
    c2t = CATransform3DTranslate(c2t, 100, 0, 0);  
    c2t = CATransform3DRotate(c2t, -M_PI_4, 1, 0, 0);  
    c2t = CATransform3DRotate(c2t, -M_PI_4, 0, 1, 0);  
    CALayer *cube2 = [self cubeWithTransform:c2t];  
    [self.containerView.layer addSublayer:cube2];  
}  
  
- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event  
{  
    UITouch *touch = [touches anyObject];  
  
    startPoint = [touch locationInView:self.view];  
}  
  
- (void)touchesMoved:(NSSet *)touches withEvent:(UIEvent *)event  
{  
    UITouch *touch = [touches anyObject];  
  
    CGPoint currentPosition = [touch locationInView:self.view];  
  
    CGFloat deltaX = startPoint.x - currentPosition.x;  
  
    CGFloat deltaY = startPoint.y - currentPosition.y;  
  
    CATransform3D c1t = CATransform3DIdentity;  
    c1t = CATransform3DTranslate(c1t, -100, 0, 0);  
    c1t = CATransform3DRotate(c1t, pix+M_PI_2*deltaY/100, 1, 0, 0);  
    c1t = CATransform3DRotate(c1t, piy-M_PI_2*deltaX/100, 0, 1, 0);  
  
    s_Cube.transform = c1t;  
}  
  
- (void)touchesEnded:(NSSet *)touches withEvent:(UIEvent *)event  
{  
    UITouch *touch = [touches anyObject];  
  
    CGPoint currentPosition = [touch locationInView:self.view];  
  
    CGFloat deltaX = startPoint.x - currentPosition.x;  
  
    CGFloat deltaY = startPoint.y - currentPosition.y;  
  
    pix = M_PI_2*deltaY/100;  
    piy = -M_PI_2*deltaX/100;  
}  
  
@end  
```

分析：
1. 给立方体创建每个面，设置不一样的颜色，并添加到同一个transform layer
2. 给每个面设置不一样的Z轴的值已经旋转角度，增加层次感
3. 每次旋转都是基于x轴和y轴的偏移。注意，代码里是给 **sublayerTransform** 设置 **transform** 值，它可以应用到transform layer的每个sublayers


## CAEmitterLayer
CAEmitterLayer 渲染粒子动画，每个粒子都是一个 CAEmitterCell 对象。通过设置 CAEmitterLayer 和 CAEmitterCell 的属性，可以改变粒子的速率，大小，形状，颜色，加速度，生命周期等等。

```objc
@interface JHCAEmitterLayerViewController ()
@property (weak, nonatomic) IBOutlet UIView *viewForEmitterLayer;
@end

@implementation JHCAEmitterLayerViewController
- (CAEmitterLayer *)emitterLayer {
    if (!_emitterLayer) {
        _emitterLayer = ({
            CAEmitterLayer *layer = [CAEmitterLayer layer];
            layer.frame=  CGRectMake(0, 0, kScreenWidth, kScreenHeight * 0.5);
            layer.seed = [[NSDate date] timeIntervalSince1970];
            layer.emitterPosition = CGPointMake(kScreenWidth * 0.5, kScreenHeight * 0.25);
            [_viewForEmitterLayer.layer addSublayer:layer];
            layer;
        });
    }
    return _emitterLayer;
}

- (CAEmitterCell *)emitterCell {
    if (!_emitterCell) {
        _emitterCell = ({
            CAEmitterCell *emitterCell = [CAEmitterCell emitterCell];
            emitterCell.enabled = YES;
            emitterCell.name = @"yjh";
            emitterCell.contents = (__bridge id _Nullable)([UIImage imageNamed:@"smallStar"].CGImage);
            emitterCell.contentsRect = CGRectMake(0, 0, 1, 1);
            emitterCell.color = [UIColor colorWithRed:0 green:0 blue:0 alpha:1.0].CGColor;
            emitterCell.redRange = 1.0;
            emitterCell.greenRange = 1.0;
            emitterCell.blueRange = 1.0;
            emitterCell.alphaRange = 0.0;
            emitterCell.redSpeed = 0.0;
            emitterCell.greenSpeed = 0.0;
            emitterCell.blueSpeed = 0.0;
            emitterCell.alphaSpeed = -0.5;
            emitterCell.scale = 1.0;
            emitterCell.scaleRange = 0.0;
            emitterCell.scaleSpeed = 0.1;
            
            CGFloat zeroDegreesInRadians = [self degreesToRadians:0.0];
            emitterCell.spin = [self degreesToRadians:130.0];
            emitterCell.spinRange = zeroDegreesInRadians;
            emitterCell.emissionLatitude = zeroDegreesInRadians;
            emitterCell.emissionLongitude = zeroDegreesInRadians;
            emitterCell.emissionRange = [self degreesToRadians:360.0];
            
            emitterCell.lifetime = 1.0;
            emitterCell.lifetimeRange = 0.0;
            emitterCell.birthRate = 250.0;
            emitterCell.velocity = 50.0;
            emitterCell.velocityRange = 500.0;
            emitterCell.xAcceleration = -750.0;
            emitterCell.yAcceleration = 0.0;
            
            emitterCell;
        });
    }
    return _emitterCell;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    [self resetEmitterCells];
}
```

分析：
1. 创建 emitter layer 和 emitter cell
2. 设置 emitter cells 的渲染模式 renderMode
3. 将异步渲染 drawsAsynchronously 设置为YES，这可以提高性能，因为发射器层必须持续地重绘其发射器单元。
4. 设置 emitterCell 时，有很多属性需要赋值。contents 设置粒子的内容，然后设置初始速度和最大方差（velocityRange），emitter layer 使用 初始值+/-方差 区间内的随机数来初始化粒子，很多 emitterCell 的属性都有方差设置。
5. 设置 cell 的生命周期是1秒，默认是0，所以如果你没明确设置，cell 永远不会出现，birthRate（每秒产生数量） 也一样，默认是0，必须设置为正整数才能显示 cell
6. 设置 x 和 y轴的加速度，它会影响粒子的运动轨迹


## 参考资料
* [IOS Core Animation Advanced Techniques的学习笔记(五)](http://blog.csdn.net/iunion/article/details/26221213)
* [关于贝塞尔曲线](http://blog.csdn.net/shidongdong2012/article/details/22478463)

