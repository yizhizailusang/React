### 虚拟 Dom 具体实现浅析
  
初学react只是停留在会用的层面，react内部运行的机制还处于懵懂阶段，那就从react Virtual DOM学起~


#### jsx的背后
说起虚拟DOM 就不得不提jsx，浏览器无法识别jsx, 所以必须将jsx语法汇编成javascript,说的通俗些就是转化成js代码。</br>

举个栗子：<br>

	<input 
		className='jsx-ele'
	/>
以上代码会被babel转成浏览器可以识别的javascript代码

	React.createElement('div', {className: 'jsx-ele'}, '');
	
仔细研究一下这些参数：  
* 第一个是元素的type.对于HTML标签，它将是一个带有标签名称的字符串  
* 第二个参数是一个包含该元素所有属性的对象。  
* 剩下的参数是元素的子元素。 
 
#####children的类型：  
除了 string、React.createElement 可以作为子元素，还允许：  
* 基本类型  false, null, undefined, true  
* 数组  
* React component  

可以使用数组作为参数，传递children  

	React.createElement(
		'div',
		{className: 'react-wrap'},
		['hello react!', React.createElement('br'), 'are you right?']
	)
	
当我们这样使用一个组件时，
	
	<Table rows={rows} />
	
都会被编译成如下：

	React.createElement(Table, {rows: rows})
	
这里和创建元素不同的是Table不是一个字符串，而是一个引用，指向我们编写组件时的函数。组件的
attribute 现在接收的是props参数了。

#### 把组件（component）组合成页面

我们已经将jsx组件转换成纯javascript, 现在我们有一大堆的函数调用， 他的参数会被其他函数调用，或者还有更多的其他函数调用这些参数。。。那jsx通过createElement 是怎么转换成实体DOM的？

为此，我们需要引入ReactDOM 库，如下：

	function Table (row) {/* ... */}
	ReactDOM.render(
		React.createElement(Table, {rows: rows}), // 'creating' a component
		document.getElementById('#root')
	);
	
当ReactDOM.render 被调用时，React.createElement 最终也会被调用，返回一下对象：
	
	{
		type: Table,
		props: {
			rows: rows
		}
	}
	
这些对象，在React角度上构成了虚拟DOM.
他们将在所有进一步渲染中相互比较，并最终转换成真正的DOM
	
在构建虚拟DOM对象完成后，ReactDOM.render 将会按下面的原则，尝试将其转换为浏览器可以识别和展示的DOM节点。

* 如果type是一个string类型的标签（tag name）———— 创建一个标签，附带上props下所有attributes.  
* 如果type 是一个函数或者类, 调用他，并对结果进行递归。  
* 如果props 下有children属性 -- 在父节点下，针对每个child重复以上过程。

最后得到以下HTML	

	<table>
	  	<tr>
	   		<td>Title</td>
	  	</tr>
  	...</table>
	
	
#### 重新构建DOM  
在实际应用场景中，render通常在根节点调用一次， 后续的更新会有state来控制和触发调用。		
当state或者props改变时，react 会启动diff算法，来确定节点哪些部分必须更新，哪些可以保持不变。
那整个流程是怎样就行工作的，先让我们看几个场景，理解他们对我们优化性能有很大帮助。（以下是React Virtual DOM 对象）  
场景1：  type 是字符串，type在通话中保持不变，props也没有改变  

	// before update
	{ type: 'div', props: { className: 'react-wrap' } }
	// after update
	{ type: 'div', props: { className: 'react-wrap' } }
这是最简单的场景，DOM不会更新

场景2： type 仍然是相同的字符串，props是不同的。
	
	//befor update
	{ type: 'div', props: { className: 'react-wrap' } }
	// after update
	{ type: 'div', props: { className: 'react-wrap' } }  
	
