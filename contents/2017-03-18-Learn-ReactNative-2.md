
## 前言
上一篇文章介绍了 ReactNative 背景以及优缺点，分析了测试安装工程的工程结构，以及index.ios.js 的代码结构。
开始学习RN前，需要学习React的一些语法，本篇文章就带你学习些简单的React语法。

## React
有过Web前端开发基础的人都知道，写网页就是写HTML文件，所以先创建个HTML5的文件。引入以下js文件：

```JavaScript
<!-- react.js 是React的核心库 -->
<script src="https://unpkg.com/react@15/dist/react.js"  charset="utf-8"></script>
<!-- react-dom.js的作用是提供与DOM相关的功能 -->
<script src="https://unpkg.com/react-dom@15/dist/react-dom.js"  charset="utf-8"></script>
<!-- browser.js的作用是将JSX语法转换成JavaScript的语法 -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/babel-core/5.8.38/browser.js" charset="utf-8"></script>
```

并且在 body 标签中加入这行代码：

```
<div id="example"></div>
```

在React开发中，使用JSX，跟JavaScript不兼容，在使用JSX的地方，需要设置type:text/babel，babel是一个转换编译器，ES6转成可以在浏览器运行的代码

```JavaScript
<script type="text/babel">
</script>
```

然后，我们的React代码就可以在里面编写了，先介绍个React的最基本的方法，用于将模板转换成HTML语言：

```JavaScript
/*
ReactDOM.render()
React的最基本方法，用于将模板转换成HTML语言，渲染DOM

3个参数
第一个：模板的渲染内容（HTML形式）
第二个：这段模板需要插入的DOM节点
第三个：渲染后的回调，一般不用
*/
ReactDOM.render(
	<h1>Hello, world!</h1>,
	document.getElementById('example')
);
```
将HTML文件放到浏览器上运行，就能看到 Hello World 效果了。

### JSX
前面提到了 JSX ，它不是一门新语言，是个语法糖，有以下特点：
1. 必须借助React环境运行
2. JSX标签其实就是HTML标签，只不过我们不需要和JavaScript中一样，用“”包裹起来
3. 转换：JSX语法能够让我们更加直观的看到组件的DOM结构，不能直接在浏览器上运行，最终会转换成JavaScript代码

改写前一个例子：

```JavaScript
// 在JSX中运行JavaScript代码
// 使用{}括起来
var text = "你好";
ReactDOM.render(
	<h1>{text}</h1>,
	document.getElementById('example')
);
```

### 定义组件
创建组件有几个规则要遵守：
1. React中创建的组件类以大写字母开头，驼峰命名法
2. 在React中使用React.createClass方法创建一个组件类
3. 核心代码：每个组件类必须实现自己的render方法。输出定义好的组件模板。返回值：null、false、组件模板
4. 注意：组件类只能包含一个顶层标签

例子1：创建一个组件类，用于输出 Hello React

```JavaScript
var HelloMessage = React.createClass({
	render: function () {
		return <h1>Hello React</h1>;
	}
});

ReactDOM.render(
	<HelloMessage/>,
	document.getElementById('example')
);
```

例子2：实现组件从外部接收内容，并进行展示
```JavaScript
var HelloMessage = React.createClass({
	render: function () {
		/*
			this表示当前这个组件对象
			this.props.helloText 可以解释：当前组件对象的props对象中存储的helloText属性中的值
		*/
		return <h1>{this.props.helloText}</h1>;
	}
});

ReactDOM.render(
	<HelloMessage helloText="你好"/>,
	document.getElementById('example')
);
```

设置组件样式有三种：
1. 内联样式
2. 对象样式
3. 选择器样式

注意：在React和HTML5中设置样式时的书写格式区别：
1. HTML5以;结尾，React以,结尾
2. HTML5中key、value都不加引号，React中属于JavaScript对象，key的名字不能出现“-”，需要使用驼峰命名法。如果value为字符串，需要加引号。
3. HTML5中，value如果是数字，需要带单位，React中不需要带单位

注意：在React中使用选择器样式设置组件样式时，属性名不能使用class，需要使用className替换


```JavaScript

<style>
	.pStyle {
		font-size: 20px;
	}
</style>

var hstyle = {
	backgroundColor: "green",
	color: "red",
}

var ShowMessage = React.createClass({
	render: function () {
		return (
			<div style={{backgroundColor:"yellow", borderWidth:5, borderColor:"black", borderStyle:"solid"}}>
				<h1 style={hStyle}>{this.props.firstRow}</h1>
				<p className="pStyle">{this.props.secondRow}</p>
			</div>
		);
	}
});
```

### 复合组件

复合组件，又也叫做组合组件，创建多个组件合成一个组件

例子1：定义一个组件WebShow。功能：输出网站名字和网址，网址是一个可以点击的链接
分析：定义一个组件WebName负责输出网站名字，定义组件WebLink显示网站网址，并可以点击


