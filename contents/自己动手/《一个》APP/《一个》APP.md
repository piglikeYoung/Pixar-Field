# 《一个》APP

前段时间在微博上看到[ihappyhacking](https://github.com/ihappyhacking)通过抓数据拿到《一个》的接口，写了个APP。<br/>
我本着学习的心态去学习别人的代码，并且用自己的方式去做了个类似的APP。<br/>
毕竟是站在巨人的肩膀上，花了几天时间就模仿写出来了，里面有不少借鉴地方，但是也有一些自己的东西。具体的说明请链接到原版的地址。

原版作者：[ihappyhacking](https://github.com/ihappyhacking)<br/>
原版的github地址：https://github.com/ihappyhacking/MyOne-iOS

## 和原版的区别
 - 原版每次滑动一页就去请求一次数据，而我的是第一次请求请求2条数据，然后每次请求都去请求下一条数据，这样滑动起来更加顺畅些
 - 我把每个模块的请求方法，请求参数和返回参数都分开，这样利于排查和扩展
 - 原版中右拉刷新是抽取出来，然后每个模块都加入这个模块，让他们都具有右拉刷新的功能，我的是每个模块单独写个右拉刷新(原版有些复杂，对于我的这种菜鸟有点挑战。。)
 
 ## 总结
 看到别的源码，结合自身产生了新的解决问题的方式，再次感谢[ihappyhacking](https://github.com/ihappyhacking)的分享
 
 我的《一个》APP github地址<br/>
 https://github.com/piglikeYoung/mOne-iOS