React 知道如何通过标准DOM API 调用来更改元素的属性，而无需从DOM中删除一个节点
场景3： type 已更改为不同的string
	
	// before update
	{ type: 'div', props: { className: 'react-wrap' } }
	// after update
	{ type: 'span' props: { className: 'react-wrap' }
	
react 看到不同的type 它甚至不会尝试更新节点，old元素将直接被删除。因此，将元素替换为完全不同DOM树的东西代价会非常昂贵。
划重点，react 使用 '===' 来比较type值，所以这两个值需要是相同类，或相同函数的相同实例。
	
场景4： type是一个component  
	
	// before update
	{ type: Table, props: { rows: rows } }
	// after update
	{ type: Table, props: { rows: rows } }
	
表面看 type和props属性都没有变化， 但是在这里type是引用类型（func 或者 类），并且react会进行tree diff过程，react会逐层递进的去检查组件内部逻辑，以确保render返回值不会改变。是的————对树种的每个组件进行遍历扫描，在复杂场景下，成本可能会非常昂贵。
	
#####多个子组件（children）的情况

除了上述四种单一场景外，实际开发中会遇到四种情况的综合。如下

	props: {
		children: [
			{ type: 'div' }
			{type: 'span'}
			{type: 'br'}
		]
	}
	
实际开发中我们想要交换一个 ‘div’ 和 ‘span’ 的位置。
	
	props: {
		children: [
			{ type: 'span' }
			{ type: 'div' }
			{ type: 'br' }
		]
	}
	
当react diff 的时候，如果按照常规的diff算法，检查props.children 下的数组时，按顺序对比组内元素的话： index 0 和 index 0进行比较， index1 和index1 进行比较。对于每一次对比，react会使用之前的diff规则， 例子中他认为div变成span，类似场景3，这样操作dom，效率很低。再比如，我们删除第一个div，下面我们具体讲解 <!--**_`react 是如何高效的更新dom？？？关于react的对比算法，我们会在其他章节具体介绍`_**-->

####代码优化
通过以上场景我们简单了解reactDOM 更新机制，那在实际开发中我们可以怎么避免性能浪费，优化我们的代码呢？  

**case1:**  

	// ...
	props: {
	  children: [
	    { type: Message },
	    { type: Table },
	    { type: Footer }
	  ]
	}

现在我们接收到领导的通知，要把Message 从DOM开除掉，留下Table和Footer。 

	// ...
	props: {
	  children: [
	    { type: Table },
	    { type: Footer }
	  ]
	}

那么react 会怎么操作呢？现在children 的index 0 是Table， 比较type时发现 index 0 完全不是一个string 类 或者 function。于是会把整个Table unmount，然后再amount回去，重新渲染整个Table，这种状况不是我们想看到的，那该怎么解决，这里有一个小技巧。

	// Using a boolean trick
	<div>
	  {isShown && <Message />}
	  <Table />
	  <Footer />
	</div>
	
虽然Message被领导干掉了，但公司还有一个名额。 请记住 true false undefined null 都是虚拟DOM的允许值。
最终得到

	// ...
	props: {
	  children: [
	    false, //  isShown && <Message /> evaluates to false
	    { type: Table },
	    { type: Footer }
	  ]
	}
	// ...


**case2**  
	平时开发中高阶组件使用的非常多，高阶组件时一个将组件座位参数，执行某些操作，最后返回一个功能不同的组件。
	
	function withName(SomeComponent) {
	  // Computing name, possibly expensive...
	  return function(props) {
	    return <SomeComponent {...props} name={name} />;
	  }
	}
	
现在看如下代码

	class App extends React.Component() {
	  render() {
	    // Creates a new instance on each render
	    const ComponentWithName = withName(SomeComponent);
	    return <SomeComponentWithName />;
	  }
	}
	
我们在render函数里创建一个HOC，当我们重新渲染时 DOM 树，虚拟DOM是这样的
	
	// On first render:
	{
	  type: ComponentWithName,
	  props: {},
	}
	// On second render:
	{
	  type: ComponentWithName, // Same name, but different instance
	  props: {},
	}
		
ComponentWithName 虽然名字相同但是引用指向了不同的实例， diff失败，那这个问题怎么解决呢？
	
	// Creates a new instance just once
	const ComponentWithName = withName(Component);
	class App extends React.Component() {
	  render() {
	    return <ComponentWithName />;
	  }
	}
	
case3：  
	以上两种避免都是在dom渲染时做的优化，那有没有一种可能，提前预判避免到无谓的dom更新呢？如果了解react 的渲染周期，react 内置了shouldComponentUpdate，这个方法是在接受到新的props或者state时调用，返回true 或 false， 返回false 将不会重新渲染dom，也不会对比他的子组件。
	
	通常来说，对比两个集合 props 和state 一个简单的比较就足够了。
	
	class Table extends React.component {
		shouldComponentUpdate(nextprops, nextState) {
			const {props, state} = this;
			return !shallowequal(props, nextProps) 
					&& !shallowequal(state, nextState)
		}
		render() {}
	}
	
在最新版本的react里已经支持这个功能，react 为我们提供了PureComponent 类，类似于component，只是PureComponent里的 shouldComponent已经为你实现了一个浅比较。
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	