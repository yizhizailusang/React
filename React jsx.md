###JSX 入门

简介：  

  在使用React开发中，可以在js语法中使用dom元素的各种标签，以及可以使用自定义的react Component。支持这些功能我们需要做哪些前置工作呢？  
	
#### 安装

打开终端，安装全局babel-cli （默认已有node环境）
	
	npm install -g babel-cli
	
然后进入项目目录安装
	
	npm install --save-dev babel-cli babel-core babel-preset-es2015 babel-preset-react

#### 配置
在项目目录中创建.babelrc 配置文件。 该文件用来设置转码规则和配置插件，格式如下

	{
		"presets": [],
		"plugins": []
	}
	
presets 字段设定转码规则，我们安装了babel-preset-es2015 和 babel-preset-react；前者是将es6转换成es5， 后者是react的转码规则。  
将这两个规则写入 .babelrc 中

	{
		"presets": [
		 	"es2015", "react"
		],
		"plugins": []
	}

####编译
在src目录下创建一个test.js文件

	class Test extends React.Component {
		render() {
			return (
				<h1>hello jsx</h1>
			);
		}
	}
	ReactDOM.render(<Test />, document.body);
	
在终端里执行babel 编译语句  
	
	babel src -d lib
	
执行成功在lib文件多一个test.js文件，然后查看代码

	var Test = function (_React$Component) {
    _inherits(Test, _React$Component);

    function Test() {
        _classCallCheck(this, Test);
        return _possibleConstructorReturn(this, (Test.__proto__ || Object.getPrototypeOf(Test)).apply(this, arguments));
    }
    
    _createClass(Test, [{
        key: "render",
        value: function render() {
            return React.createElement(
                "h1",
                null,
                "test"
            );
        }
    }]);
    
    return Test;
}(React.Component);

ReactDOM.render(React.createElement(Test, null), document.body);




	



















