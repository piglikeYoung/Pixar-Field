
## 前言

下拉刷新，上拉加载更多是现在 APP 中非常常见的功能，本文就来看看 **MJRefresh** 是的源码，分析它如何实现的

## 下拉刷新的基本原理

一般的下拉刷新都是用 **ScrollView** 的 `contentInset` 来实现的，我们常用在 TableView 的顶部， TableView 在导航栏下方，它的 **contentInset.top** 就是 `64`，**contentOffset.y** 就是 `-64`。

在下面的代码中打印可以看出：

```objc
// MJRefreshComponent.m
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context
{
    NSLog(@"contentInset ---> %f", _scrollView.contentInset.top);
    NSLog(@"contentOffset ---> %f", _scrollView.contentOffset.y);
    
    // ......
}
```

继续下拉 **contentInset.top** 不变，**contentOffset.y** 变小，上拉 **contentOffset.y** 变大，直到 ScrollView 达到屏幕的左上角变为0。

默认情况下，下拉 TableView ，松手后会弹回初始位置，而下拉刷新控件就是放在 TableView 的的上方，初始 y 设置为负数，所以平时不会显示出来，只有下拉的时候才会出现。在 Loading 的时候，临时把 **contentInset.top** 变大，就把 TableView 往下挤，这样刷新控件就停留在上面，刷新完成后，再把 **contentInset.top** 改回原来的值，实现回弹的效果。

## 关键代码分析

```objc
// 下拉刷新
tableView.mj_header= [MJRefreshNormalHeader headerWithRefreshingBlock:^{
    // 模拟延迟加载数据，因此2秒后才调用（真实开发中，可以移除这段gcd代码）
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        // 结束刷新
        [tableView.mj_header endRefreshing];
    });
}];
```

上面的代码是将刷新控件添加到 `mj_header` 属性中，实际上控件并不是添加到 **TableView** 上，而是是添加到 **ScrollView** 上：

```objc
// UIScrollView+MJRefresh.m
- (void)setMj_header:(MJRefreshHeader *)mj_header {
    if (mj_header != self.mj_header) {
        // 删除旧的，添加新的
        [self.mj_header removeFromSuperview];
        [self insertSubview:mj_header atIndex:0];
        
        // 存储新的
        objc_setAssociatedObject(self, &MJRefreshHeaderKey,
                                 mj_header, OBJC_ASSOCIATION_ASSIGN);
    }
}
```
上面代码是在 **ScrollView** 的 category 中，给 **UIScrollView** 添加了属性 header 和 footer。通常 category 是不能直接添加实例变量的，所以用到了 `objc_setAssociatedObject` 关联对象，所以刷新控件是添加到了 **UIScrollView** 的 subView 中，并保留了引用。

在 MJRefresh 中，所有的刷新控件都是继承自一个公共的父类 **MJRefreshComponent**，如果要实现自定义的控件，可以通过继承它，重写它的代码。**MJRefreshComponent** 还重写 UIView 的方法：

```objc
- (void)willMoveToSuperview:(UIView *)newSuperview {
    [super willMoveToSuperview:newSuperview];
    
    // 如果不是UIScrollView，不做任何事情
    if (newSuperview && ![newSuperview isKindOfClass:[UIScrollView class]]) return;
    
    // 旧的父控件移除监听
    [self removeObservers];
    
    if (newSuperview) { // 新的父控件
        // 设置宽度
        self.mj_w = newSuperview.mj_w;
        // 设置位置
        self.mj_x = -_scrollView.mj_insetL;
        
        // 记录UIScrollView
        _scrollView = (UIScrollView *)newSuperview;
        // 设置永远支持垂直弹簧效果
        _scrollView.alwaysBounceVertical = YES;
        // 记录UIScrollView最开始的contentInset
        _scrollViewOriginalInset = _scrollView.mj_inset;
        
        // 添加监听
        [self addObservers];
    }
}
```

当刷新控件添加到 ScrollView 上时，即 **setMj_header** 的 **[self insertSubview:mj_header atIndex:0];** 将会触发 **willMoveToSuperview** ，然后还会触发：

```objc
- (void)layoutSubviews {
    [self placeSubviews];
    
    [super layoutSubviews];
}
```
子控件通过重写 **placeSubviews** 来布局子控件：

```objc
// MJRefreshNormalHeader.m
- (void)placeSubviews {
    [super placeSubviews];
    
    // 箭头的中心点
    CGFloat arrowCenterX = self.mj_w * 0.5;
    if (!self.stateLabel.hidden) {
        CGFloat stateWidth = self.stateLabel.mj_textWith;
        CGFloat timeWidth = 0.0;
        if (!self.lastUpdatedTimeLabel.hidden) {
            timeWidth = self.lastUpdatedTimeLabel.mj_textWith;
        }
        CGFloat textWidth = MAX(stateWidth, timeWidth);
        arrowCenterX -= textWidth / 2 + self.labelLeftInset;
    }
    CGFloat arrowCenterY = self.mj_h * 0.5;
    CGPoint arrowCenter = CGPointMake(arrowCenterX, arrowCenterY);
    
    // 箭头
    if (self.arrowView.constraints.count == 0) {
        self.arrowView.mj_size = self.arrowView.image.size;
        self.arrowView.center = arrowCenter;
    }
        
    // 圈圈
    if (self.loadingView.constraints.count == 0) {
        self.loadingView.center = arrowCenter;
    }
    
    self.arrowView.tintColor = self.stateLabel.textColor;
}
```

