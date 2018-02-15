
## 前言
上一篇文章学习了React的基础用法，这篇文章开始真正学习ReactNative的知识。

## 组件生命周期

在编程开发中，每个创建的对象都会有生命周期，对象创建后会占用内存，创建的越多可用内存越少，当对象的生命周期结束时，GC就会回收内存，所以了解生命周期对开发非常重要。

组件生命周期有三个状态：
1. Mounting：组件挂载，已插入真实 DOM
2. Updating：组件更新，正在被重新渲染
3. Unmounting：组件移除，已移出真实 DOM

组件生命周期四个阶段：
1. 创建
2. 实例化
3. 更新
4. 销毁

### Mounting/组件挂载相关 

* componentWillMount：组件将要挂载。在render之前执行，但仅执行一次，即使多次重复渲染该组件，或者改变了组件的state
* componentDidMount：组件以及挂载。在render之后执行，同一个组件重复渲染只执行一次

### Updating/组件更新相关

* componentWillReceiveProps(object nextProps)：已加载组件收到新的props之前调用，注意组件初始化渲染时则不会执行
* shouldComponentUpdate(object nextProps, object nextState)：组件判断是否重新渲染时调用。当组件收到新的props或者新的state时，就会调用，根据返回值true/false决定是否重新渲染
* componentWillUpdate(object nextProps, object nextState)：组件将要更新
* componentDidUpdate(object prevProps, object prevState)：组件已经更新

### Unmounting/组件移除相关

* componentWillUnmount：在组件要被移除之前的时间点触发，可以利用它来处理清理组件的一些工作

### 生命周期中与props和state相关

* getDefaultProps 设置props属性默认值
* getInitialState 升值state属性初始值

### 例子


```JavaScript
	var Demo = React.createClass({
       /*
       *  一、创建阶段
       *  流程：
       *      只调用getDefaultProps方法
       * */ 
       getDefaultProps: function () {

           // 在创建类的时候被调用，设置this。props的默认值
           console.log('getDefaultProps');
           return {};
       },
       /*
       *   二、实例化阶段
       *   流程：
       *       getInitialState
       *       componentWillMount
       *       render
       *       componentDidMount
       * 
       * */
       getInitialState: function () {
           // 设置this。state的默认值
           console.log('getInitialState');
           return null;
       },
       componentWillMount: function () {
           // 在render之前调用
           console.log('componentWillMount');
       },
       render: function () {
           // 渲染并返回一个虚拟DOM
           console.log('render');
           return <div>Hello React</div>
       },
       componentDidMount: function () {
           // 在render之后调用
           // 在该方法中，React会使用render方法返回的虚拟DOM对象创建真实的DOM结构
           // 可以在这个方法中读取DOM节点
           console.log('componentDidMount');
       },

       /*
       *   三、更新阶段
       *   流程：
       *   componentWillReceiveProps
       *   shouldComponentUpdate 如果返回值是false，之后三个方法不执行
       *   componentWillUpdate
       *   render
       *   componentDidUpdate
       *
       * */
       componentWillReceiveProps: function () {
           console.log('componentWillReceiveProps');
       },
       shouldComponentUpdate: function () {
           // 是否需要更新
           console.log('shouldComponentUpdate');
           return true;
       },
       componentWillUpdate: function () {
           console.log('componentWillUpdate');
       },
       componentDidUpdate: function () {
           console.log('componentDidUpdate');
       },

       /*
       *   四、销毁阶段
       *   流程：
       *       componentWillUnmount
       *
       * */

       componentWillUnmount: function () {
           console.log('componentWillUnmount');
       }
       
   });

   // 第一次创建并加载组件
   ReactDOM.render(
       <Demo />,
       document.getElementById('container')
   );

   // 重新渲染组件
   ReactDOM.render(
           <Demo />,
       document.getElementById('container')
   );

   // 移除组件
   ReactDOM.unmountComponentAtNode(document.getElementById('container'));
```

## StyleSheet

ReactNative 中定义组件样式，是通过 `StyleSheet` 样式表定义的，有几个需要注意的地方：
1. HTML5以;结尾，React以,结尾
2. HTML5中的key、value都不加引号， React中属于JavaScript对象，key的名字不能出现‘-’，需要使用驼峰命名法，如果value是字符串要加引号
3. HTML5中，value如果是数字，需要带单位，React中不需要单位

盒子模型

