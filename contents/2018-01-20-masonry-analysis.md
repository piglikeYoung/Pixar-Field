
## 前言

在 iOS 开发中 Masonry 是控件布局中经常使用的一个轻量级框架布局控件，它简化了手写 AutoLayout 代码的难度，本文就分析一下 Masonry 框架的源码。

## Masonry 与 NSLayoutConstraint 调用方式的对比

### 1.NSLayoutConstraint

给一个 View 设置相对父控件的顶部距离 `10个pt`：

```objc
[NSLayoutConstraint constraintWithItem:view1
                                 attribute:NSLayoutAttributeTop
                                 relatedBy:NSLayoutRelationEqual
                                    toItem:superview
                                 attribute:NSLayoutAttributeTop
                                multiplier:1.0
                                  constant:10]
```
代码相当直观的说明的约束关系。

### 2.Masonry

Masonry 是通过链式调用和匿名闭包的方式来简化设置约束：

```objc
[view1 mas_makeConstraints:^(MASConstraintMaker *make) {
    make.top.equalTo(superview.mas_top).with.offset(10); 
    // 等价于
    make.top.equalTo(@10);
    // 等价于
    make.top.equalTo(superview.mas_top).multipliedBy(1).offset(10); 
}];
```

## 类结构

### 1. View+MASAdditions

上面使用到的 **mas_makeConstraints** 是 UIView 扩展 **UIView+MASAdditions** 提供给的方法，这个类提供了一系列 **MASViewAttribute** 成员属性，和四个公开的方法： **mas_closestCommonSuperview** 寻找两个视图最近的公共父类、 **mas_makeConstraints** 创建安装约束、 **mas_updateConstraints** 更新已经存在的约束（不存在的就创建）、 **mas_remakeConstraints** 移除已经创建的约束并添加新的约束。

### 2. MASViewAttribute

**MASViewAttribute** 内部有三个属性和三个方法，从这些属性和方法可以看出，它就是对 UIView 和 NSLayoutAttribute 的封装，`MASViewAttribute = UIView + NSLayoutAttribute + item`，view 属性就是要约束的控件，item 就是控件上约束的部分。

item 属性就是在创建 NSLayoutConstriant 构造方法时 **constraintWithItem** 与 **toItem** 的参数。对于 UIView 来说该 item 就是 UIView 本身。而对于UIViewController，该出 Item 就 topLayoutGuide，bottomLayoutGuide 。

该类中除了初始化方法外还有一个 **isSizeAttribute** 方法，该方法用来判断 MASViewAttribute 类中的 **layoutAttribute** 属性是否是 **NSLayoutAttributeWidth** 或者**NSLayoutAttributeHeight**，如果是Width或者Height的话，那么约束就添加到当前View上，而不是添加在父视图上。

### 3. MASViewConstraint

**MASViewConstraint** 是对 **NSLayoutConstraint** 的进一步封装，对控件设置约束，底层还是使用了 **NSLayoutConstraint** 来设置约束，Masonry 只是简化了代码，所以 **MASViewConstraint** 最重要的事情就是创建 **NSLayoutConstraint** 并设置给相应的视图。**总的来说 MASViewAttribute 是对 View 与 NSLayoutAttribute 进行的封装，所以 MASViewConstraint 类要依赖于 MASViewAttribute 类。**

**MASConstraint** 是 **MASViewConstraint** 的父类，它是个抽象父类，不能实例化，它定义了很多公共接口方法。

**MASConstraint** 还有个子类是 **MASCompositeConstraint** ，它里面只有一个方法 **- (id)initWithChildren:(NSArray *)children;**，其实就是存储一系列约束的类，内部有个私有属性数组 **childConstraints** 存储约束。总的来说是对它就是一个存储 **MASViewAttribute** 对象数组的封装。

### 4. MASConstraintMaker

**MASConstraintMaker** 是个工厂类，负责创建 **MASConstraint** 类型的对象。在 **View+MASAdditions** 中 **mas_makeConstraints** 的 block 参数就是 **MASConstraintMaker** 对象。用户可以通过 block 回调的 **MASConstraintMaker** 对象 给 View 指定约束。类的内部有个 **constraints** 属性来记录它创建的所有 **MASConstraint** 对象。

