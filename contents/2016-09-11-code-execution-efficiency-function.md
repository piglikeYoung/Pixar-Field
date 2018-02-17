
## 前言
开发中我们经常需要检测**某个方法**或者**某段代码**具体的执行时间，有人使用NSDate来计算，但是这个类的精确度太差，你可以抛弃它了。

其实在iOS中有个`mach_absolute_time()`函数可以获取到CPU的tickcount的计数值（貌似Android也有个获取精确时间的函数），获取到纳秒级的精确度。

## 使用
先导入头文件

```objc
#import <mach/mach_time.h>
```

简单使用
```objc
uint64_t start = mach_absolute_time();
NSLog(@"%f", machTimeToSecs(start));
    
for (NSInteger i = 0; i<10000; i++) {
   [items addObject:@"hello"];
}
    
uint64_t stop = mach_absolute_time();
NSLog(@"%f", machTimeToSecs(stop));
    
NSLog(@"%f", subtractTimes(stop, start));
```

由于`mach_absolute_time()`返回的是纳米级别的时间，我们需要将转换成秒，可以使用下面的函数完成转换。

```c
// mach_absolute_times going in, seconds out
double machTimeToSecs(uint64_t time)
{
    mach_timebase_info_data_t timebase;
    mach_timebase_info(&timebase);
    return (double)time * (double)timebase.numer /
    (double)timebase.denom / 1e9;
}
```

或者直接点，将开始和结束的时间都传入，返回计算并转换好的秒数。

```c
//Raw mach_absolute_times going in,difference in seconds out
double subtractTimes( uint64_t endTime, uint64_t startTime )
{
    uint64_t difference = endTime - startTime;
    static double conversion = 0.0;
    if( conversion == 0.0 )
    {
        mach_timebase_info_data_t info;
        kern_return_t err =mach_timebase_info( &info );
        //Convert the timebase into seconds
        if( err == 0  )
            conversion= 1e-9 * (double) info.numer / (double) info.denom;
    } 
    return conversion * (double)difference; 
}
```

## 参考链接
* [性能和时间](http://www.cocoachina.com/industry/20130608/6362.html)
* [Quick Performance Measurements](http://iosdeveloperzone.com/2011/05/03/quick-performance-measurements/)


