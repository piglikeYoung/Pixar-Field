
## 前言

[React](https://github.com/facebook/react) 应该算是目前最火的前端框架，通过它衍生出来的 [ReactNative](https://github.com/facebook/react-native) 几乎颠覆了人们对 Web APP 的认识。其实2013年5月 **React** 就开源了，而 **ReactNative** 也在 2015年开源了，我在业余时间学习了下 **ReactNative** 基本语法, 希望能在未来的工作使用上。

## React简介
React 起源于 Facebook 公司，初期用于 instagram 网站开发，React 是一个用于构建用户界面的JavaScript库，不是一个MVC框架，提出了一种新的开发模式和理念，它强调的是“用户界面”。

### 作为UI
React 可以作为MVC中的view层进行使用，并且在已有项目中很容易使用React开发新功能。

### 虚拟DOM
虚拟DOM是React最重要的特性，实现了优化视图的渲染和刷新，以前没有ajax技术时，web页面从服务器整体渲染出html输出到浏览器进行渲染，用户的一个操作也会刷新整个页面。直到ajax的出现，实现页面局部刷新，带来的高效和分离让web开发们惊叹不已，但是又有新的问题出现，赋值的用户交互和展现需要通过大量的DOM操作来完成，这让页面的性能以及开发的效率又出现了新的瓶颈，如何进行高性能的复杂DOM操作通常是衡量一个前端开发人员技能的重要指标。

时至今日，谈到前端性能优化，减少DOM元素，减少reflow和repaint，编码过程中尽可能减少DOM查询等等而页面的任何UI的变化都是通过整体的刷新来完成的，而React之所以快，是因为它不是直接操作DOM，而是引进虚拟的DOM实现来解决这个问题。

基于React进行开发时所有的DOM构建都是通过虚拟的DOM进行，每当数据发生变化，React都会重新构建一个完整的虚拟DOM，虚拟DOM是内存数据，React通过自己实现的 DOM Diff算法将当前的虚拟DOM树和上一次的DOM进行对比，得到两个DOM结构的差异，然后仅仅将需要变化的部分进行实际的浏览器DOM更新，达到最小化重绘，避免不必要的DOM操作，解决了这两个公认的前端性能瓶颈，实现高效DOM渲染。

### 组件化
虚拟的DOM不仅带来简单的UI开发逻辑，同时也带来了组件化开发的思想，所谓组件，即封装起来的具有独立功能的UI部件，React推荐以组件的方式去思考UI构成，将UI上每个功能都相对独立的模块定义成组件，然后将小的组件通过组合或者嵌套的方式构成大的组件，最终完成整体UI构建。

如果说MVC的思想让你做到了视图-数据-控制器分离，那么组件化的思考方向则是带来了UI功能模块之间的分离。
对于React而言，开发者从功能的角度出发，将UI分成不同的组件，每个组件都独立封装。在React中，你按照界面模块自然划分的方式来组织和编写你的代码，，对于评论界面而言，整个UI是一个通过小组件构成的大组件，每个组件只关心自己部分的逻辑，彼此独立，React组件应该具有以下特征：可组合，可重用，可维护。

### 数据流
React 实现了单向的数据流，相对于传统的数据绑定，React更加灵活，便捷。


## React和ReactNative的关系

React用于web应用开发，ReactNative采用React方式进行移动应用开发，既有原生Native的加护体验，又能保留React自由的开发效率，使用灵活的HTML和CSS布局，使用React语法构建组件，然后同时运行在iOS和Android上，“learn once ，write anywhere”。


## 零、安装

安装 ReactNative 环境可以参考[搭建开发环境](http://reactnative.cn/docs/0.41/getting-started.html) 这篇文章，一步步来。

安装完成后，测试安装初始化了一个 **AwesomeProject** 工程，工程的结构如图：

![Snip20170305_2](http://p44bkxib3.bkt.clouddn.com/Snip20170305_2.png)

可以直接使用 命令行cd到该目录 或者 Xcode运行起来（由于我是使用Mac环境，所以文章都是以iOS作为实例）。

从图中可以看出，Android 和 iOS 是分开文件夹的，index.android.js 和index.ios.js 分别对应各自的系统，我们代码的入口就是这两个js文件。

## 一、index.ios.js

打开 **index.ios.js** 文件，可以看到分成四部分存在：

### 第一部分

```JavaScript
import React, { Component } from 'react';
import {
  AppRegistry,
  StyleSheet,
  Text,
  View
} from 'react-native';
```

导入ReactNative包，导入ReactNative组件：
* AppRegistry: JS运行所有ReactNative应用的入口
* StyleSheet: ReactNative中使用的样式表，类似CSS样式表
* 各种开发需要使用的组件

模板中使用的是ES6语法，ES5语法如下：

```JavaScript
let React = require('react-native');
let {
AppRegistry,
StyleSheet,
Text,
View
} = React;
```

当我们需要引入别的文件，使用 `require` 函数：

```JavaScript
var BookList = require('./iOS_views/book/book_list');
```

### 第二部分

```JavaScript
export default class HelloWorld extends Component {
  render() {
    return (
      <View style={styles.container}>
        <Text style={styles.welcome}>
          Welcome to 淘宝!
        </Text>
        <Text style={styles.instructions}>
          To get started, edit index.ios.js
        </Text>
        <Text style={styles.instructions}>
          Press Cmd+R to reload,{'\n'}
          Cmd+D or shake for dev menu
        </Text>
      </View>
    );
  }
}
```

创建ReactNative组件，模板中使用的是ES6的语法，render() {} 是ES6中的函数简写

ES5语法如下：

```JavaScript
var HelloWorld = React.createClass({
 render: function{
   return ();
 }
});
```

### 第三部分

```JavaScript
const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#F5FCFF',
  },
  welcome: {
    fontSize: 20,
    textAlign: 'center',
    margin: 10,
  },
  instructions: {
    textAlign: 'center',
    color: '#333333',
    marginBottom: 5,
  },
});
```
StyleSheet.create创建样式实例，在应用中只会被创建一次，不用每次在渲染周期中重新创建
  

### 第四部分

```JavaScript
AppRegistry.registerComponent('HelloWorld', () => HelloWorld);
```

注册入口组件。

* **AppRegistry**：负责注册运行ReactNative应用程序的JavaScript入口
* **registerComponent**：注册应用程序的入口组件，告知ReactNative哪一个组件被注册为应用的根容器

第二个参数使用了ES6的语法，箭头函数：

```JavaScript
// 返回的必须是定义的组件类的名字
// 等价于 function() {return HelloWorld}
() => HelloWorld 
```

## 总结

本章介绍了 ReactNative 背景以及优缺点，分析了测试安装工程的工程结构，以及index.ios.js 的代码结构，还有很多未涉及的方面，比如IDE的安装等等，想要了解的可以点下方的参考链接。

## 参考链接
* [React 入门实例教程](http://www.ruanyifeng.com/blog/2015/03/react.html)
* [也许，DOM 不是答案](http://www.ruanyifeng.com/blog/2015/02/future-of-dom.html)
* [ReactNavite官网](https://facebook.github.io/react-native/docs/getting-started.html)
* [ReactNavite中文网](http://reactnative.cn/docs/0.41/getting-started.html)
* [ React Native开发之IDE（Atom+Nuclide）安装，运行，调试](http://blog.csdn.net/hello_hwc/article/details/51612139)
* [3分钟带你玩转React Native研发所有调试技巧](http://www.52learn.wang/archives/1071?utm_source=tuicool&utm_medium=referral)
* [学习 React Native for Android：环境搭建](http://hahack.com/codes/learn-react-native-for-android-01/)
* [React-Native入门指南](https://github.com/vczero/react-native-lesson)