## View+MASAdditions源码解析

## 1.主要属性和getter方法


```objc
@property (nonatomic, strong, readonly) MASViewAttribute *mas_left;
@property (nonatomic, strong, readonly) MASViewAttribute *mas_top;
@property (nonatomic, strong, readonly) MASViewAttribute *mas_right;
@property (nonatomic, strong, readonly) MASViewAttribute *mas_bottom;
......
```

上面是部分属性，属性的类型都是 **MASViewAttribute** 类型，因为 **MASViewAttribute** 是对 View 与 NSLayoutAttribute 进行的封装，所以 **mas_left** 就表示 View 和 NSLayoutAttributeLeft，一次类推别的属性。

查看它们的 getter 方法：

```objc
#pragma mark - NSLayoutAttribute properties

- (MASViewAttribute *)mas_left {
    return [[MASViewAttribute alloc] initWithView:self layoutAttribute:NSLayoutAttributeLeft];
}

- (MASViewAttribute *)mas_top {
    return [[MASViewAttribute alloc] initWithView:self layoutAttribute:NSLayoutAttributeTop];
}

- (MASViewAttribute *)mas_right {
    return [[MASViewAttribute alloc] initWithView:self layoutAttribute:NSLayoutAttributeRight];
}

- (MASViewAttribute *)mas_bottom {
    return [[MASViewAttribute alloc] initWithView:self layoutAttribute:NSLayoutAttributeBottom];
}
```

就可以得知它所做的事情就是创建一个 **MASViewAttribute** ，也就是 **mas_left = self + NSLayoutAttributeLeft**。

### 2. mas_makeConstraints方法解析

```objc
- (NSArray *)mas_makeConstraints:(void(^)(MASConstraintMaker *))block {
	  // 关闭自动添加约束，自己手动添加
    self.translatesAutoresizingMaskIntoConstraints = NO;
    // 创建 MASConstraintMaker
    MASConstraintMaker *constraintMaker = [[MASConstraintMaker alloc] initWithView:self];
    // 给 maker 中的各种属性赋值，通过 block 进行值的回调
    block(constraintMaker);
    // 进行约束添加，并返回所有Install的约束数组
    return [constraintMaker install];
}
```

上面也说到用 **mas_makeConstraints** 来添加约束，该方法返回一个数组，数组里存放的是当前视图中添加的所有约束。因为 Masonry 对 **NSLayoutAttribute** 封装成了 **MASViewConstraint** ，所以数组中存储的全是 **MASViewConstraint** 类型的对象。

方法的参数是 **void(^)(MASConstraintMaker *)** 的匿名闭包，无返回值，需要个 **MASConstraintMaker** 对象。在方法中，首先将 **translatesAutoresizingMaskIntoConstraints** 设置为 NO，然后创建了一个 **MASConstraintMaker** 对象，通过 block 将 **MASConstraintMaker** 对象传递给用户来设置约束，也就是让用户使用 **MASConstraintMaker** 中的 **MASConstraint** 类型的属性。

### 3. mas_updateConstraints 与 mas_remakeConstraints 函数的解析

这两个函数内部的是想和 mas_makeConstraints 差不多，只是多设置一个属性。

mas_updateConstraints 中将 `updateExisting` 设置为 YES，就是说添加约束时先检查是否已经被添加，如果被添加了就更新，如果没有被添加就添加。

mas_remakeConstraints 中所做的事情是将 `removeExisting` 设置为 YES，就是说将视图上的旧约束先移除掉，在添加新的约束。

```objc
constraintMaker.updateExisting = YES;
```

```objc
constraintMaker.removeExisting = YES;
```

### 4. mas_closestCommonSuperview

**mas_closestCommonSuperview** 方法负责计算出两个视图的公共父视图，找到最近的那个公共父视图后就返回，如果找不到就返回 nil：

