本篇文章是参考[南峰子_老驴#iOS知识小集](http://huati.weibo.com/k/iOS%E7%9F%A5%E8%AF%86%E5%B0%8F%E9%9B%86?from=501)并结合自己实践所做。

## 问题
在iOS上有很多让人耳目一新的动画，非常吸引眼球，苹果也对许多控件都加上了默认的动画，比如UITableView和UIColletionView增加一个元素时，会有默认的动画，我们创建一个UIButton，然后改变它的文字也会有个渐变动画

```objc
self.button = ({
   UIButton *btn = [UIButton buttonWithType:UIButtonTypeSystem];
   btn.frame = (CGRect){100.0f, 50.0f, 200.0f, 30.0f};
   [btn setTitle:@"This is a button" forState:UIControlStateNormal];
   [btn setTitleColor:[UIColor redColor] forState:UIControlStateNormal];
   btn.backgroundColor = [UIColor blueColor];
   btn;
});
    
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
	[self.button setTitle:@"Button is here" forState:UIControlStateNormal];
	[self.button layoutIfNeeded];
});
```

## 解决
但是某些情况下我们不想要这些动画，在iOS7+的系统上，可以使用UIView的`performWithoutAnimation:`方法来达到这一目的。

这个方法的具体解释，可以参考objc.io的[文章](http://t.cn/Rthp9c3)，它的说法是这个方法就是一个使动画失效的简单封装。
![Snip20160906_8](http://p44bkxib3.bkt.clouddn.com/Snip20160906_8.png)

`performWithoutAnimation:`只能影响到block参数里的动画，对外面的动画没有影响，我们可以这么做取消掉动画

```objc
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
   [UIView performWithoutAnimation:^{
       [self.button setTitle:@"Button is here" forState:UIControlStateNormal];
       [self.button layoutIfNeeded];
   }];
});
```

可以把UIView动画放到`performWithoutAnimation:`中，这样动画也不会执行

```objc
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
   [UIView performWithoutAnimation:^{
       [UIView animateWithDuration:3 animations:^{
           self.button.frame = CGRectMake(100.0, 100.0, 100.0, 30.0);
       }];
   }];
});
```

### CALayer动画
假如你使用到了CALayer的动画，使用上述方法是解决不了的，`performWithoutAnimation:`只能解决UIView的Animation，原文章也给了解决方案

```objc
- (void)layoutSubviews {
	[super layoutSubviews];
	    
	[CATransaction begin];
	[CATransaction setDisableActions:YES];
	self.frameLayer.frame = self.frameView.bounds;
	[CATransaction commit];
}
```

再次感谢**南峰子_老驴**的Tips！

## 参考链接
[Tips:取消UICollectionView的隐式动画](http://adad184.com/2015/11/10/disable-uicollectionview-implicit-animation/)

## 补充
```objc
UIButton *btn = [[UIButton alloc] init];
```
如果你是通过这种方式创建UIButton，改变button文字是没有渐变动画的，因为通过这种方式创建没有走苹果类方法中的某些处理，苹果也就不会给你加上动画。



