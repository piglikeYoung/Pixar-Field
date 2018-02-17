
## 前言
浏览微博时，看到一篇关于`遍历中修改可变数组`Tips，觉得很实用，在此记录下来。

原出处：[南峰子_老驴#iOS知识小集](http://huati.weibo.com/k/iOS%E7%9F%A5%E8%AF%86%E5%B0%8F%E9%9B%86?from=501)

## 问题
相信每个开发都会遇到在遍历中修改可变数组的情况，稍不小心就会数组越界，**我们不建议在遍历可变数组时修改数组**，包括添加，删除，替换元素。

不过，如果真的要在遍历的同时删除元素，可以采用`倒序遍历`，不会引起崩溃。

```objc
NSMutableArray *array = [NSMutableArray arrayWithObjects:@"a", @"b", @"c", @"d", @"e", nil];
[array enumerateObjectsWithOptions:NSEnumerationReverse usingBlock:^(id  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
   if ([obj isEqualToString:@"c"] || [obj isEqualToString:@"e"]) {
       [array removeObject:obj];
   }
}];
    
NSLog(@"%@", array);// a, b, d
```


```objc
NSMutableArray *array = [NSMutableArray arrayWithObjects:@"a", @"b", @"c", @"d", @"e", nil];
for (NSString *obj in array.reverseObjectEnumerator) {
if ([obj isEqualToString:@"a"] || [obj isEqualToString:@"e"] || [obj isEqualToString:@"c"]) {
       [array removeObject:obj];
   }
}
NSLog(@"%@", array);// b, d
```

## 分析原因
无论是`for...in...` 还是 `enumerateObjectsWithOptions`    都可以认为是先获取了数组的大小 **int count = array.count**，然后**for(int i=0;i<count;i++)**

假设count是10，这时候你在循环时删除了一个元素，数组剩余9个，你能获取到的index最多到8，但是你的循环还是执行到i到9为止，之后就越界了。

如果是倒序 **for(int i=count-1;i>=0;i--)**，这么遍历就算把数组元素全部删除也不会有问题。
你调用枚举器遍历数组时count只会获取一次，且中途不能修改。

**如果非要顺序删除也有方法，你删完立刻break。**

## Update 2016-9-11
之前的文章说到遍历中修改可变数组可能会造成Crash，但是亲身尝试了发现在一些情况下并不会有什么问题。

我们常用的遍历数组的方式大概有三种：`for`，`for in`，`enumerateObjectsUsingBlock`。

### for 和 enumerateObjectsUsingBlock
在这三种方法中，使用`for`和`enumerateObjectsUsingBlock`遍历时，可以修改数组，不会有Crash等问题。

```objc
NSMutableArray *array = [NSMutableArray arrayWithCapacity:8];
[array addObject:@"1"];
[array addObject:@"2"];
[array addObject:@"3"];
[array addObject:@"4"];
[array addObject:@"5"];
    
[array enumerateObjectsUsingBlock:^(id  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
   if ([obj isEqualToString:@"3"]) {
       [array removeObject:obj];
   }
}];
    
NSLog(@"%@", array);// 1,2,4,5
    
for (NSInteger i = 0; i<array.count; i++) {
   NSString *temp = array[i];
   if ([temp isEqualToString:@"2"]) {
       [array removeObject:temp];
       [array addObject:@"8"];
   }
}
    
NSLog(@"%@", array);// 1,4,5,8
```

**上述代码虽然没有造成Crash，但是有个潜在的问题，如果遍历时删除元素，可能导致后面的元素不会被遍历到**。

```objc
NSMutableArray *array = [NSMutableArray arrayWithCapacity:8];
[array addObject:@"1"];
[array addObject:@"2"];
[array addObject:@"3"];
[array addObject:@"4"];
[array addObject:@"5"];
    
[array enumerateObjectsUsingBlock:^(id  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
   if ([obj isEqualToString:@"3"]) {
       [array removeObject:obj];
   }
   
   if ([obj isEqualToString:@"4"]) { // 无效
       [array removeObject:obj];
   }
}];
    
NSLog(@"%@", array);// 1,2,4,5
    
for (NSInteger i = 0; i<array.count; i++) {
   NSString *temp = array[i];
   if ([temp isEqualToString:@"2"]) {
       [array removeObject:temp];
   }
   
   if ([temp isEqualToString:@"4"]) { // 无效
       [array addObject:@"8"];
   }
}
    
NSLog(@"%@", array);// 1,4,5
```

虽然遍历时修改数组不会造成Crash，但是**我们还是不建议在遍历可变数组时修改数组**。

### for in
当使用`for in`来遍历时，修改元素会抛出NSGenericException异常。

```objc
for (NSString *str in array) { // Crash 
   if ([str isEqualToString:@"2"]) {
       [array addObject:@"8"];
   }
}
```
![Snip20160911_1](http://p44bkxib3.bkt.clouddn.com/Snip20160911_1.png)

一个通用的做法是：`从原数组copy一个数组出来，然后遍历copy数组，过滤出自己需要的元素，最后再对原数组进行操作`。

```objc
NSArray *temp = [array copy]; // 拷贝一份
for (NSString *str in temp) { // 遍历拷贝
   if ([str isEqualToString:@"2"]) {
       [array addObject:@"8"]; // 操作原数组
       [array removeObject:str];
   }
}
    
NSLog(@"%@", array); // 1,3,4,5,8
```

## 参考链接
[iOS : Modifying NSFastEnumerationState to hide mutation while enumerating](http://stackoverflow.com/questions/20069825/ios-modifying-nsfastenumerationstate-to-hide-mutation-while-enumerating)


