###浅析React 生命周期函数的使用  

#####1. constructor 构造函数  

* 执行时间： 组件被加载前最先调用，并且仅调用一次。  
* 作用：定义状态机变量，以及绑定this关键字。 
* 注意： 第一个语句必须是super()， 正确定义状态机代码如下  

		constructor(props) {
			super(props);
			this.state = {}
		}
		
#####2. componentWillMount  

* 执行时间： 组件初始渲染，render被调用之前调用，并且仅调用一次
* 作用： 如果在这个函数中调用setState改变某些状态机，react会等待setState完成后再渲染组件。 
* 注意： 子组件也有componentWillMount函数，在父组件的该函数调用后再被调用。  

#####3. render  

* 执行时间： componentWillMount之后， componentDidMount前。  
* 作用： 渲染挂载组件  
* 触发条件： 1）初始化加载页面 2）状态机改变state 3）接收到新的props
* 注意： 组件所必不可少的核心函数；不能在该函数中修改状态机state

#####4. componentDidMount 
 
* 执行时间：render之后被调用，并且仅调用一次  
* 作用：渲染挂在组件；可以使用refs（备注：React 支持一个特殊的属性，你可以将这个属性加在任何通过render返回的组件中，这也就是说对render返回的组件进行一个标记，可以方便的定位这个组件实例。）
* 注意： 子组件也有该函数，在父组件的该函数调用前被调用；如果在该函数中修改某些状态机state,会重新渲染render组件，所以有些组件为减少渲染次数，可以将某些状态机的修改放在componengWillMount函数中；如果需要在程序启动显示出页面后从网络获取数据，可以将网络请求代码放在该函数中。  

#####5. componentWillReceiveProps(nextProps)  
* 执行时间： 组件渲染后，当组件接收到新的props时被调用；这个函数接收一个object参数，props是父组件传递给子组件的。父组件执行render的时候子组件就会调用。  
* 作用：渲染挂载组件；可以使用refs
* 注意： react 初次渲染时，该函数并不会被处罚，ongoing因此有时该函数需要和componentWillMount 或 componentDidMount 组合使用；使用该函数一定要加nextProps参数。  

#####6. shouldComponent(nextProps, nextState)  
* 执行时间：组件挂载后，接收到新的state和props时被调用，即每次执行setstate都会执行该函数，来判断是否重新render组件，默认返回true；接收两个参数： 第一个是新的props，第二个是新的state.  
* 作用：如果有些变化不需要重新render组件，可以在该函数中阻拦。  
* 注意：该方法在初始化渲染的时候不会调用，在使用forceUpdate 方法的时候也不会  

#####7. componentWillUpdate  
* 执行时间：在接收到新的props 或者state，重新渲染之前立刻调用，在初始化渲染的时候该方法不会被调用  
* 作用：为即将发生的重新渲染做一些准备工作  
* 注意： 不能再将函数通过this.setstate再次改变状态机。如果需要则在componentWillReceiveProps 函数中改变。  

#####8. componentDidUpDate  
* 执行时间： 重新渲染后调用，在初始化渲染的时候方法不会被调用  
* 作用： 使用该方法可以在组件更新后操作元素  

#####9. componentWillUnmount
* 执行时间： 组件被卸载前被调用  
* 作用： 在该方法中执行任何必要的清理， 比如无效的定时器，或者清除在componentDidMount 中创建的DOM元素。  

#####10. 特别注意  
* 当一个页面中存在子父组件时，要注意componentWillMount 和componentDidMount的使用，如果需要先加载父组件（获取网络数据），父组件





