```objc
- (instancetype)mas_closestCommonSuperview:(MAS_VIEW *)view {
    // 临时父视图存储
    MAS_VIEW *closestCommonSuperview = nil;

    MAS_VIEW *secondViewSuperview = view;
    // 遍历 secondView 所有的父视图
    while (!closestCommonSuperview && secondViewSuperview) {
        MAS_VIEW *firstViewSuperview = self;
        // 遍历当前视图（self）的父视图
        while (!closestCommonSuperview && firstViewSuperview) {
            if (secondViewSuperview == firstViewSuperview) {
                // 找到公共父视图后结束循环
                closestCommonSuperview = secondViewSuperview;
            }
            firstViewSuperview = firstViewSuperview.superview;
        }
        secondViewSuperview = secondViewSuperview.superview;
    }
    // 返回公共父视图
    return closestCommonSuperview;
}
```

寻找两个视图的公共父视图对于约束很重要，因为相对的约束是添加到其公共父视图上的。比如  viewA.left = viewB.right + 10, 因为是 viewA 与 viewB 的相对约束，那么约束是添加在 viewA 与 viewB 的公共父视图上的，如果 viewB 是 viewA 的父视图，那么约束就添加在 viewB 上从而对 viewA 起到约束作用。

## 工厂类 MASConstraintMaker 解析

### 1. 公共属性

**MASConstraintMaker** 类里面的属性都有对应的 getter 方法：

```objc
- (MASConstraint *)left {
    return [self addConstraintWithLayoutAttribute:NSLayoutAttributeLeft];
}

- (MASConstraint *)top {
    return [self addConstraintWithLayoutAttribute:NSLayoutAttributeTop];
}
.....
```

### 2. 工厂方法

getter 方法都会调用 **addConstraintWithLayoutAttribute** ，在 **addConstraintWithLayoutAttribute** 方法中又调用了 MASConstraintMaker 工厂类的工厂方法：

```objc
- (MASConstraint *)addConstraintWithLayoutAttribute:(NSLayoutAttribute)layoutAttribute {
    return [self constraint:nil addConstraintWithLayoutAttribute:layoutAttribute];
}
```

```objc
- (MASConstraint *)constraint:(MASConstraint *)constraint addConstraintWithLayoutAttribute:(NSLayoutAttribute)layoutAttribute {
    // 创建 MASViewAttribute
    MASViewAttribute *viewAttribute = [[MASViewAttribute alloc] initWithView:self.view layoutAttribute:layoutAttribute];
    // 根据 layoutAttribute 创建 MASViewConstraint
    MASViewConstraint *newConstraint = [[MASViewConstraint alloc] initWithFirstViewAttribute:viewAttribute];
    // constraint 不为 nil时，就和 newConstraint 合并为 MASCompositeConstraint 对象
    if ([constraint isKindOfClass:MASViewConstraint.class]) {
        //replace with composite constraint
        NSArray *children = @[constraint, newConstraint];
        MASCompositeConstraint *compositeConstraint = [[MASCompositeConstraint alloc] initWithChildren:children];
        compositeConstraint.delegate = self;
        [self constraint:constraint shouldBeReplacedWithConstraint:compositeConstraint];
        return compositeConstraint;
    }
    if (!constraint) {
        // 设置代理 MASConstraintDelegate，链式调用的核心，让 MASViewConstraint 也能调用这个方法
        newConstraint.delegate = self;
        // 添加进数组
        [self.constraints addObject:newConstraint];
    }
    return newConstraint;
}
```

上面代码根据提供的参数创建 MSAViewConstraint 对象，如果该函数的第一个参数不为空的话就会将新创建的 MSAViewConstraint 对象与参数进行合并组合成 **MASCompositeConstraint** 类（ MASCompositeConstraint 本质上是 MSAViewConstraint 对象的数组）的对象。

创建完 MASConstraint 类的相应的对象后，会把该创建的对象添加进 MASConstraintMaker 工厂类的私有 constraints 数组，来记录该工厂对象创建的所有约束。`newConstraint.delegate = self;` 这句话是非常重要的，由于为 MASConstraint 对象设置了代理，所以才支持链式调用。