```JavaScript
  // 定义WebName
  var WebName = React.createClass({
      render: function () {
          return <h1>{this.props.webname}</h1>;
      }
  });

  // 定义WebLink
  var WebLink = React.createClass({
      render: function () {
          return <a href={this.props.weblink}>{this.props.weblink}</a>
      }
  });

  // 定义WebShow
  var WebShow = React.createClass({
      render: function () {
          return (
              <div>
                  <WebName webname={this.props.wname}/>
                  <WebLink weblink={this.props.wlink}/>
              </div>
          )
      }
  });

  ReactDOM.render(
      <WebShow wname="淘宝" wlink="https://www.taobao.com"/>,
      document.getElementById('example')
  );
```

### props、state

props 是组件自身的属性，一般用于嵌套的内外层组件中，负责传递信息（通常是由浮层组件向子层组件传递）。

注意：props对象中属性与组件的属性一一对应，不要直接去修改props中属性的值。


#### ...this.props

其中props提供了一个语法糖：`...this.props`，可以将父组件中的全部属性都复制给子组件

例子：定义一个link，link组件中只包含一个超链接，不写任何属性，任何属性从父组件中拿到


```JavaScript
  var Link = React.createClass({
      render: function () {
          return <a {...this.props}>{this.props.name}</a>
      }
  });

  ReactDOM.render(
      <Link href="https://www.taobao.com" name='淘宝'/>,
      document.getElementById('example')
  );
```

#### this.props.childern

children是个例外，不是跟组件的属性对应，表示组件的所有子节点。

例子：定义一个列表组件，列表项中显示的内容，以及列表项的数量都由外部决定


```JavaScript
  var ListComponent = React.createClass({
     render: function () {
         return (
           <ul>
               {
                   /*
                   *  列表项数量以及内容不确定，在创建模板时确定
                   *  利用this.props.children从父组件获取需要展示的内容
                   *
                   *  获取到列表内容后，需要遍历设置
                   *  使用this.props.map方法
                   *  返回值：数组对象。这里数组中的元素是<li>
                   * */

                   React.Children.map(this.props.children, function (child) {
                       // child是遍历得到的父组件的子节点
                       return <li>{child}</li>
                   })
               }
           </ul>
         );
     }
  });

  ReactDOM.render(
      (
          <ListComponent>
              <h1>淘宝</h1>
              <a href="https://www.taobao.com">taobao</a>
          </ListComponent>
      ),
      document.getElementById('example')
  );
```

#### 属性验证 propType

用于验证组件实例的属性是否符合要求.


```JavaScript
  var ShowTitle = React.createClass({
      propTypes: {
          // title属性类型必须输字符串
          title: React.PropTypes.string.isRequired
      },
      render: function () {
          return <h1>{this.props.title}</h1>
      }
  })

  ReactDOM.render(
      <ShowTitle title=123 />,
      document.getElementById('example')
  );
```

#### 设置组件属性的默认值

通过实现组件的getDefaultProps方法，对组件设置默认值


```JavaScript
  var MyTitel = React.createClass({
      getDefaultProps: function () {
          return {
              title: 'O(∩_∩)O哈哈~'
          }
      },

      render: function () {
          return <h1>{this.props.title}</h1>
      }
  });

  ReactDOM.render(
      <MyTitel/>,
      document.getElementById('example')
  );
```


#### 事件处理

例子： 定义一个组件，组件中包括一个button，给button绑定onClick事件


```JavaScript
  var MyButton = React.createClass({

      handleClick: function () {
        alert("点击了");
      },
      render: function () {
          return (
              <button onClick={this.handleClick}>{this.props.buttonTitle}</button>
          )
      }
  });

  ReactDOM.render(
      <MyButton buttonTitle="我是个按钮"/>,
      document.getElementById('example')
  );
```

#### state 状态

例子1： 创建一个CheckButton组件，包含一个checkBox类型的输入框，复选框在选中和未选中两种状态下会显示不同的文字，即根据状态渲染。

```JavaScript
  var CheckButton = React.createClass({
     // 定义初始状态
      getInitialState: function () {
          return {
              isCheck: false
          }
      },
      // 定义事件绑定的方法
      handleChange: function () {
          // 修改状态值，通过this.state读取设置的状态
          this.setState({
             isCheck: !this.state.isCheck
          });
      },
      render: function () {
          // 根据状态值，设置显示的文字
          // 在JSX语法中，不能直接使用if，使用三目运算符
          var text = this.state.isCheck ? '已选中' : '未选中';

          return (
                <div>
                    <input type="checkbox" onChange={this.handleChange} />
                    {text}
                </div>
          );
      }
  });

  ReactDOM.render(
      <CheckButton/>,
      document.getElementById('example')
  );
```

例子2：定义一个组件，将用户在输入框内输入的内容实时显示


```JavaScript
	var Input = React.createClass({
       getInitialState: function () {
          return {
              value: '请输入'
          };
       },
       handleChange:function (event) {
           // 通过event.target.value读取用户输入的内容
           this.setState({
               value: event.target.value
           });
       },
       render: function () {
           var value = this.state.value;
           return (
               <div>
                   <input type="text" value={value} onChange={this.handleChange}/>
                   <p>{value}</p>
               </div>
           );
       }
   });

   ReactDOM.render(
       <Input/>,
       document.getElementById('example')
   );
```

## 总结
本章了解了JSX语法，学习了如何定义组件、设置组就按样式、创建符合组件，给组件设置属性等等，为之后学习ReactNative打基础。


