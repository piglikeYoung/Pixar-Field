
## 前言

本章继续学习 ReactNative 常用组件，ScrollView 和 ListView。有过移动开发经验的同学都知道 ScrollView 和 ListView 都是不可获取的组件，ListView 就是继承自 ScrollView。

还有学习非常重要的 Navigator组件。

## ScrollView

ScrollView 有很多回调方法，比如开始拖拽、结束拖拽、开始滑动、结束滑动等等，下面这个例子就是实现相关的回调，看看 ScrollView 的简单实现。

例子1：ScrollView 的简单实现，实现监测拖拽、滑动的相关方法

```JavaScript
var MyScrollView = React.createClass({
  _onScrollBeginDrag: function() {
    console.log('开始拖拽');
  },
  _onScrollEndDrag: function() {
    console.log('结束拖拽');
  },
  _onMomentumScrollBegin: function() {
    console.log('开始滑动');
  },
  _onMomentumScrollEnd: function() {
    console.log('结束滑动');
  },
  _onRefresh: function() {
    console.log('刷新');
  },
  render: function() {
    return (
      <View style={styles.container}>
        <ScrollView
          style={styles.scrollView}
          showsVerticalScrollIndicator={true}
          onScrollBeginDrag={this._onScrollBeginDrag}
          onScrollEndDrag={this._onScrollEndDrag}
          onMomentumScrollBegin={this._onMomentumScrollBegin}
          onMomentumScrollEnd={this._onMomentumScrollEnd}
          refreshControl={
            <RefreshControl
              refreshing={false}
              tintColor='red'
              title='正在刷新......'
              onRefresh={this._onRefresh}
            />
          }>
          <View style={styles.view_1}></View>
          <View style={styles.view_2}></View>
          <View style={styles.view_3}></View>
        </ScrollView>
      </View>
    );
  }
});

var styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: 'cyan',
  },
  scrollView: {
    marginTop: 25,
    backgroundColor: '#CCCCCC',
  },
  view_1: {
    margin: 15,
    flex: 1,
    height: 300,
    backgroundColor: 'yellow',
  },
  view_2: {
    margin: 15,
    flex: 1,
    height: 300,
    backgroundColor: 'blue',
  },
  view_3: {
    margin: 15,
    flex: 1,
    height: 300,
    backgroundColor: 'green',
  },
});

module.exports = MyScrollView;
```

下面这个例子是通过 ScrollView 实现电影列表，电影的数据是 RN 提供的[数据](// https://raw.githubusercontent.com/facebook/react-native/master/docs/MoviesExample.json)，因为还没有学习如何获取网络数据，我把数据拷贝下来，生成了个JSON文件，需要的时候本地加载。

例子2：
```JavaScript
// 从文件中读取数据。执行JSON.Parse()方法，将JSON格式的字符串转换为JSON格式对象
var movieData = require('./data.json');

// 获取所有movies数据
var movies = movieData.movies;

var MovieList = React.createClass({
  render: function() {
    // 创建电影列表组件，根据movie中元素的个数，创建对应的组件
    // 遍历数组，每次创建一个movie对象
    // 定义一个空数组，存储电影组件
    var movieRows = [];
    // 遍历数组
    for (var i in movies) {
      // 获取movie对象
      var movie = movies[i];
      // 创建组件，显示电影信息：
      // 图像(movie.posters.thumbnail)
      // 电影名称(movie.title)
      // 上映时间(movie.year)
      var row = (
        <View key={i} style={styles.row}>
          <Image
            style={styles.thumbnail}
            source={{uri:movie.posters.thumbnail}}/>
          <View style={styles.rightContainer}>
            <Text style={styles.title}>{movie.title}</Text>
            <Text style={styles.year}>{movie.year}</Text>
          </View>
        </View>
      );
      movieRows.push(row);
    }

    return (
      <View style={styles.container}>
        <ScrollView style={styles.scrollView}>
          {
            // 数组(所有子组件)
            movieRows
          }
        </ScrollView>
      </View>
    );
  }
});

var styles = StyleSheet.create({
  container: {
    flex: 1,
  },
  scrollView: {
    flex: 1,
    marginTop: 25,
    backgroundColor: '#F5FCFF'
  },
  row: {
    flexDirection: 'row',
    padding: 5,
    alignItems: 'center',
    backgroundColor: '#F5FCFF',
  },
  thumbnail: {
    width: 53,
    height: 81,
    backgroundColor: 'gray',
  },
  rightContainer: {
    marginLeft: 10,
    flex: 1,
  },
  title: {
    fontSize: 18,
    marginTop: 3,
    marginBottom: 3,
    textAlign: 'center',
  },
  year: {
    marginBottom: 3,
    textAlign: 'center',
  }
});

module.exports = MovieList;
```

## ListView

ListView 和 ScrollView 一样都可以滚动视图显示内容，但是它有一些高级特性：譬如给每段/组(section)数据添加一个带有粘性的头部（类似iPhone的通讯录，其首字母会在滑动过程中吸附在屏幕上方）；在列表头部和尾部增加单独的内容；在到达列表尾部的时候调用回调函数(onEndReached)，还有在视野内可见的数据变化时调用回调函数(onChangeVisibleRows)，以及一些性能方面的优化。

特别是加载大数据时的性能优化，只需渲染显示在屏幕上的内容即可，超出屏幕的内容可以不渲染。
[官方文档](https://facebook.github.io/react-native/docs/listview.html)

例子1：简单的ListView列表

```JavaScript
var contents = [
  '312312',
  'fsdfsad',
  '312312444',
  '4324231421',
  '423142134213',
];

var MyListView = React.createClass({
  getInitialState: function() {
    // 创建dataSource对象
    var ds = new ListView.DataSource({
      // 通过判断决定渲染哪些行组件，避免全部渲染，提高性能
      rowHasChanged: (oldRow, newRow) => oldRow!==newRow
    });

    return {
      // 设置dataSource时，不直接使用提供的原始数据，使用cloneWithRows对数据源进行复制
      // 使用复制后的数据源实例化ListView。
      // 优点：当原始数据源发生变化时，ListView组件的dataSource不会改变
      dataSource: ds.cloneWithRows(contents)
    };
  },
  // 渲染行组件，参数是每行要显示的数据对象
  _renderRow: function(rowData:string) {
    return (
      <View style={styles.row}>
        <Text style={styles.content}>{rowData}</Text>
      </View>
    );
  },
  render: function() {
    return (
      <ListView
        style={styles.container}
        dataSource={this.state.dataSource}
        renderRow={this._renderRow}/>
    );
  }
});

var styles = StyleSheet.create({
  container: {
    flex: 1,
    marginTop: 25,
  },
  row: {
    justifyContent: 'center',
    alignItems: 'center',
    padding: 5,
    height: 100,
    borderBottomWidth: 1,
    borderColor: '#CCCCCC',
  },
  content: {
    flex: 1,
    fontSize: 20,
    color: 'blue',
    lineHeight: 100,
  }
});

module.exports = MyListView;
```

例子2：使用 ListView 改写 ScrollView 的电影列表

```JavaScript
// 从文件中读取数据
var movieData = require('./data.json');

// 获取所有movies数据，属性movies是个数组
// 原始数据
var movies = movieData.movies;

var MovieList = React.createClass({
  getInitialState: function() {
    var ds = new ListView.DataSource({
      rowHasChanged: (oldRow, newRow) => oldRow!==newRow
    });

    return {
      dataSource: ds.cloneWithRows(movies)
    };
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
  render: function () {
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
  }
});

var styles = StyleSheet.create({
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

