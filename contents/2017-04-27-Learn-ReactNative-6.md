
## 前言

本篇文章将学习 Navigator组件、TabBarIOS组件和fetch网络请求。

## Navigator 组件

[官方文档](https://facebook.github.io/react-native/docs/navigator.html)

app 由很多公共视图组成，一个应用中重要的功能之一是“导航”，在ReactNative中成为“路由route”，使用导航器可以让你在应用的不同场景（页面）间进行切换。

在ReactNative中，有两个实现导航功能的组件：`Navigator` 和 `NavigatorIOS`

* Navigator支持安卓和iOS，NavigatorIOS支持iOS
* NavigatorIOS比Navigator具有更多的属性和方法，在UI方面可以进行更多的设置，例如：backButtonIcon、backButtonTitle、onLeftButtonPress等等，比较方便，如果想实现更多的自定义设置，建议使用Navigator

### 导航功能

导航器通过路由对象(route)来分辨不同的场景。每个路由对象都对应一个页面组件，开发者设置什么，导航器显示什么，所有route是导航器中重要的一个对象。

三部操作实现导航功能：

1. 设置路由对象（告诉导航器我要现实哪个页面），创建路由对象，对象的内容自定义，但是必须包含该场景需要展示的页面组件
2. 场景渲染配置（告诉导航器我要什么样的页面跳转效果）
3. 渲染场景（告诉导航器如何渲染页面）
    利用第一步设置的路由对象进行渲染场景
    
    
```JavaScript
// 定义一个页面
// FirstPage 一个button，点击进入下一级页面
var FirstPage = React.createClass({
  // 按钮onPress事件处理方法
  pressPush: function() {
    // 推出下一级页面
    var nextRoute = {
      component: SecondPage
    };
    this.props.navigator.push(nextRoute);
  },
  render: function() {
    return (
      <View style={[styles.flex, {backgroundColor:'green'}]}>
        <TouchableOpacity style={styles.btn} onPress={this.pressPush}>
          <Text>点击推出下一级页面</Text>
        </TouchableOpacity>
      </View>
    );
  }
});

// 定义第二个页面
// SecondPage 一个button，点击返回上一级页面

var SecondPage = React.createClass({
  // 按钮onPress事件处理方法
  pressPop: function() {
    // 返回上一级页面
    this.props.navigator.pop();
  },
  render: function() {
    return (
      <View style={[styles.flex, {backgroundColor:'pink'}]}>
        <TouchableOpacity style={styles.btn} onPress={this.pressPop}>
          <Text>点击返回上一级页面</Text>
        </TouchableOpacity>
      </View>
    );
  }
});

var LessonNavigator = React.createClass({
  render: function() {
    var rootRoute = {
      component: FirstPage
    };
    return(
      <Navigator
        /*
          第一步：
          initialRoute

          这个指定了默认的页面，也就是启动APP之后会看到界面的第一屏
          对象的属性是自定义的，这个对象中的内容会在renderScene方法中处理

          备注：必须包含的属性，即component，表示需要渲染的页面组件
        */
        initialRoute={rootRoute}
        /*
          第二步：
          configureScene

          场景渲染的配置
        */
        configureScene={(route) => {
          return Navigator.SceneConfigs.PushFromRight;
        }}
        /*
          第三步：
          renderScene

          渲染场景

          参数：route (第一步创建并设置给导航器的路由对象)，navigator（导航器对象）
          实现：给需要显示的组件设置属性
        */
        renderScene={(route, navigator) => {
          // 从route对象中获取页面组件
          var Component = route.component;
          return (
            <Component
              navigator={navigator}
              route={route}
            />
          );
        }}
      />
    );
  }
});

var styles = StyleSheet.create({
  flex: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
  },
  btn: {
    width: 150,
    height: 30,
    backgroundColor: '#0089FF',
    borderWidth: 1,
    borderRadius: 3,
    justifyContent: 'center',
    alignItems: 'center',
  }
});

module.exports = LessonNavigator;
```

### 传值功能

将第一个界面输入的值，传递到第二个界面显示。

```JavaScript
// 有输入的页面
var InputPage = React.createClass({
  getInitialState: function() {
    return {
      // 记录输入的值
      content: ''
    };
  },
  getInputContent: function(inputText) {
    // 将输入框的值进行记录
    this.setState({
      content: inputText
    });
  },
  pushNextPage: function() {
    //进入下一个界面并传值
    var route = {
      component: DetailsPage,
      passProps: {
        showText: this.state.content
      }
    };
    this.props.navigator.push(route);
  },
  render: function() {
    return (
      <View style={inputStyle.container}>
        <TextInput
          style={inputStyle.input}
          placeholder='请输入内容'
          onChangeText={this.getInputContent}
        />
        <TouchableOpacity style={inputStyle.btn} onPress={this.pushNextPage}>
          <Text>进入下一页</Text>
        </TouchableOpacity>
      </View>
    );
  }
});

var inputStyle = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: 'white',
  },
  input: {
    height: 45,
    marginLeft: 25,
    marginRight: 25,
    paddingLeft: 5,
    borderWidth: 1,
    borderColor: 'black',
    borderRadius: 4,
  },
  btn: {
    marginTop: 20,
    height: 30,
    borderWidth: 1,
    borderRadius: 4,
    borderColor: 'black',
    padding: 5,
    justifyContent: 'center',
    alignItems: 'center',
  }
});

// 显示输入内容的页面

var DetailsPage = React.createClass({
  popFrontPage: function() {
    // 返回上一级
    this.props.navigator.pop();
  },
  render: function() {
    return (
      <View style={detailStyle.container}>
        <Text style={detailStyle.text}>{this.props.showText}</Text>
        <TouchableOpacity style={detailStyle.btn} onPress={this.popFrontPage}>
          <Text>返回上一页</Text>
        </TouchableOpacity>
      </View>
    );
  }
});

var detailStyle = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: 'white',
  },
  text: {
    marginLeft: 25,
    marginRight: 25,
    padding: 25,
    backgroundColor: 'cyan',
    fontSize: 20,
    textAlign: 'center',
  },
  btn: {
    marginTop: 20,
    height: 30,
    borderWidth: 1,
    borderRadius: 4,
    borderColor: 'black',
    padding: 5,
    justifyContent: 'center',
    alignItems: 'center',
  }
});

var LessonNavigator = React.createClass({
  render: function() {
    var rootRoute = {
      component: InputPage,
      // 存储需要传递的内容
      passProps: {

      }
    };
    return (
      <View style={{flex:1}}>
        <Navigator
          initialRoute={rootRoute}
          configureScene={(route) => {
            return Navigator.SceneConfigs.PushFromRight;
          }}
          renderScene={(route, navigator) => {
            var Component = route.component;
            return (
              <Component
                navigator={navigator}
                route={route}
                {...route.passProps}
              />
            );
          }}
        />
      </View>
    );
  }
});

var styles = StyleSheet.create({

});

module.exports = LessonNavigator;
```

## TabBarIOS

[官方文档](https://facebook.github.io/react-native/docs/tabbarios.html)

在ReactNative中，实现页面切换，提供了两个组件：TabBarIOS和TabBarIOS.Item，常用功能：

* selected：是否选中某个Tab。如果为true则选中并显示组件
* title： 标题
* barTintColor：Tab栏的背景颜色
* icon：图标
* onPress：点击事件，当某个tab被选中时，需要改变组件的select={true}

实现原理：点击tab时触发onPress方法，记录被点击tab的title。再通过title设置tab是否被选中（通过比较设置selected的值，true/false）

例子：先创建三个页面，引入模块，在 `index.ios.js` 中统一装配


```JavaScript
var LessonTextInput = require('./testInput');
var LessonImage = require('./loadImage');
var MovieList = require('./movieList');

var LessonTabBarIOS = React.createClass({
  getInitialState: function() {
    return {
      // 记录点击tab的title
      tab: 'LessonTextInput'
    };
  },
  // TabBarIOS.Item 的onPress触发方法
  select: function(tabName) {
    this.setState({
      tab: tabName
    });
  },
  render: function() {
    return (
      <TabBarIOS style={{flex:1}}>
        <TabBarIOS.Item
          title='LessonTextInput'
          icon={require('./img/icon_chat_normal@3x.png')}
          onPress={this.select.bind(this, 'LessonTextInput')}
          selected={this.state.tab==='LessonTextInput'}
          >
            <LessonTextInput></LessonTextInput>
        </TabBarIOS.Item>
        <TabBarIOS.Item
          title='LessonImage'
          systemIcon='bookmarks'
          onPress={this.select.bind(this, 'LessonImage')}
          selected={this.state.tab==='LessonImage'}
          >
            <LessonImage></LessonImage>
        </TabBarIOS.Item>
        <TabBarIOS.Item
          title='MovieList'
          icon={require('./img/icon_discover_selected@3x.png')}
          onPress={this.select.bind(this, 'MovieList')}
          selected={this.state.tab==='MovieList'}
          >
            <MovieList></MovieList>
        </TabBarIOS.Item>
      </TabBarIOS>
    );
  }
});
```

## fetch网络请求

在ReactNative中，使用fetch实现了网络请求。fetch和XMLHttpRequest非常类似，是一个封装程度更高的网络API，使用起来很简洁，因为使用了Promise。

Promise是异步编程的一种解决方案，比传统的解决方案（回调函数和事件），更加合理和强大，ES6将其写进了语言标准，统一了用法，原生提供了Promise对象。简单点说就是一个容器，里面保存着某个未来才会结束的事件（通常是一个异步操作）的结果。

Promise对象代表一个异步操作，有三种状态：Pending（进行中）、Resolved（已完成）、Rejected（已失败）。

Promise对象生成后，可以分别指定“完成”和“失败”状态的回调函数，链式调用方法。

语法：

```JavaScript
fetch(参数)
  .then(完成的回调函数)
  .catch(失败的回调函数)

  // opts 网络请求的配置
  fetch(url, opts)
  .then((response) => {
    // 网络请求成功执行该回调函数，得到相应对象，通过response可以获取请求的数据
    // 例如：json、text等等

    return response.text();
    // return response.json();
  })
  .then((responseData) = > {
    // 处理请求得到的数据

  })
  .catch((error) => {
    // 网络请求失败执行该回调函数，得到错误信息

  })
```

### GET请求


```JavaScript
function getRequest(url) {
  var opts = {
    method: 'GET'
  }

  fetch(url, opts)
  .then((response) => {
    return response.text();
  })
  .then((responseText) => {
    alert(responseText);
  })
  .catch((error) => {
    alert(error);
  })
}
```

### POST请求

POST请求需要请求参数，配置请求参数需要先认识一个类 `FromData`。

Web应用中频繁使用的一项功能就是表单数据的序列化，XMLHttpRequest的2级定义了FormData类型，FromData主要用于实现序列化表单已经创建与表单格式相同的数据：

```JavaScript
var data = new FormData();
data.append('name', 'yjh');
```

append方法接收两个参数：键和值，分别对应表单字段的名字和值，可添加多个键值对。

> 在jquery中，‘key1=value1&key2=value2’作为参数传入对象框架会自动封装成FormData形式
在Fetch中进行的post请求时，需要自动创建FormData对象传给body


```JavaScript
function postRequest(url) {
  let formData = new FormData();
  formData.append('username', 'yjh');
  formData.append('password', '123456');

  var opts = {
    method: 'POST',
    body: formData
  }

  fetch(url, opts)
  .then((response) => {
    return response.text();
  })
  .then((responseText) => {
    alert(responseText);
  })
  .catch((error) => {
    alert(error);
  })
}

var GetData = React.createClass({
  render: function() {
    return (
      <View style={styles.container}>
        <TouchableOpacity onPress={getRequest.bind(this, 'https://httpbin.org/get?userName=yjh')}>
          <View style={styles.btn}>
            <Text>GET</Text>
          </View>
        </TouchableOpacity>
        <TouchableOpacity>
          <View style={styles.btn} onPress={postRequest.bind(this, 'https://httpbin.org/post')}>
            <Text>POST</Text>
          </View>
        </TouchableOpacity>
      </View>
    );
  }
});

var styles = StyleSheet.create({
  container: {
    flex: 1,
    marginTop: 30,
    backgroundColor: 'cyan',
    flexDirection: 'row',
    justifyContent: 'space-around',
    alignItems: 'center',
  },
  btn: {
    width: 60,
    height: 30,
    borderWidth: 1,
    borderRadius: 3,
    borderColor: 'black',
    backgroundColor: 'yellow',
    justifyContent: 'center',
    alignItems: 'center',
  }
});

module.exports = GetData;
```



### 网络请求获取电影列表数据


```JavaScript
/*
  请求电影列表

  未获得数据时，显示等待页面；获取数据后，显示电影列表页面
  需要给state添加一个属性，用于记录下载状态
*/

var REQUEST_URL = 'https://raw.githubusercontent.com/facebook/react-native/master/docs/MoviesExample.json';

var MovieList = React.createClass({
  getInitialState: function() {
    var ds = new ListView.DataSource({
      rowHasChanged: (oldRow, newRow) => oldRow!==newRow
    });
    return {
      loaded: false, // 数据是否请求完成
      dataSource: ds,
    };
  },
  getData: function() {
    fetch(REQUEST_URL)
    .then((response) => {
      return response.json();
    })
    .then((responseData) => {
      // 刷新组件，展示ListView
      // 更新dataSource
      this.setState({
        loaded: true,
        dataSource: this.state.dataSource.cloneWithRows(responseData.movies),
      });
    })
    .catch((error) => {
      alert(error);
    })
  },
  render: function() {
    // 判断请求是否完成
    if (!this.state.loaded) {
      return this.renderLoadingView();
    }

    // 电影列表
    return (
      <ListView
        style={styles.listView}
        dataSource={this.state.dataSource}
        renderRow={this._renderRow}
        renderHeader={this._renderHeader}
        renderSeparator={this._renderSeparator}
        // 一开始渲染的行数
        initialListSize={10}
      />
    );
  },
  // 组件挂载完成
  componentDidMount: function() {
    // 组件挂载后，开始下载数据
    this.getData();
  },
  // 等待请求页面
  renderLoadingView: function() {
    return (
      <View style={styles.loadingContainer}>
        <Text style={styles.loadingText}>Loading movie.....</Text>
      </View>
    );
  },
  // 渲染行组件
  _renderRow: function(movie) {
    return (
      <View style={styles.row}>
        <Image
          style={styles.thumbnail}
          source={{uri:movie.posters.thumbnail}}/>
        <View style={styles.rightContainer}>
          <Text style={styles.title}>{movie.title}</Text>
          <Text style={styles.year}>{movie.year}</Text>
        </View>
      </View>
    );
  },
  // 渲染头部
  _renderHeader: function() {
    return (
      <View style={styles.header}>
        <Text style={styles.header_text}>Movies List</Text>
        <View style={styles.separator}></View>
      </View>
    );
  },
  // 渲染分隔线
  _renderSeparator: function(sectionID:number, rowID:number) {
    return (
      <View style={styles.separator} key={sectionID+rowID}></View>
    );
  },
});


var styles = StyleSheet.create({
  // Loading样式
  loadingContainer: {
    flex: 1,
    marginTop: 25,
    backgroundColor: 'cyan',
    justifyContent: 'center',
    alignItems: 'center',
  },
  loadingText: {
    fontSize: 30,
    fontWeight: 'bold',
    textAlign: 'center',
    marginLeft: 10,
    marginRight: 10,
  },
  // listView
  listView: {
    marginTop: 25,
    flex: 1,
    backgroundColor: '#F5FCFF',
  },
  row: {
    flexDirection: 'row',
    padding: 5,
    alignItems: 'center',
    backgroundColor: '#F5FCFF'
  },
  thumbnail: {
    width: 53,
    height: 81,
    backgroundColor: 'gray'
  },
  rightContainer: {
    marginLeft: 10,
    flex: 1,
  },
  title: {
    fontSize: 10,
    marginTop: 3,
    marginBottom: 3,
    textAlign: 'center'
  },
  year: {
    marginBottom: 3,
    textAlign: 'center',
  },
  // header
  header: {
    height: 44,
    backgroundColor: '#F5FCFF',
  },
  header_text: {
    flex: 1,
    fontSize: 20,
    fontWeight: 'bold',
    textAlign: 'center',
    lineHeight: 44,
  },
  // 分隔线
  separator: {
    height: 1,
    backgroundColor: '#CCCCCC',
  },
});

module.exports = MovieList;
```

## 总结

截止至本章，学习 ReactNative 基础系列文章，已经把 ReactNative 基础的内容都已经过了个遍，通过这几篇文章的内容学习，相信你应该对 ReactNative 已经有个大致的了解，并且能够上手完成一个小应用了，如果你还想深入了解 ReactNative 的高级特性，可以去官网上查阅文档。

