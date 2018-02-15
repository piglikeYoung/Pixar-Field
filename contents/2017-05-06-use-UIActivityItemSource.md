
## 前言

最近产品提了个需求，要求能够针对多个APP定制分享，如果用户没有装某个APP，则不需要显示分享。看到这个需求我第一时间想到使用iOS原生的分享，因为它不需要APP引入各种分享的SDK就能达到分享的目的，虽然UI无法定制，但是内容可以定制，并且用户安装了哪些APP，系统能够自动判定，但是缺点也很明显，假如想要分享的APP没有支持 `Share Extension` 那么就无法使用分享了。

## UIActivityViewController

iOS 从3.2就提供了 `UIDocumentInteractionController` 支持同设备上app之间的文档分享外，还可以实现文档的预览、打印、发邮件以及复制，使用起来也非常简单。但是 iOS 6 之后苹果有提供了功能更加强大的 `UIActivityViewController`。它俩的区别可以参考：

* [《Tutorial: Sharing Data Locally Between iOS Apps》](Tutorial: Sharing Data Locally Between iOS Apps)
* [《UIActivityViewController vs UIDocumentInteractionController in iOS》](http://stackoverflow.com/questions/21234699/uiactivityviewcontroller-vs-uidocumentinteractioncontroller-in-ios)

![WX20170513_234019](http://p44bkxib3.bkt.clouddn.com/WX20170513_234019.png)

如图，第一排是 AirDrop 能够近距离分享文件，照片等内容；第二排就是待会要讲的 `Share Extension` ；第三排就是 `Action Extension` ；

显示在 Share Extension 上的APP都是系统自己识别的，并且只有第三方APP支持 Share Extensions 才会显示，在列表的最末端有三个点的按钮，点开它可以看到支持 Share Extensions 的全部APP列表。

使用 `UIActivityViewController` 非常简单，创建它并通过modal的形式弹出来：

```objc
UIActivityViewController *activityController = [[UIActivityViewController alloc] initWithActivityItems:@[[UIImage imageNamed:@"share"]] applicationActivities:nil];
activityController.excludedActivityTypes = @[UIActivityTypeAirDrop];
        
//        [activityController setCompletionWithItemsHandler:^(NSString * __nullable activityType, BOOL completed, NSArray * __nullable returnedItems, NSError * __nullable activityError){
//            
//        }];
[self presentViewController:activityController animated:YES completion:nil];
```

第一个数组内的对象代表的是我们想要操作的数据的一些内容，而且这个数组不能为空，比如我们PDF文档的名称、URL、、文本或者图片；第二个数组指定了泛型，数组内的对象必须是UIActivity类型的对象，代表的是iOS系统支持的我们自定义的服务，关于这点跟自定义UIActivity服务的内容有关，目前没用上，现在我们暂时置为nil。

执行代码后，你会看到弹出和上图一样的控制器，这时候你就有疑问了，可分享的平台这么多，我怎么定制根据分享的app不同，内容也不同呢？

## 定制分享内容

### 分享文本
定制分享内容你需要创建一个对象，实现 `UIActivityItemSource` 协议：

```objc
@interface LSCustomizedStringItemSource: NSObject <UIActivityItemSource>
@property (nonatomic, copy) NSString *partnerCode;
@end

@implementation LSCustomizedStringItemSource

- (id)activityViewControllerPlaceholderItem:(UIActivityViewController *)activityViewController {
    return @"";
}

- (id)activityViewController:(UIActivityViewController *)activityViewController itemForActivityType:(NSString *)activityType {

    if ([activityType isEqualToString:UIActivityTypePostToFacebook]) {
        return @"Facebook";
    } else if ([activityType isEqualToString:UIActivityTypePostToTwitter]) {
        return @"Twitter";
    } else if ([activityType isEqualToString:UIActivityTypeMessage]) {
        return @"Message";
    } else if ([activityType isEqualToString:UIActivityTypeMail]) {
        return @"Mail";
    }
    return @"Default";
}

@end
```

上面是自定义的文本分享内容，根据平台的不同，分享不同的文本。

### 分享图片
```objc
@interface LSCustomizedImageItemSource: NSObject <UIActivityItemSource>
@end

@implementation LSCustomizedImageItemSource

- (id)activityViewControllerPlaceholderItem:(UIActivityViewController *)activityViewController {
    return @"";
}

- (id)activityViewController:(UIActivityViewController *)activityViewController itemForActivityType:(NSString *)activityType {
    if ([activityType isEqualToString:UIActivityTypeMessage]) {
        return nil;
    }
    return [UIImage imageNamed:@"share"];
}

@end
```

上面是自定义的图片分享，分享到 iMessage 的时候，不需要分享图片

### 分享URL

```objc
@interface LSCustomizedURLItemSource: NSObject <UIActivityItemSource>
@end

@implementation LSCustomizedURLItemSource

- (id)activityViewControllerPlaceholderItem:(UIActivityViewController *)activityViewController {
    return @"";
}

- (id)activityViewController:(UIActivityViewController *)activityViewController itemForActivityType:(NSString *)activityType {
    
    return [NSURL URLWithString:@"https://www.taobao.com"];
}

@end
```

### 重新设置 UIActivityViewController


```objc
LSCustomizedStringItemSource *stringItemSource = [[LSCustomizedStringItemSource alloc] init];
LSCustomizedURLItemSource *urlItemSource = [[LSCustomizedURLItemSource alloc] init];
LSCustomizedImageItemSource *imageItemSource = [[LSCustomizedImageItemSource alloc] init];

UIActivityViewController *activityController = [[UIActivityViewController alloc] initWithActivityItems:@[stringItemSource, urlItemSource, imageItemSource] applicationActivities:nil];
```

创建自定义分享内容，并赋值给 UIActivityViewController，这样就可以根据APP的不同，分享不同的内容。

## 总结

使用 UIActivityViewController 分享，优点和缺点都非常明显，所以需要根据你的业务需求来决定是否使用。

## 参考链接

* [Give thumbnail image with UIActivityViewController](http://stackoverflow.com/questions/37468195/give-thumbnail-image-with-uiactivityviewcontroller/37548529)
* [通过UIActivityViewController实现更多分享服务](http://www.jianshu.com/p/a1c9621a3f4e)
* [研究 UIActivityViewController](http://www.cocoachina.com/industry/20140425/8233.html)
* [iOS8扩展插件开发配置](http://blog.csdn.net/phunxm/article/details/42715145)
* [实现iOS app之间的内容分享](https://segmentfault.com/a/1190000004237771s)



