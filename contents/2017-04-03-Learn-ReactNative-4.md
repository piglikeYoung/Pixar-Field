
## 前言

本章，我们学习新的组件 Text 、 Touchable 、 TextInput 和 Image。

## Text组件

Text组件是 ReactNative 中显示文本的组件，详细的内容可以参考[官方文档](https://facebook.github.io/react-native/docs/text.html)，这里只讲几个常用的属性。Text组件需要导入组件：

```
// onPress   手指触摸事件
// numberOfLines 显示多少行

// 可以设置字体颜色、大小、对齐方式等等

import {
  AppRegistry,
  StyleSheet,
  Text,
  View,
  Text,
} from 'react-native';
```

例子：做一个简单版的网易新闻web单页

![WX20170501-114928@2x](http://p44bkxib3.bkt.clouddn.com/WX20170501-114928@2x.png)

界面分析可以分成两部分：Header和新闻内容，整个界面是一个组件，由两个子组件组成。如果都写在index.ios.js文件中，代码比较乱，在单独一个文件中定义子组件，使用Module.exports将组件导出为独立的模块，可以在其他文件中引用。

header.js

```JavaScript
// 组件
var Header = React.createClass({
  render: function () {
    return (
      <View style={styles.flex}>
        <Text style={styles.font}>
          <Text style={styles.font_1}>网易</Text>
          <Text style={styles.font_2}>新闻</Text>
          <Text>有态度</Text>
        </Text>
      </View>
    );
  }
});

// 样式
var styles = StyleSheet.create({
  flex: {
    marginTop: 25,
    height: 40,
    borderBottomWidth: 1,
    borderBottomColor: '#EF2D26',
    alignItems: 'center',
  },
  // 字体公共部分
  font: {
    fontSize: 25,
    fontWeight: 'bold',
    textAlign: 'center',
  },
  font_1: {
    color: '#CD1D1C',
  },
  font_2: {
    color: '#FFF',
    backgroundColor: '#CD1D1C',
  }
});

// 导出模块
module.exports = Header;
```

news.js

```JavaScript
// 组件
var News = React.createClass({
  show: function(title) {
    alert(title);
  },
  render: function () {
    // 定义数组，用于存储设置好的Text组件
    var newsComponents = [];
    // 遍历存储信息的数组，从外部传入的
    for (var i in this.props.news) {
      var text = (
        <Text
          onPress={this.show.bind(this, this.props.news[i])}
          style={styles.news_item}
          numberOfLines={2}
          key={i}>
          {this.props.news[i]}
        </Text>
      );
      // 将设置好的Text存入数组
      newsComponents.push(text);
    }

    return (
      <View style={styles.flex}>
        <Text style={styles.news_title}>今日要闻</Text>
        {newsComponents}
      </View>
    );
  }
});

// 样式
var styles = StyleSheet.create({
  flex: {
    flex: 1,
  },
  // ‘今日要闻’标题
  news_title: {
    fontSize: 20,
    fontWeight: 'bold',
    color: '#CD1D1C',
    marginLeft: 10,
    marginTop: 15,
  },
  // 每条新闻
  news_item: {
    marginTop: 10,
    marginLeft: 10,
    marginRight: 10,
    fontSize: 15,
    lineHeight: 30,
  }
});

// 导出模块
module.exports = News;
```

index.ios.js

```JavaScript
// 引入
var Header = require('./header');
var News = require('./news');

export default class LessonText extends Component {
  render() {

    // 定义数组存储新闻内容
    var news = [
      '1.回复看到撒后方可',
      '2.发生的路口附近',
      '3.附件是到付',
      '4.房间号大搜罗咖啡就撒打开开发和四大，发神经的后来开飞机上大力开发接收到放假撒的路口附近',
    ];

    return (
      <View style={styles.flex}>
        {/* Header */}
        <Header></Header>
        {/* News */}
        <News news={news}></News>
      </View>
    );
  }
}

const styles = StyleSheet.create({
  flex: {
    flex: 1,
  },
});
```

上面的代码通过 `module.exports` 导出模块，主入口通过 `require('./header');` 引入模块，达到了模块分离，精简了代码。

## Touchable

ReactNative 中提供3个组件用于给其他没有触摸事件的组件绑定触摸事件（都需要导入组件），[官方文档](https://facebook.github.io/react-native/docs/touchableopacity.html)：

```
import {
  AppRegistry,
  StyleSheet,
  Text,
  View,
  TouchableOpacity,
  TouchableHighlight,
  TouchableWithoutFeedback,
} from 'react-native';
```


* TouchableOpacity 透明触摸，点击时，组件会出现透明过度效果
* TouchableHighlight 高亮触摸，点击时，组件会出现高亮效果
* TouchableWithoutFeedback 无反馈性触摸，点击时，组件无视觉变化

例子：

```JavaScript
var LessonTouchable = React.createClass({
  clickBtn: function () {
    alert('按钮被点击');
  },
  render: function () {
    return (
      <View style={styles.container}>
        <View style={styles.flex}>
          <View style={styles.input}></View>
        </View>
        <TouchableHighlight style={styles.btn} onPress={this.clickBtn}>
          <Text style={styles.search}>搜索</Text>
        </TouchableHighlight>
      </View>
    );
  }
});

var styles = StyleSheet.create({
  container: {
    flexDirection: 'row',
    height: 45,
    marginTop: 25,
  },
  flex: {
    flex: 1,
  },
  input: {
    height: 45,
    borderWidth: 1,
    marginLeft: 5,
    paddingLeft: 5,
    borderColor: '#CCC',
    borderRadius: 4,
  },
  btn: {
    width: 55,
    marginLeft: 5,
    marginRight: 5,
    backgroundColor: '#23BEFF',
    height: 45,
    justifyContent: 'center',
    alignItems: 'center',
  },
  search: {
    color: '#FFF',
    fontSize: 15,
    fontWeight: 'bold',
  }
});
```

上面只提供了 `TouchableHighlight` 的示例，另外2个自己尝试效果。

## TextInput

TextInput是一个允许用户在应用中通过键盘输入文本的基本组件，本组件的属性提供了多种特性的配置，譬如自动完成、自动大小写、占位文字，以及多种不同的键盘类型（如纯数字键盘）等等，[官方文档](https://facebook.github.io/react-native/docs/textinput.html)，常用：
    
```
placeholder 占位符
value 输入框的值
password 是否密文输入
editable 输入框是否可编辑
returnKeyType 键盘return键类型
onChange 当文本变化时调用
onEndEditing 当结束编辑时调用
onSubmitEditing 当结束编辑，点击提交按钮时调用
```

例子：

```JavaScript
var LessonTextInput = React.createClass({
  getInitialState: function() {
    return {
      inputText: '',
    };
  },
  // 输入框的onChange实现
  getContent: function(text) {
    this.setState({
      inputText: text
    });
  },
  clickBtn: function() {
    alert(this.state.inputText);
  },
  render: function() {
    return (
      <View style={styles.container}>
        <View style={styles.flex}>
          <TextInput
            style={styles.input}
            returnKeyType='search'
            placeholder='请输入内容'
            onChangeText={this.getContent}
          />
        </View>
        <TouchableOpacity style={styles.btn} onPress={this.clickBtn}>
          <Text style={styles.search}>搜索</Text>
        </TouchableOpacity>
      </View>

    );
  }
});

var styles = StyleSheet.create({
  container: {
    flexDirection: 'row',
    height: 45,
    marginTop: 25,
  },
  flex: {
    flex: 1,
  },
  input: {
    height: 45,
    borderWidth: 1,
    marginLeft: 5,
    paddingLeft: 5,
    borderColor: '#CCC',
    borderRadius: 4,
  },
  btn: {
    width: 55,
    marginLeft: 5,
    marginRight: 5,
    backgroundColor: '#23BEFF',
    height: 45,
    justifyContent: 'center',
    alignItems: 'center',
  },
  search: {
    color: '#FFF',
    fontSize: 15,
    fontWeight: 'bold',
  }
});
```

## Image
Image 用于显示图片的组件，包括网络图片、静态资源等等，[官方文档](https://facebook.github.io/react-native/docs/image.html)，常用属性：

```
resizeMode  图片适应模式cover、contain、stretch
source 图片引用地址
iOS 支持的属性：onLoad、onLoadEnd、onLoadStart、onProgress
```

获取图片主要有两种方式：本地图片和网络图片，`Image` 都提供了获取的方式。

* 获取本地图片，现在工程目录下创建图片文件夹，和 `index.ios.js` 同一级目录，然后通过 **source={require}** 属性获取
* 获取网络图片，通过 **source={uri}** 获取


例子：

```JavaScript
export default class LessonImage extends Component {
  render() {
    return (
      <View style={styles.container}>
        <View style={styles.net}>
          <Image
            style={styles.netImage}
            source={{uri:'http://reactnative.cn/static/docs/0.41/img/react-native-congratulations.png'}}
            />
        </View>
        <View style={styles.local}>
          <Image
            style={styles.localImage}
            source={require('./img/react-native-congratulations.png')}
            />
        </View>
      </View>
    );
  }
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    margin: 25,
    alignItems: 'center',
  },
  net: {
    marginTop: 38,
    width: 300,
    height: 240,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: 'cyan',
  },
  netImage: {
    width: 280,
    height: 200,
    backgroundColor: 'gray',
  },
  local: {
    marginTop: 30,
    width: 300,
    height: 200,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: 'cyan',
  },
  localImage: {
    width: 180,
    height: 180,
    backgroundColor: 'gray',
  },
});
```

## 总结

本篇文章非常简单的介绍了 ReactNative 的几个组件，并给出了示例，更加详细的用法请参考官方文档。
    