举个例子：maker.top.left.right.equalTo(@10)，第一步 maker.top 调用返回了 **MASViewConstraint** 类的对象，当调用 .left 时，其实就是 newConstraint.left ，这意味着，.left 并不是调用 maker 的 left 的 getter 方法，而是 **MASViewConstraint** 的 left 的 getter 方法：

```objc
// MASConstraint.m
- (MASConstraint *)left {
    return [self addConstraintWithLayoutAttribute:NSLayoutAttributeLeft];
}
```

```objc
// MASViewConstraint.m
- (MASConstraint *)addConstraintWithLayoutAttribute:(NSLayoutAttribute)layoutAttribute {
    NSAssert(!self.hasLayoutRelation, @"Attributes should be chained before defining the constraint relation");

    return [self.delegate constraint:self addConstraintWithLayoutAttribute:layoutAttribute];
}
```

从上面的代码可以看出，调用 **MASViewConstraint** 的 left 的 getter 方法其实就是调用 delegate 的 **addConstraintWithLayoutAttribute** 方法，而 delegate 就是前面提到的 `maker`，从而实现了链式调用。

### 3. install 方法

```objc
// 添加约束到View上
- (NSArray *)install {
    // 如果是mas_remarkConstraint，先将该视图上的约束 uninstall
    if (self.removeExisting) {
        // 获取当前视图上添加的所有约束
        NSArray *installedConstraints = [MASViewConstraint installedConstraintsForView:self.view];
        // 移除掉所有约束
        for (MASConstraint *constraint in installedConstraints) {
            [constraint uninstall];
        }
    }
    
    // 添加约束
    // self.constraints 中存储的是通过 Block 中配置的参数
    NSArray *constraints = self.constraints.copy;
    for (MASConstraint *constraint in constraints) {
        // 是否为 更新的约束
        constraint.updateExisting = self.updateExisting;
        // install 每个约束
        [constraint install];
    }
    [self.constraints removeAllObjects];
    return constraints;
}
```

从上面的代码可以看出，install 方法就是遍历工厂对象所创建所有约束对象，并调用每个约束对象的 isntall 方法来添加约束。

在安装约束时，如果 self.removeExisting == YES，那么就先 uninstall 旧约束，再 install 新约束。在安装约束时，将 updateExisting 赋值给每个约束，每个约束在调用本身的 install 方法时会判断是否更新。

## MASViewConstraint 解析

**MASConstraintMaker** 工厂类所创建的对象实质上是 **MASViewConstraint** 类的对象。而 **MASViewConstraint** 类实质上是对 **MASLayoutConstraint** 的封装，进一步说 **MASViewConstraint** 负责为 **MASLayoutConstraint** 构造器组织参数并创建 **MASLayoutConstraint** 的对象，并将该对象添加到相应的视图中。

### 1. 链式调用

我们设置约束的时候经常这么使用： **constraint.top.left.equalTo(superView).offset(10);** ，在 Objective-C 中这不是方法调用的形式，在 Objective-C 中的调用形式是 `[]` ，但是 Masonry 使用了 `()`，我们来看下怎么做到的：

```objc
- (MASConstraint *)addConstraintWithLayoutAttribute:(NSLayoutAttribute)layoutAttribute {
    NSAssert(!self.hasLayoutRelation, @"Attributes should be chained before defining the constraint relation");

    return [self.delegate constraint:self addConstraintWithLayoutAttribute:layoutAttribute];
}
``` 
这个在上面说到，调用类似 left、Top 等属性的时候其实是通过 delegate 去调用 maker 工厂类的工厂方法来创建 MASConstraint 对象。

但是 **.offset(10)** 是怎么做到的，因为 OC 中不能通过小括号来调用方法：

```objc
- (MASConstraint * (^)(CGFloat))offset {
    return ^id(CGFloat offset){
        self.offset = offset;
        return self;
    };
}
```

原来 offset() 是个闭包，它的返回值是个匿名block，参数是个 CGFloat 类型。总的来说 offset() = offset + ()，即先调用了 offset 方法，然后通过 () 调用了返回的匿名 block，闭包返回了 MASConstraint 对象。

### 2. install 方法解析