![box-model](http://p44bkxib3.bkt.clouddn.com/box-model.gif)

例子：

```
const styles = StyleSheet.create({
  container: {
    backgroundColor: 'red',
    width: 300,
    height: 400,
    marginTop: 25,
    marginLeft: 30,
  },
  top: {
    backgroundColor: 'green',
    width: 280,
    height: 250,
    margin: 10,
  },
  bottom: {
    backgroundColor: 'yellow',
    width: 280,
    height: 110,
    margin: 10,
  },
  boder: {
    borderWidth: 3,
    borderColor: 'black',
  },
});
```

## CSS3 弹性盒子(Flex Box)

![flexBox-model](http://p44bkxib3.bkt.clouddn.com/flexBox-model.jpg)

弹性盒模型，主要有两个轴，主轴和交叉轴，主轴并不一定是横向的，是根据Flex Item元素的排列方向来的，如果Item是横向排列，则主轴是横向的，如果Item是纵向的，主轴就是纵向的。

弹性盒子有很多属性设置，可以参考[《CSS3 弹性盒子》](http://www.runoob.com/css3/css3-flexbox.html)设置。

例子：

```JavaScript
export default class LessonFlex extends Component {
  render() {
    return (
      <View style={styles.container}>
        <View style={styles.child1}></View>
        <View style={styles.child2}></View>
      </View>
    );
  }
}

const styles = StyleSheet.create({
  container: {
    margin: 30,
    width: 300,
    height: 300,
    backgroundColor: 'yellow',
    // 默认主轴方向是column（竖向）
    // 设置为横向排列
    flexDirection: 'row',
    // 主轴方向
    justifyContent: 'center',
    // 交叉轴
    alignItems: 'center',
  },
  child1: {
    width: 100,
    height: 100,
    backgroundColor: 'green',
  },
  child2: {
    width: 100,
    height: 100,
    backgroundColor: 'blue',
  }
});
```

### flex属性

Flex Box里面一个重要的属性， `flex` 属性，可以给组件指定flex，flex的值是数字。
* flex:1 表示组件可以撑满父组件所有的剩余空间
* 同时存在多个并列的子组件，flex:1，均分
* 如果这些并列的子组件的flex值不一样，谁的值越大，谁占的剩余空间比例就越大(即占据剩余空间的比等于并列组件间flex值的比)

例子：

```JavaScript
const styles = StyleSheet.create({
  container: {
    marginBottom: 30,
    flex: 1,
    backgroundColor: 'cyan',
  },
  child1: {
    flex: 2,
    backgroundColor: 'green',
  },
  child2: {
    flex: 1,
    backgroundColor: 'yellow',
  }
})
```

## View组件

现在我们开始学习 ReactNative 的组件，先从[View组件](https://facebook.github.io/react-native/docs/view.html)开始。

在Web开发中，div是最重要的一个元素，是页面布局的基础。
在ReactNative开发中，View组件的作用类似于div。是最基础的组件，被看做是容器组件。

例子：

```JavaScript
export default class LessonView extends Component {
  render() {
    return (
      <View style={[styles.container, styles.flex]}>
        <View style={styles.item}>
          <View style={[styles.flex, styles.center]}>
            <Text>酒店</Text>
          </View>
          <View style={[styles.flex, styles.lineLeftRight]}>
            <View style={[styles.flex, styles.center, styles.lineCenter]}>
              <Text>海外酒店</Text>
            </View>
            <View style={[styles.flex, styles.center]}>
              <Text>特价酒店</Text>
            </View>
          </View>
          <View style={[styles.flex]}>
            <View style={[styles.flex, styles.center, styles.lineCenter]}>
              <Text>团购</Text>
            </View>
            <View style={[styles.flex, styles.center]}>
              <Text>民宿.客栈</Text>
            </View>
          </View>
        </View>
      </View>
    );
  }
}

const styles = StyleSheet.create({
  container: {
    marginTop: 64,
    backgroundColor: '#F2F2F2',
  },
  flex: {
    flex: 1
  },
  center: {
    justifyContent: 'center',
    alignItems: 'center',
  },
  item: {
    flexDirection: 'row',
    backgroundColor: '#FF607C',
    marginTop: 5,
    marginLeft: 5,
    marginRight: 5,
    height: 80,
    borderRadius: 5,
  },
  // 给中间的区域设置左右边线
  lineLeftRight: {
    borderLeftWidth: 1,
    borderRightWidth: 1,
    borderColor: 'white',
  },
  // 给上半区域设置下边线
  lineCenter: {
    borderColor: 'white',
    borderBottomWidth: 1,
  }
});
```

## 总结

本章学习了组件生命周期、了解了 StyleSheet 以及 Flex Box ，并且开始学习了第一个 ReactNative 组件 View。

