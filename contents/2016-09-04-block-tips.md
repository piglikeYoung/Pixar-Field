
## 原文链接
* [南峰子_老驴#iOS知识小集](http://huati.weibo.com/k/iOS%E7%9F%A5%E8%AF%86%E5%B0%8F%E9%9B%86?from=501)
* [Why do nil / NULL blocks cause bus errors when run?](http://stackoverflow.com/questions/4145164/why-do-nil-null-blocks-cause-bus-errors-when-run)

iOS开发中使用到`block`的机会非常多，经常会遇到block为nil或者NULL时程序会崩溃，
例如
![Snip20160904_1](http://p44bkxib3.bkt.clouddn.com/Snip20160904_1.png)
![Snip20160904_2](http://p44bkxib3.bkt.clouddn.com/Snip20160904_2.png)


但是别的Objective-C对象为空就不会出现崩溃的情况
```Objc
NSArray *foo = nil;
NSLog(@"%zd", [foo count]); // 正常运行
```
所以，每次调用block之前我们都需要判断block是否存在
```Objc
if (mBlock) {
	mBlock();
}
```
## 分析异常
调用nil的block会崩溃，抛出的异常是`EXC_BAD_ACCESS(code=1, address=0xc)`（32位系统是0xc，64位是0x10）。这个异常表示程序在试图读取内存地址为0xc的信息时出错。

在定一个block时，编译器在栈上创建了个结构体，结构体类似
```c
struct Block_layout {
    void *isa;
    int flags;
    int reserved;
    void (*invoke)(void *, ...);
    struct Block_descriptor *descriptor;
    /* Imported variables. */
};
```
block就是指向这个结构体指针。其中的`invoke`就是指向具体实现的函数指针。当block被调用时，程序最终会跳转到这个函数指针(`invoke`)指向的代码区。而当block为nil时，程序就会试图去读取0xc地址的信息，而这个地址什么都不会有(duff address)，于是抛出一个segmentation fault。在32位系统下，之所以是0xc，是因为invoke前面的三个成员变量的大小正好是12。

所以我们在使用block时，应该首先去判断block是否为空。一种比较优雅的写法是：

```Objc
!block ?: block()
```