**willMoveToSuperview** 方法中还添加了监听，监听 **UIScrollView** 的 **contentInset** 和 **contentOffset**：

```objc
// MJRefreshComponent.m
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context {
    
    // 遇到这些情况就直接返回
    if (!self.userInteractionEnabled) return;
    
    // 这个就算看不见也需要处理
    if ([keyPath isEqualToString:MJRefreshKeyPathContentSize]) {
        [self scrollViewContentSizeDidChange:change];
    }
    
    // 看不见
    if (self.hidden) return;
    if ([keyPath isEqualToString:MJRefreshKeyPathContentOffset]) {
        [self scrollViewContentOffsetDidChange:change];
    } else if ([keyPath isEqualToString:MJRefreshKeyPathPanState]) {
        [self scrollViewPanStateDidChange:change];
    }
}
```


```objc
// MJRefreshHeader.m
- (void)scrollViewContentOffsetDidChange:(NSDictionary *)change {
    [super scrollViewContentOffsetDidChange:change];
    
    // 在刷新的refreshing状态
    if (self.state == MJRefreshStateRefreshing) {
        // 暂时保留
        if (self.window == nil) return;
        
        // sectionheader停留解决
        CGFloat insetT = - self.scrollView.mj_offsetY > _scrollViewOriginalInset.top ? - self.scrollView.mj_offsetY : _scrollViewOriginalInset.top;
        insetT = insetT > self.mj_h + _scrollViewOriginalInset.top ? self.mj_h + _scrollViewOriginalInset.top : insetT;
        self.scrollView.mj_insetT = insetT;
        
        self.insetTDelta = _scrollViewOriginalInset.top - insetT;
        return;
    }
    
    // 跳转到下一个控制器时，contentInset可能会变
     _scrollViewOriginalInset = self.scrollView.mj_inset;
    
    // 当前的contentOffset
    CGFloat offsetY = self.scrollView.mj_offsetY;
    // 头部控件刚好出现的offsetY
    CGFloat happenOffsetY = - self.scrollViewOriginalInset.top;
    
    // 如果是向上滚动到看不见头部控件，直接返回
    // >= -> >
    if (offsetY > happenOffsetY) return;
    
    // 普通 和 即将刷新 的临界点
    CGFloat normal2pullingOffsetY = happenOffsetY - self.mj_h;
    CGFloat pullingPercent = (happenOffsetY - offsetY) / self.mj_h;
    
    if (self.scrollView.isDragging) { // 如果正在拖拽
        self.pullingPercent = pullingPercent;
        if (self.state == MJRefreshStateIdle && offsetY < normal2pullingOffsetY) {
            // 转为即将刷新状态
            self.state = MJRefreshStatePulling;
        } else if (self.state == MJRefreshStatePulling && offsetY >= normal2pullingOffsetY) {
            // 转为普通状态
            self.state = MJRefreshStateIdle;
        }
    } else if (self.state == MJRefreshStatePulling) {// 即将刷新 && 手松开
        // 开始刷新
        [self beginRefreshing];
    } else if (pullingPercent < 1) {
        self.pullingPercent = pullingPercent;
    }
}
```

👆方法中根据 **contentOffset** 的位置计算刷新控件的状态更新控件的UI：

```objc
// MJRefreshComponent.m
- (void)setState:(MJRefreshState)state {
    _state = state;
    
    // 加入主队列的目的是等setState:方法调用完毕、设置完文字后再去布局子控件
    dispatch_async(dispatch_get_main_queue(), ^{
        [self setNeedsLayout];
    });
}
```

```objc
// MJRefreshHeader.m
- (void)setState:(MJRefreshState)state {
    MJRefreshCheckState
    
    // 根据状态做事情
    if (state == MJRefreshStateIdle) {
        if (oldState != MJRefreshStateRefreshing) return;
        
        // 保存刷新时间
        [[NSUserDefaults standardUserDefaults] setObject:[NSDate date] forKey:self.lastUpdatedTimeKey];
        [[NSUserDefaults standardUserDefaults] synchronize];
        
        // 恢复inset和offset
        [UIView animateWithDuration:MJRefreshSlowAnimationDuration animations:^{
            self.scrollView.mj_insetT += self.insetTDelta;
            
            // 自动调整透明度
            if (self.isAutomaticallyChangeAlpha) self.alpha = 0.0;
        } completion:^(BOOL finished) {
            self.pullingPercent = 0.0;
            
            if (self.endRefreshingCompletionBlock) {
                self.endRefreshingCompletionBlock();
            }
        }];
    } else if (state == MJRefreshStateRefreshing) {
         dispatch_async(dispatch_get_main_queue(), ^{
            [UIView animateWithDuration:MJRefreshFastAnimationDuration animations:^{
                CGFloat top = self.scrollViewOriginalInset.top + self.mj_h;
                // 增加滚动区域top
                self.scrollView.mj_insetT = top;
                // 设置滚动位置
                CGPoint offset = self.scrollView.contentOffset;
                offset.y = -top;
                [self.scrollView setContentOffset:offset animated:NO];
            } completion:^(BOOL finished) {
                [self executeRefreshingCallback];
            }];
         });
    }
}
```

如果正在刷新，增大 contentInset ，让 Header 保留在屏幕上，执行 Callback 结束后恢复原来位置。

## 总结

从上面可以得出结论，可以看出来下拉刷新的过程大致是：
初始化下拉刷新控件 -> 设置Frame，布局子控件 -> 控件添加监听 -> 监控contentOffset -> 判断contentOffset并做出相应回调。