```objc
MASLayoutConstraint *layoutConstraint
        = [MASLayoutConstraint constraintWithItem:firstLayoutItem
                                        attribute:firstLayoutAttribute
                                        relatedBy:self.layoutRelation
                                           toItem:secondLayoutItem
                                        attribute:secondLayoutAttribute
                                       multiplier:self.layoutMultiplier
                                         constant:self.layoutConstant];
                                         
layoutConstraint.priority = self.layoutPriority;
layoutConstraint.mas_key = self.mas_key;
```

install 方法创建 **MASLayoutConstraint** 对象，并将该对象添加到相应的 View 上。

上面的代码就是根据获取的属性来创建 MASLayoutConstraint 对象，MASLayoutConstraint 就是继承自 NSLayoutConstraint。

```objc
// 寻找要添加约束的 View
if (self.secondViewAttribute.view) {
		  // 寻找两个视图的公共父视图
        MAS_VIEW *closestCommonSuperview = [self.firstViewAttribute.view mas_closestCommonSuperview:self.secondViewAttribute.view];
        NSAssert(closestCommonSuperview,
                 @"couldn't find a common superview for %@ and %@",
                 self.firstViewAttribute.view, self.secondViewAttribute.view);
        self.installedView = closestCommonSuperview;
    } else if (self.firstViewAttribute.isSizeAttribute) {
        self.installedView = self.firstViewAttribute.view;
    } else {
        self.installedView = self.firstViewAttribute.view.superview;
    }
```

上面的代码就是创建约束对象后，寻找要添加约束的 View，如果是两个视图相对约束，就获取两种的公共父视图。如果添加的是 Width 或者 Height ，那么就添加到当前视图上。如果既没有指定相对视图，也不是 Size 类型的约束，那么就将该约束对象添加到当前视图的父视图上。

创建完约束对象 **MASLayoutConstraint** ，并且找到要添加约束的 View 后，就要开始添加约束了。添加前判断是不是对约束的更新，如果约束已经存在就更新约束，如果不存在就进行添加。添加成功后通过 **mas_installedConstraints** 属性记录一下本安装的约束。mas_installedConstraints 是通过运行时为 UIView 关联的一个 NSMutable 类型的属性，用来记录约束该视图的所有约束。
参考下面代码：

```objc
MASLayoutConstraint *existingConstraint = nil;
    if (self.updateExisting) {
        existingConstraint = [self layoutConstraintSimilarTo:layoutConstraint];
    }
    if (existingConstraint) {
        // 更新约束
        existingConstraint.constant = layoutConstraint.constant;
        self.layoutConstraint = existingConstraint;
    } else {
        // 添加约束
        [self.installedView addConstraint:layoutConstraint];
        self.layoutConstraint = layoutConstraint;
        // 约束所在的 View 记录被添加的约束
        [firstLayoutItem.mas_installedConstraints addObject:self];
    }
```

### 3. UIView 的私有类目 UIView+MASConstraints

MASViewConstraint 中定义了一个 UIView 的私有扩展 `UIView+MASConstraints`，它的作用是为 UIView 通过运行时来关联一个 **NSMutableSet** 类型的 **mas_installedConstraints** 属性。该属性记录了该 View 的所有约束。

```objc
/*
	私有类，只有 MASViewConstraint 使用它
*/
@interface MAS_VIEW (MASConstraints)

@property (nonatomic, readonly) NSMutableSet *mas_installedConstraints;

@end

@implementation MAS_VIEW (MASConstraints)

// 动态添加属性的 Key， 用来表示动态添加的属性
static char kInstalledConstraintsKey;

- (NSMutableSet *)mas_installedConstraints {
    // 通过 kInstalledConstraintsKey 来获取已经动态添加的约束集合
    NSMutableSet *constraints = objc_getAssociatedObject(self, &kInstalledConstraintsKey);
    // 如果不存在，创建一个，并进行动态绑定
    if (!constraints) {
        constraints = [NSMutableSet set];
        objc_setAssociatedObject(self, &kInstalledConstraintsKey, constraints, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }
    return constraints;
}

@end
```

## 总结

本文简单的对 Masonry 的核心代码进行分析，更加细节的东西还请自行去阅读源码。

