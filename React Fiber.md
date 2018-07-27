###React Fiber 源码分析

##### 为什么要重写React  
React 16 以前，对virtual dom的更新和渲染是同步的。就是当一次更新或者一次加载开始以后，diff virtual dom以及渲染的过程是一口气完成的。如果组件比较深，相应的堆栈也会很深，长时间占用浏览器主线程，一些类似用户输入，鼠标滚动操作得不到响应。  

#####Fiber Reaconciler  
React 16 用了分片的方式解决上面的问题。  
就是把一个任务分成很多小片，当分配给这个小片的时间用尽的时候，就检查任务列表中有没有新的，优先级更高的任务，有就做这个新任务，没有就继续做原来的任务。这种方式被叫做异步渲染。  
##### Fiber 对开发者影响  
* componentWillMount componentWillUpdate componentWillReceiveProps, 几个生命周期方法不再安全，由于任务执行过程可以被打断，这个生命周期可能会执行多次，如果他们包含副作用，会有意向不到的bug。React团队提供了替换的生命周期方法。建议如果使用以上方法，尽量用纯函数。  
* 需要关注下React为任务片设置的优先级，特别是页面用动画的情况下。  

##### 如何试用Fiber异步渲染  
默认情况下，异步渲染没有打开，如果你想试用，可以：   
	
	import React from 'react';
	import ReactDOM from 'react-dom';
	import App from 'components/App';
	
	const AsyncMode = React.unstable_AsyncMode;
	const createApp = (store) => {
		<AsyncMode>
			<App store={store}/>
		</AsyncMode>
	};
	export default createApp;
	
代码将开启严格模式和异步模式，React16不建议使用的API会在控制台有错误提示，比如componentWillMount。  
#####Fiber如何工作  
首先，Fiber是什么  
> A Fiber is work on a Component that needs to be done or was done. There can be more than one per component. 

Fiber就是通过对象记录组件上需要做或者已经完成的更新，一个组件可以对应多个fiber.  
在render函数中创建的React Element 树在第一次渲染的时候会创建一颗结构一模一样的Fiber节点树。不同的React Element 类型对应不同的Fiber节点类型。一个React Element的工作就由它对应的Fiber节点来负责。  
一个React Element 可以对应不止一个Fiber，因为Fiber 在update的时候，会从原来的Fiber clone出一个新的Fiber。两个Fiber diff出的变化记录在alternate 上。所以一个组件在更新时最多会有两个fiber与其对应，在更新结束后alternate会取代之前的current的成为新的current节点。   
其次，fiber的基本原则：  
更新任务分成两个阶段，Reconciliation Phase. Reconciliation Phase 的任务是，找出要做的更新工作（diff Fiber Tree），就是一个计算阶段，计算结果可以被缓存，也可以被打断；commit Phase需要提交所有更新并渲染，为防止页面抖动，被设置为不能被打断。  

#####数据结构  
* fiber是个链表，有child何sibling属性，指向第一个子节点和相邻的兄弟节点，从而构建fiber tree。return属性指向其父节点。  
* 更新队列，updateQueue，是一个链表，有first何last两个属性，指向第一个和最后一个update对象。   
* 每个fiber有一个属性updateQueue，指向其对应的更新队列。  
* 每个fiber 有一个属性alternate，开始是指向一个自己的clone体，update的变化会先更新到alternate上，当更新完毕后，alternate替换current。  

更新入口肯定是setState方法，下面是Fiber的调用关系图。  
![](https://img-blog.csdn.net/20180428113757273?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FpcWluZ2ppbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

1. 用户操作引起setState被调用以后，先调用enqueueSetState方法，该方法可以划分成两个阶段，第一阶段Data Preparation，是初始化一些数据结构，比如fiber，updateQueue，update。  
2. 新的update会通过insertUpdate会通过insertUpdateIntoQueue方法，根据优先级插入到队列的对应位置，ensureUpdateQueues方法初始化两个更新队列，queuel和current.updateQueue对应，queue2和current.alternate.updateQueue对应。  




























