# React 详细解析
##### 前沿
React 16的推出可以说是react 里程碑，在原来15版本stack Reconciliation 锐变成了 fiberReconciliation，提出了纤程的概念。  
React 的定位是一个构建用户界面的javascript类库，它使用javascript语言开发ui组件，可以使用多种方式渲染这些组件，输出用户界面，较大程度达到了跨技术栈平台的兼容重用：  
> We don’t make assumptions about the rest of your technology stack, so you can develop new features in React without rewriting existing code.  

现在的react依然在一下几个方面发挥的不错:   
1. react web 应用用户界面开发；  
2. react native app 用户界面开发；  
3. nodejs服务端渲染；  

在这些不同场景，渲染的主题很明显是不一样的，有诸如web应用的dom渲染，React Native的原生view渲染，服务器端字符串渲染等，要做到兼容适应多种不同渲染环境，很显然，React 不能局限固定渲染ui的方式。  
React 核心内容也确实只包含定义组件相关的内容和api，实际项目需要使用如下代码：  

	import React from 'react';
	
这句代码做的就是引入了React核心源码模块。   

#####渲染  
上一节已经说到React 核心内容只涉及如何定义组件，并不涉及具体的组件渲染，这需要额外引入渲染模块，以渲染React定义的组件：  
1. React DOM 渲染模块： 将React 组件渲染为DOM，然后可以被浏览器处理呈现给用户，这就是通常在web应用中引入的react-dom模块：  

	import React from 'react';
	import { render } from 'react-dom';
	import App from './apps/App.js';
	
	render(
	 <App />,
	 document.getElementById('mainBox')
	);
	
如上代码，App是使用React 核心模块定义的组件，然后使用React-dom渲染模块提供的render方法将其渲染为DOM输出至页面。  
2.  Reat Native 渲染： 将React 组件渲染为移动端原生view，在React Native应用中引入react-native模块，它提供相应渲染方法可以渲染react组件：  

	import {AppRegistry} from 'react-native';
	import App from './src/app.js';
	
	AppRegistry.registerComponent('fun', ()=> App);

如上，App 是react根组件，使用react-native渲染器的AppRegistry.registerComponent方法将其渲染成为原生view。  

3.React 测试渲染： 将React组件渲染为JSON树，用来完成jest的快照测试，内容在react-test-renderer模块： 
 	import ReactTestRenderer from 'react-test-renderer';  
 	
 	const renderer = ReactTestRenderer.create(
 		<Link page='https://www.facebook,com/'>Facebook</link>
 	);
	
	console.log(renderer.toJSON());
	// { type: 'a',
	//   props: { href: 'https://www.facebook.com/' },
	//   children: [ 'Facebook' ] }
	
4. React 矢量图渲染： 将React组件渲染为对应的矢量图。
	
web React 应用是最常见的，也是最易于理解的，所以本篇后均从React-DOM渲染角度解析fiber。  
调和（Reconciliation）  
如前面两节所述，React核心是定义组件，渲染组件方式由环境决定，定义组件，组件状态管理，生命周期方法管理，组件更新等应该跨平台一致处理，不受渲染环境影响，这部分内容统一由调和器处理，源码传送，不同渲染器都会使用该模块。 调和器主要作用就是在组件状态变更时，调用组件树各组件的render方法，渲染，卸载组件。  

Stack Reconciler  
我们知道浏览器渲染引擎是单线程的，在react 15版本以及之前，计算组件树变更时将会阻塞整个线程，整个渲染过程是连续不中断完成的，而这时的其他任务都会被阻塞，如动画等，这可能会使用户感觉到明显卡顿，比如当你在访问某一网站时，输入某个搜索关键字，更优先的应该是交互反馈动画效果，如果交互反馈延迟200ms，用户则会感觉较明显的卡顿，而数据响应晚200ms并没有太大问题。这个版本的调和器可以称为栈调和器，其调和算法大致过程见[React diff算法](http://blog.codingplayboy.com/2016/10/27/react_diff/)	  
Stack Reconcilier的主要缺陷就是不能暂停渲染任务，也不能切分任务，无法有效平衡组件更新渲染与动画相关任务间的执行顺序，既不能划分任务优先级，有可能导致重要任务卡顿，动画掉帧等问题。  
Fiber Reconciler  
React 16 版本提出了一个更先进的调和器，它允许渲染进程分段完成，而不必须一次性完成，中间可以返回至主进程控制执行其他任务。而这时通过计算部分组件树的变更，并暂停渲染更新，询问进程是否有更高需求的绘制或者更新任务需要执行，这些高需求的任务完成后才开始渲染。这一切的实现是在代码层引入一个新的数据结构Fiber对象，每一个组件实例对应有一个fiber实例，此fiber实例负责管理组件实例的更新，渲染任务及与其他fiber实例的联系。  
这个新推出的调和器就叫做纤维调和器，它提供的新功能主要有：  
1. 可切分，可中断任务；  
2. 可重用各阶段任务，且可以设置优先级  
3. 可以在父子组件任务间前进后退切换任务；  
4. render方法可以返回多元素；  
5. 支持异常边界处理异常  

Fiber 与 javascript  
前面说到Fiber可以异步实现不同优先级任务的协调执行，那么对于DOM渲染器而言，在javascript层是否提供这种方式呢，还是说只能使用setTimtout模拟呢？目前新版本主流浏览器已经提供API： requestIdleCallback requestAnimationFrame： 

1. requestIdleCallback 在线程空闲时期调度执行低优先级函数。  
2. requestAnimationFrame 在下一个动画帧调度执行高优先级函数。  

Fiber 与 requestIdleCallback  
Fiber 所做的就是需要分解渲染任务，然后根据优先级使用API调度，异步执行指定任务：   
1. 低优先级任务由requestIdleCallback处理；  
2. 高优先级任务由requestAnimationFrame处理；   
3. requestIdleCallback  可以在多个空闲期调用空闲期回调，执行任务；   
4. requestIdleCallback 方法提供deadline， 即任务执行限制时间，以切分任务，避免长时间执行，阻塞ui渲染而导致掉帧。  

具体实现如下：  
1. 若支持requestIdleCallback如下  

	rIC = window.requestIdleCallback;
	cIC = window.cancelIdleCallback;
	export {now, rIC, cIC};
	

2.若不支持，则自定义实现：  

	let isIdleSchduled = false; // 是否在执行空闲期回调  
	let frameDeadlineObejct = {
		didTimeout: false,
		timeRemaning() {
			const remaning = frameDeadline - now();
			// 计算当前帧运行剩余时间
			return remaning > 0 ? remaning : 0;
		}
	}
	
	// 帧回调
	const animationTick = function (rafTime) {
		if (!isIdleScheduled) {
			// 不在执行空闲期回调，表明可以调用空闲期回调
			isIdleScheduled = true;
			// 执行空闲期回调。
			idleTick();
		}
	};

	// 空闲期回调
	const idleTick = function () {
		// 重置为false，表明可以调用空闲期回调
		isIdleSchedule = false;
		const currentTime = now();
		if (frameDeadline - currentTime <= 0) {
			// 帧到期时间小于当前时间，说明已过期
			if (timeoutTime !== -1 && timeoutTime <= currentTime) {
				// 此帧已过期，且发生任务处理函数的超时
				// 需要执行任务处理，下文将调用；
				frameDeadlineObject.didTimeout = true;
			}
			else {
				// 帧已过期，但没有发生任务处理函数的超时，暂时不调用任务处理函数
				if (!isAnimationFrameScheduled) {
					// 当前没有调度别的帧的回调函数
					// 调度下一帧
					isAnimationFrameScheduled = true;
					requestAimationFrame(animationTick);
				}
				return;
			}
		}
		else {
			// 这一帧还有剩余时间
			// 标记未超时，之后调用任务处理函数
			frameDeadlineObject.didTimeout = false;
		}
		timeoutTime = -1; // 缓存的任务处理函数
		const callback = scheduledRICCallback;
		scheduledRICCallback = null;
		if (callback != null) {
			// 执行回调
			callback(frameDeadlineObject);
		}
	}
	
	rIC = function (
		callback: (deadline: Deadline) => void,
		options ?:{timeout: number}
	) {
		// 回调函数
		scheduledRICCallback = callback;
		if (options != null && typeof options.timeout === 'number') {
			// 计算过期时间
			timeoutTime = now() + options.timeout;
		}
		if (!isAnimationFrameScheduled) {
			// 当前没有调度别的帧回调函数
			isAnimationFrameScheduled = true;
		}
		if (!isAnimationFrameScheduled) {
			// 当前开始执行帧回调函数
			isAnimationFrameScheduled = true;
			// 初始开始执行帧回调
			requestAnimationFrmae(animationTick);
		}
		return 0;
	}

1. frameDeadline:是以启发法，从30fps开始调整得到的更适于当前环境的一帧限制时间；  
2. timeRemaining: 计算requestIdleCallback 此次空闲执行任务剩余时间，即距离deadline的时间。  
3. options.timeout:fiber内部调用 rIC API执行异步任务时，传递的任务到期时间参数；  
4. frameDeadlineObject： 计算得到的某一帧可用时间对象，两个属性分别表示： 
    didTimeout: 传入的异步任务处理函数是否超时；
    timeRemaining：当前帧可执行任务处理函数的剩余空闲时间
5. frameDeadlineObject对象是基于传入的timeout参数和此模块内部自调整得到frameDeadline参数计算得出

Fiber组件  
我们已经知道了Fiber的功能以及其主要特点，那么其如何和组件联系的，并且如何实现效果的呢，一下几个点可以概括：
1. React应用中的基础单元是组件，应用以组件树形式组织，渲染组件；  
2. Fiber调和器基础单元则是fiber，应用以fiber树形式组织，应用fiber算法；  
3. 组件树和fiber树结构对应，一个组件实例有一个对应的fiber实例；  
4. fiber负责整个应用层面的调和，fiber实例负责对应组件的调和；  
注意Fiber和fiber的区别，Fiber是指调和器算法，fiber则是调和器算法组成单元，和组件与应用关系类似，每个组件实例会有对应的fiber实例负责该组件的调和。  
Fiber数据结构  
截止目前，我们对Fiber应该有了初步的了解，在具体介绍Fiber的实现与架构之前，准备先简单介绍一下Fiber的数据结构，数据结构能一定程度反映其整体工作架构。  
其实，一个fiber就是一个javascript对象，以键值对形式存储了一个关联组件的信息，包括组件接收的props，维护的state，最有需要渲染出内容等，接下来我们将介绍Fiber对象的主要属性。  

Fiber对象  

	// 一个Fiber对象作用一个组件  
	export type Fiber = {|
		// 标记Fiber类型tag
		tag: TypeOfWork,
		// fiber 对应的function/class/module类型组件名。
		type: any,
		// fiber 所在组件树的根组件FberRoot对象
		stateNode:any,
		// 处理完当前Fiber后返回的fiber，
		// 返回当前fiber所在fiber树的父级fiber实例
		return: Fiber | null,
		// fiber 树结构相关连接 
		child: Fiber | null,
		sibling: Fiber | null,
		index: number,
		
		// 当前处理过程中的组件props对象
		pendingProps: any,
		// 缓存之前组件porps对象
		memoizedProps:any,
		memoizedState: any,
		
		// 组件状态更新及对应回调函数的存储队列
		updateQueue: UpdateQueue<any> | null,
		
		// 描述当前fiber实例及其子fiber树的数位，
	  // 如，AsyncUpdates特殊字表示默认以异步形式处理子树，
	  // 一个fiber实例创建时，此属性继承自父级fiber，在创建时也可以修改值，
	  // 但随后将不可修改。
	  internalContextTag: TypeOfInternalContext,
	  
	  // 更新任务的最晚执行时间
	  expirationTime: ExpirationTime,
	  
	  // fiber的版本池，即记录fiber更新过程，便于恢复
	  alternate: Fiber | null,
	|}
1. type & key: 同React元素的值；  
2. type： 描述fiber对应的react组件；
	对于组合组件：值为function或class组件本身；  
	对于原生组件： 值为该元素类型字符串；
3. key: 调和阶段，表示fiber，以检测是否可重用该fiber实例；
4. child & sibling: 组件树，对应生成fiber树，类比的关系；  
5. pendingProps & memoizedProps： 分别表示组件当前传入及之前的props；  
6. return： 返回当前fiber所在fiber树的父级fiber实例，即当前组件的父组件对应的fiber；  
7. alternate： fiber的版本池，即记录fiber更新过程，便于恢复重用；  
8. workInProgress: 正在处理的fiber，概念上叫法，实际上没有此属性；  


#####ALTERNATE FIBER  
可以理解为一个fiber版本池，用于交替记录组件更新过程中fiber的更新，因为在组件更新的各阶段，更新前及更新过程中fiber状态并不一致，在需要恢复时，既可使用另一者直接回退至上一版本fiber。  
> 1. 使用alternate属性双向连接一个当前fiber和其workinprogress，当前fiber实例的alternate属性指向其workinprogress，workinprogress的alternate属性指向当前稳定的fiber；
> 2. 当前fiber的替换版本是其workinprogress，workinprogress的交替版本是当前的fiber；
> 3. 当workinprogress更新一次后，将同步至当前fiber，然后继续处理，同步直至任务完成；
> 4. workinprogress 指向处理过程中的fiber，而当前fiber总是维护处理完成的新版本的fiber；

#####创建FIBER实例  
创建fiber实例即返回一个带有上一个小节描述的诸多属性的javascript对象，FiberNode即根据传入的参数构造返回一个初始化的对象。

	var createFiber = function (
		tag: TypeOfWork,
		key: null | string,
		internalContextTag: TypeOfInternalContext,
	) {
		return new FiberNode(tag, key, internalContextTag);
	}
	
创建alternate fiber 以处理任务的实现如下： 
	
	// 创建一个alternate fiber 处理任务
	export function createWorkInprogress(
		current: Fiber,
		pendingProps: any,
		expirationTime: ExpirationTime,
	) {
		let workInProgress = current.alternate;
		if (workInProgress === null) {
			workInprogress = createFiber(
				current.tag,
				current.key,
				current.internalContextTag,
			);
			workInProgress.type = current.type;
			workInProgress.stateNode = current.stateNode;
			
			// 形成alternate关系，互相交替模拟版本池
			workInProgress.alternate = current;
			current.alternate = workInProgress;
		}
		
		workInProgress.expirationTime = expirationTime;
		workInProgress.pendingProps = pendingProps;
		workInPorgress.child = current.child;
		workInProgress.memoizedProps = current.memoizedProps;
		workInProgress.memoziedState = current.memoizedState;
		workInProgress.updateQueue = current.updateQueue;
		...
		return workInProgress;
	}

#####Fiber 类型
上一小节，Fiber对象中有个tag属性，标记fiber类型，而fiber实例是和组件对应的，所以其类型基本上对应于组件类型
	
	export type TypeOfWork = 0 | 1| 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10;
	
	export const IndeterminateComponent = 0; // 尚不知是类组件还是函数式组件
	export const FunctionalComponent = 1; // 函数式组件
	export const ClassComponent = 2; // class类组件
	export const HostRoot = 3; // 组件树根组件，可以嵌套
	export const HostPortal = 4; // 子树，
	export const HostComponet = 5; // 标准组件，如div span等
	export const HostText = 6; // 文本
	export const CallComponent = 7; // 组件调用 
	export const CallHandlerPhase = 8; // 调用组件方法
	export const ReturnComponent = 9; // placeholder 占位符
	export const Fragment = 10; // 片段
	
在调度执行任务的时候会根据不同类型fiber，即fiber.tag值进行不同处理。  

#####FiberRoot对象  
FiberRoot对象，主要用来管理组件的更新进程，同时记录组件树挂载的DOM容器相关信息，具体定义见ReactFiberRoot模块。
	
	export type FiberRoot = {
		// Fiber 节点的容器元素相关信息，通常会直接传入容器元素
		containerInfo: any,
		// 当前fiber树种激活状态的fiber及诶单
		current: Fiber,
		// 此节点剩余的任务到期时间
		remainingExpirationTime: ExpirationTime,
		// 更新是否可以提交
		isReadyForCommit: boolean,
		// 准备好提交的已处理完成workinprogress
		finishedWork: Fiber | null,
		// 多组件树FiberRoot对象以单链表存储连接，指向下一个需要调度的FiberRoot
		nextScheduleRoot: FiberRoot | null,
	};
	
创建FIBERROOT实例
	
	import {
		ClassComponent,
		HostRoot
	} from 'shared/ReactTypeOfWork';

	// 创建返回一个初始根组件对应的fiber实例
	function createHostRootFiber():Fiber () {
		// 创建fiber
		const fiber = createFiber(HostRoot, null, NoContext);
		return fiber;
	}

	export function createFiberRoot(
		containerInfo: any,
		hydrate: boolean,
	){
		// 创建初始根组件对应的fiber实例
		const uninitializedFiber = createHostFiber();
		// 组件树根组件的FiberRoot对象
		const root = {
			// 根组件对应的fiber实例
			current: uninitializedFiber,
			containerInfo: containerInfo,
			pendingChildren: null,
			remainingExpirationTime: NoWork,
			isReadyForCommit: false,
			finishedWork: null, 
			context: null,
			pendingContext: null,
			hydrate,
			nextScheduledRoot: null
		};
		// 组件树根组件fiber实例的stateNode指向FiberRoot对象
	}


#####ReactChildFiber
在生成组件树的FiberRoot对象后，会为了子组件生成各自的fiber实例，这一部分由ReactChildFiber模块实现： 

	// 调和子fiber
	export const reconcileChildFibers = ChildReconciler(true);
	// 初始化子fiber
	export const mountChildFibers = ChildReconciler(false);
	
而childReconciler 方法所做的则是根据传入参数判断是调用初始化子组件fibers逻辑还是执行和已有调和已有子组件fiber逻辑  
ChildReconciler 方法，返回reconcileChildFibers方法：  
1. 判断子级传递内容的数据类型，执行不同的处理，这也对应着我们写react组件时传递props.children时，其类型可以是对象或数组，字符串，是数字等。  
2. 然后具体根据子组件类型，调用不同的具体调和处理函数；
3. 最后返回根据子组件创建或更新得到的fiber实例；

	function ChildReconciler(a) {
		function reconcileChildFibers(
			returnFiber: Fiber
			currentFirstChild: Fiber | null,
			newChild: any,
			exirationTime: ExpirationTime
		){
			const isObject = typeof newChild === 'object' && newChild !== null;
			
			if (isObject) {
				switch (newChild.$$typeof) {
					// React Elment
					case REACT_ELEMENT_TYPE:
						return placeSingleChild(
							reconcileSingleElement(
								returnFiber,
								currentFirstChild,
								newChild,
								expirationTime
							)
						);
					// 组件调用
					case REACT_CALL_TYPE:
						return placeSingleChild(reconcileSingleCall(...));
					....
				}
			}
			if
		}
	}

####Fiber架构  
#####优先级（ExpirationTime VS PrioritylLevel）
我们已经知道fiber可以切分任务并设置不同优先级，那么是如何实现划分优先级的呢，其表现形式是什么呢？
#####EXPIRATIONTIME 
Fiber切分任务并调用requestIdleCallback 和requestAnimationFrame API,保证渲染任务和其他任务，在不影响应用交互，不掉帧的前提下，稳定执行，而实现调度的方式正式给每个fiber实例设置到期执行时间，不同时间即代表不同优先级，到期时间越短，则代表优先级越高，需要尽早执行  

> 所谓的到期时间，是相对于调度器初始调用的起始时间而言的一段时间；调度器初始调用后的某段时间内，需要调度完成这项更新，这个时间段长度值就是到期时间值。
> 

Fiber提供ReactFiberExpirationTime模块实现到期时间的定义：

	export const NoWork = 0; // 没有任务等待处理
	export const Sync = 1; // 同步模式，立即处理任务
	export const Never = 2147483647; // Max int32: Math.pow(2, 31) - 1
	const UNIT_SIZE = 10; // 过期时间单元
	const MAGIC_NUMBER_OFFSET = 2; // 到期时间偏移量

	// 以ExpirationTime 特定单位表示的到期执行时间
	// 1 unit of exporation time represents 10ms
	export function msToExpirationTime(ms) {
		// 总是增加一个偏移量，在ms<10时与nowork模式进行区别
		return ((ms / UNIT_SIZE) | 0) + MAGIC_NUMBER_OFFSET; 
	}

	// 以毫秒表示的到期执行时间
	export function expirationTimeToMs(expirationTime: Exiration)
-------------------------
更新器（Updater）
调度器协调，调度的任务主要就是执行组件或组件树更新，而这些任务则具体由更新器（updater）完成，可以说调度器是在整个应用组件树层面掌控全局，而更新器则深入到每个更具体的每一个组件内部执行。  
每个组件实例化时都会被注入一个更新器，负责协调组件与React核心进程的通信，其职责主要可以概括为以下几点：
1. 找到组件实例对应的fiber实例；  
2. 询问调度器当前组件fiber实例的优先级；  
3. 将更新推入fiber的更新队列；  
4. 根据优先级调度更新任务；  

	export default function (
		scheduleWork:(fiber: Fiber, expirationTime: ExpirationTime) => void,
		computeExpirationForFiber:(fiber: Fiber) => ExpirationTime,
		memoizeProps: (workInprogress: Fiber, props: any) => void,
		memoizeState: (workInProgress: Fiber, state: any) => void,
	) {
		const updater = {
			isMounted,
			// 状态变更，更新入队列
			enqueueSetState(instance, partialState, callback) {
				// 获取fiber
				const fiber = ReactInstanceMap.get(instance);
				const expirationTime = computeExpirationForFiber(fiber);
				// 创建更新任务
				const update = {
					expirationTime,
					partialState,
					callback,
					isReplace: false,
					isForced: false,
					nextCallback: null,
					next: null,
				};
				// 添加更新任务至fiber
				insertUpdateIntoFiber(fiber, update);
				// 调用调度器API以调度fiber任务
				scheduleWork(fiber, expirationTime);
			},
			// 替换状态时
			enqueueReplaceState(instance, state, callback) {
				const fiber = ReactInstanceMap.get(instance);
				const expirationTime = computeExpirationForFiber(fiber);
				
				const update = {
					expirationTime,
					partialState: state,
					callback,
					isRepalce: true,
					isForced: false,
					nextCallback: null,
					next: null
				};
				// 添加更新任务至fiber
				insertUpdateIntoFiber(fiber, update);
				// 调用调度器API以调度fiber任务
				scheduleWork(fiber, expirationTime);
			},
			// 强制更新
			enqueueForceUpdate(instance, callback) {
				const fiber = ReactInstanceMap.get(instance);
				const expirationTime = computeExpirationForFiber(fiber);
				
				const update = {
					expirationTime,
					partialState: null,
					callback,
					isReplace: false,
					isForced: true,
					nextCallback: null,
					next: null
				};
				
				insertUpdateIntoFiber(fiber, update);
				scheduleWork(fiber, expirationTime);
			},
		};
		
		// 调用组件实例生命周期方法并调用更新器API
		function callComponentWillReceiveProps(
			workInProgress, instance, newProps, newContext
		) {
			const oldState = instance.state;
			instance.computeWillReceiveProps(newProps, newContext);
			
			if (instance.state !== oldState) {
				// 调用更新器入队列方法
				updater.enqueueRepaceState(instance, instance.state, null);
			}
		}
		
		// 设置class 组件实例的更新器和fiber
		function adoptClassInstance(workInProgress, instance): {
			// 设置更新器
			instance.updater = updater;
			workInProgress.state.Node = instance;
			// 设置fiber
			ReactInstanceMap.set(instance, workInProgress);
		}
		
		// 实例化class组件实例
		function constructClassInstance(workInProgress, props) {
			const ctor = workInProgress.type;
			const unmaskedContext = getUnmaskedContext(workInProgress);
			const needsContext = isContextConsumer(workInProgress);
			const context = needsContext
				? getMaskedContext(workInProgress, unmaskedContext)
				: emptyObject;
			// 实例化组件类型
			const instance = new ctor(props, context);
			// 设置class 实例的更新器和fiber
			adoptClassInstance(workInProgress, instance);
			return instance;
		}
		
		// 挂载组件实例
		function mountClassInstance(
			workInProgress, renderExpirationTime
		) {
			if (typeof instance.componentWillMount === 'function') {
				callComponentWillMount(workInProgress, instance);
			}
		}
		
		// 更新组件实例
		function updateClassInstance(
			current, workInProgress, renderExpirationTime
		) {
			// 组件实例
			const instance = workInProgress.stateNode;
			// 原props或新props
			const oldProps = workInprogress.memoizeProps;
			let newProps = workInProgress.pendingProps;
			
			if (!newProps) {
				// 没有新props则直接使用props
				newProps = oldProps;
			}
			
			if (typeof instance.componentWillReceiveProps === 'function' &&
				(oldProps !== newProps)
			) {
				// 调用方法进行更新器相关处理
				callComponentWillReceiveProps(
					workInProgress, instance, newProps
				);
			}
			
			// 根据原状态对象和更新队列计算得到新状态对象
			const oldState = workInProgress.memoizedState;
			let newState;
			if (workInProgress.updateQueue !== null) {
				// 处理更新队列更新，计算得到新的state对象
				newState = processUpdateQueue(
					current,
					workInProgress,
					workInProgress.updateQueue,
					instance,
					newProps,
					renderExpirationTime
				);
			}
			else {
				newState = oldState;
			};
			
			const shouldUpdate = checkShouldComponentUpdate(...);
			
			if (shouldUpdate) {
				if (typeof instance.componentWillUpdate === 'function') {
					instance.componentWillUpdate(newProps, newState, newContext);
				}
			}
			// 调用生命周期方法
			...
			return shouldUpdate;
		}
		
		return {
			adoptClassInstance,
			constructClassInstance,
			mountClassInstance,
			updateClassInstance
		};
	}
	
	
主要实现以下几个功能：  
1. 初始化组件实例并为其设置fiber实例和更新器；  
2. 初始化或更新组件实例，根据跟新队列计算得到新状态等；  
3. 调用组件实例生命周期方法，并且调用组件实例生命周期方法，并且调用更新器API更新fiber实例等，如更新组件实例调用的callComponentWillReceiveProps方法，更新fiber实例，并将更新添加至更新队列：  

	// 调用组件实例生命周期方法并调用更新器API
	function callComponentWillReceiveProps(
		workInProgress, instance, newProps, newContext
	) {
		const oldState = instance.state;
		instance.componentWillReceiveProps(newProps, newContext);
		
		if (instance.state != oldState) {
			// 调用更新器入队列方法
			updater.enqueueReplaceState(instance, instance.state, null);
		}
	}


另外需要重点关注的是`insertUpdateIntoFiber`方法，该方法实现将更新任务添加至组件fiber实例，内部会处理将任务添加至fiber更新队列，源码见上文更新队列中介绍的ReactFiberUpdateQueue模块，最终还是调用insertUpdateIntoQueue;  
获取FIBER实例  
获取FIBER实例比较简单，fiber实例通过ReactInstanceMap 模块提供的API进行维护：  

	export function get(key) {
		return key._reactInternalFiber;
	}
	
	export function set(key, value) {
		key._reactInternalFiber = value;
	}

使用节点上的`_reactInternalFiber` 属性维护fiber实例，调用`get`方法即可获取。  

获取优先级  
fiber 实例的优先级是由调度器控制，所以需要询问调度器关于当前fiber实例的优先级，调度器提供`computeExpirationForFiber` 获取特定fiber实例的优先级，即后去特定fiber实例的到期时间（expirationtime），方法具体实现见调度器与优先级章节。  

将更新任务添加至更新队列  
组件状态变更时，将对应的组件更新任务划分优先级并根据优先级从高到低一次推入fiber实例的更新队列，诸如使用 `setState`方法触发的更新任务通常是添加至更新对列尾部。  
调度器完成切分任务为任务单元后，将使用`performUnitOfWork` 方法开始处理任务单元，然后按调用组件的更新器（实现见上下文介绍）先关API，按优先级将任务单元添加至fiber实例的更新队列：  
1. 从work-in-progress的alternate属性获取当前稳定fiber，然后调用beginWork开始处理更新。

	// 处理任务单元
	function performUnitOfWork(workInProgress: Fiber): Fiber | null {
		// 当前最新版本fiber实例使用fiber的alternate属性获取  
		const current = workInProgress.alternate;
		// 开始处理，返回子组件fiber实例
		let next = beginWork(current, workInProgress, nextRenderExpirationTime);
		if (next === null) {
			// 不存在子级fiber，完成单元任务的处理，之后继续处理下一个任务  
			next = completeUnitOfWork(workInProgress);
		}
		return next;
	}


2. `beginwork` 返回传入fiber实例的子组件fiber实例，若为空，则代表此组件树任务处理完成，否则会在workloop方法内迭代调用`performUnitWork`方法处理:  

a. deadline: 是调用requrestIdleCallback API 执行任务处理函数时返回的帧时间对象；  
b. nextUnitOfWork： 下一个要处理的任务单元。  
c. shouldYield： 判断是否暂停当前任务处理过程；
  

	function workLoop(expirationTime) {  
 		// 渲染更新至DOM的到期时间值 小于调度开始至开始处理此fiber的时间段值。
 		// 说明任务已经过期  
 		if (nextRenderExpirationTime <= mostRecentCurrentTime) {  
 			// Flush all expired work, 处理所有已经到期的更新  
 			while (nextUnitOfWork !== null) {  
 				nextUnitOfWork = performUnitOfWork(nextUnitOfWork);  
 			}  
 		}  
 		else {  
 			// Flush asyncchronous work until the deadline runs out of time.   
 			// 依次处理异步更新，直至deadline到达    
 			while (nextUnitOfwork !== null && !shouldYield()) {   
 				nextUnitOfWork = performUnitOfWork(nextUnitOfWork);  
 			}  
 		}  
	}  
 	
 	// 处理异步任务时，调和器将询问渲染器是否暂停执行；  
 	// 在DOM中，使用requestIdleCallback API实现；  
 	function shouldYield() {  
 		if(deadline === null) {  
 			return false;  
 		}  
 		if (deadline.timeRemaining() > 1) {  
 			// 这一帧还有剩余时间，不需要暂停；  
 			// 只有非过期任务可以到达此判断条件  
 			return false;  
 		}  
 		deadlineDidExpire = true;  
		return true;  
	}  

3.`beginWork` 方法内根据组件类型调用不同方法，这些方法内调用更新器API将更新添加至更新队列， 具体实现见 ReactFiberBeginWork 模块


1.引入ReactFiberClassComponent更新器相关模块并初始化获得api；
2.beginwork 方法内根据传入的work-in-progress 的fiber类型调用不同逻辑处理；
3.在逻辑处理里面会调用更新api，将更新添加至更新队列；
4.以classComponent 为例，将调用updateClassComponent方法；  

a.判断若第一次则初始化挂载组件实例，否则调用updateClassInstance 方法更新组件实例；  
b.最后调用finishComponent 方法，调和处理其子组件并返回其子级fiber实例；


	function updateClassComponent(
		current, workInProgress, renderExpirationTime
	) {
		let shouldUpdate;
		if (current == null) {
			if (!workInProgress.stateNode) {
				// fiber 没有组件实例时需要初始化组件实例
				constructClassInstance(workInprogress, workInProgress.pendingProps);
				// 挂载组件实例
				mountClassInstance(workInProgress, renderExpirationTime);
				// 默认需要更新
				shouldUpdate = true;
			}
		}
		else {
			// 处理实例更新并返回是否需要更新组件  
			shouldUpdate = updateClassInstance(current, workInprogress, renderExpirationTime)
		}
		
		// 更新完成后，返回子组件fiber实例
		return finishClassComponent(current, workInprogress, shouldUpdate, hasCOntext)
	}
	
	function finishClassComponent(
		current, workInprogress, shouldUpdate, hasContext
	) {
		if (!shouldUpdate) {
			// 明确设置不需要更新时，不处理更新，
			// 如shouldComponentUpdate 方法return false
			return bailoutOnAlreadyFinishedWork(current, workInProgress);
		}
	}
	
	const instance = workInProgress.stateNode;
	// 重新渲染
	ReactCurrentOwner.current = workInProgress;
	// 返回组件子组件树等内容  
	let nextChildren = instance.render();
	// 调和子组件树，将迭代处理每一个组件
	// 函数内将调用ReactChildFiber模块提供的api
	reconcileChildren(current, workInProgress, nextChildren);
	// 返回子组件fiber实例
	return workInProgress.child;
	}
	
调度更新任务  
上一节更新器已经能按照优先级将更新添加至更新队列，那么如何调度执行更新任务呢？  
在更新器实现ReactFiberClassComponent模块中，在enqueueSetState，enqueueReplaceState和enqueueForceUpdate 入对列方法中，均会调用如下方法： 
		
	insertUpdateIntoFiber(fiber, update);
	scheduleWork(fiber, expirationTime);
		
1. insertUpdateIntoFiber: 将更新添加至fiber实例，最终会添加至更新对列；
2. scheduleWork: 调度任务，传入fiber实例和任务到期时间

#####渲染与调和
在调和阶段，不涉及任何DOM处理，在处理完更新后，需要渲染模块将更新渲染至DOM，这也是React应用中虚拟DOM的概念。即所有的更新计算都基于虚拟DOM，计算完后才将优化后的更新渲染至真实DOM。Fiber使用requestIdleCallback，API更高效的执行渲染更新的任务，实现任务的切分。  

#####源码简单分析
React-DOM 渲染模块  
在项目中，如果要将应用渲染至页面，通常会有如下代码：  
	
	import ReactDOM from 'react-dom';
	import App from './App'; // 应用根组件
	
	ReactDOM.render(
		<App />,
		ele // 挂载到容器
	)
	
react-dom 模块就是适用于浏览器端渲染react应用的渲染方案，reactdom源码如下：  
	
	const ReactDOM = {
		render(
			element: React$Element<any>, // react 元素，通常是项目根组件
			container: DOMContainer, // React应用挂载的DOM容器
			callback: ?Function, // 回调函数
		) {
			return renderSubtreeIntoContainer(
				null,
				element,
				container,
				false,
				callback
			);
		}
	}

常用的渲染组件至DOM的render方法如上，调用renderSubtreeIntoContainer方法，渲染组件的子组件树：  
	
	// 渲染组件的子组件树至父容器
	function renderSubtreeIntoContainer(
		parentComponent: ?React$Component<any, any>,
		children: ReactNodeList,
		container: DOMContainer,
		forceHydrate: boolean,
		callback: ?Function, 
	) {
		let root = container._reactRootContainer;
		if (!root) {
			// 初次渲染时初始化
			// 创建react根容器
			const newRoot = DOMReact.createContainer(container, shouldHydrate);
			// 缓存react根容器至DOM容器的reactRootContainer属性
			root = container._reactRootContainer = newRoot;
			// 初始化容器相关
			DOMRenderer.unbatchedUpdates(() => {
				DOMRenderer.updateContainer(children, newRoot, parentComponent, callback)
			});
		}
		else {
			// 如果不是初次渲染则直接更新容器
			DOMRenderer.updateContainer(children, root, parentComponent, callback);
		}
		
		return DOMRenderer.getPublicRootInstance(root);
	}

DOM 渲染器对象  
DOMRenderer 是调用调和算法返回的DOM渲染器对象，在此处会传入渲染模块的渲染UI操作API，如：  
	
	// 调用调和算法方法
	const DOMRenderer = ReactFiberReconciler(
		// 传递至调和算法中的渲染ui （react-dom 模块即DOM）
		// 实际操作API
		{
			getPublicInstance(instance) {
				return instance;
			},
			createInstance(
				type: string,
				props: Props,
				rootContainerInstance: Container,
				hostContext: HostContext,
				inernalInstanceHandle: Object,
			) {
				// 创建DOM元素
				const domElement = createElement(
					type,
					props,
					rootContainerInstance,
					parentNamespace,
				);
				precacheFiberNode(internalInstanceHandle, domElement);
				updateFiberProps(domElement, props);
				return domElement;
			},
			now: ReactDOMFrameScheduling.now,
			mutation: {
				// 提交渲染
				commitMount(
					domElement: Instance,
					type: string,
					newProps: Props,
					internalInstanceHandle: Object,
				) {
					((domElement: any)):
					| HTMLButtonElement
					| HTMLInputElement
					| HTMLSelectElement
					| HTMLTextAreaElement).focus();
				}
			},
			// 提交更新
			commitUpdate(
				domElement: Instance,
				updatePayload: Array<mixed>,
				type: string,
				oldProps: Props,
				newProps: Props,
				internalInstanceHandle: Object,
			) {
				// 更新属性
				updateFiberProps(domElement, newProps);
				// 对DOM节点进行diff算法分析
				updateProperties(domElement, updatePayload, type, oldProps, newProps);
			},
			// 清空文本内容
			resetTextContent(domElement: Instance):void {
				domElement.textContent = '';
			},
			// 添加子级
			appendChild(
				parentInstance: instance,
				child: Instance | TextInstance,
			): void {
				parentInstance.appendChild(child);
			}
		}
	)


在任务完成时将执行`createInstance` 方法，然后调用`createElement` 创建DOM元素并添加至文档。  

调和算法入口：  
	
	import ReactFiberScheduler from './ReactFiberScheduler';
	import {insertUpdateIntoFiber} from './ReactFiberUpdateQueue';
	
	export default function Reconciler(
		// 下文用到的config参数即从此处传入
		getPublicInstance,
		createInstance,
		...
	) {
		// 生成调度器API
		var {
			computeAsyncExpiration, computeExpirationForFiber, scheduleWork,
			batchedUpdates, unbatchedUpdates, flushSync, deferredUpdates,
		} = ReactFiberScheduler(config);
		
		return {
			// 创建容器
			createContainer(containerInfo, hydrate: boolean) {
				// 创建根fiber实例
				return createFiberRoot(containerInfo, hydrate);
			},
			// 更新容器
			updateContainer(
				element: ReactNodeList,
				container: OpaqueRoot,
				parentComponent: ?React$Component<any, any>,
				callback: ?Function,
			): void {
				const current = container.current;
				...
				// 更新
				scheduleTopLevelUpdate(current, element, callback);
			},
			...
			// 获取容器fiber树的根fiber实例
			getPublicRootInstace (container) {
				// 获取fiber实例
				const containerFiber = container.current;
				if (!containerFiber.child) {
					return null;
				}
				switch (containerFiber.child.tag) {
					case HostComponent: 
						return getPublicInstance(containerFiber.child.stateNode);
					default: 
						return containerFiber.child.stateNode;
				}
			},
			unbatchedUpdates
		};
	}

在 `react-dom` 渲染模块调用 `createContainer` 创建容器和根fiber实例，FiberRoot对象，调用updateContainer方法更新容器内容。  

开始更新
	
	// 更新
	function scheduleTopLevelUpdate(
		current: Fiber,
		element: ReactNodeList,
		callback: ?function,
	) {
		callback = callback === 'undefined' ? null : callback;
		const update = {
			expirationTime,
			partialState: {element},
			callback,
			isReplace: false,
			isForced: false,
			nextCallback: null, 
		};
		// 更新fiber实例
		insertUpdateIntoFiber(current, update);
		// 执行任务
		scheduleWork(current, expirationTime);
	}
	
处理更新  
调用scheduleWork 方法处理更新任务，实现见上文。  

提交更新  
处理完成更新后需要确认提交更新至渲染模块，然后渲染模块才才能将更新渲染至DOM。  
	
	import ReactFiberCommmitWork from './ReactFiberCommitWork';
	
	const {
		commitResetTextContent,
		commitPlacement,
		commitDeletion,
		commitWork,
		commitLifeCycles,
		commitAttachRef,
		commitDetachRef,
	} = ReactFiberCommitWork(config, captureError);
	
	
	function commitRoot(finishedWork) {
		...
		commitAllHostEffects();
	}
	// 循环执行提交更新
	function commitAllHostEffects() {
		while (nextEffect !== null) {
			let primaryEffectTag = effectTag & ~(Callback | Err | ContentReset | Ref | PerfomedWork);
			switch (primaryEffectTag) {
				case Placement: {
					commitPlacement(nextEffect);
					nextEffect.effectTag &= ~Placement;
					break;
				}
				case PlacementAndUpdate： {
					// placement
					commitPlacement(nextEffect);
					nextEffect.effectTag &= ~Placement;
					// update
					const current = nextEffect.alternate;
					commitWork(current, nextEffect);
					break;
				}
				case Update: {
					const current = nextEffect.alternate;
					commitWork(current, nextEffect);
					break;
				}
				case Deletion: {
					isUnmounting = true;
					commitDeletion(nextEffect);
					isUnmounting = false;
					break;
				}
			}
			nextEffect = nextEffect.nextEffect;
		}
	}
	
	// Flush sync work.
	let finishedWork = root.finishedWork;
	if (finishedWork !== null) {
		// this root is already complete, we can commit it
		root.finishedWork = null;
		root.remainingExpirationTime = commitRoot(finisedWork);
	}
	
提交更新是最后确认更新组件的阶段，主要逻辑如下：

	export default function (mutation, ...) {
		const {
			commitMount,
			commitUpdate,
			resetTextContent,
			commitTextUpdate,
			appendChild,
			appendChildToContainer,
			insertBefore,
			insertInContainerBefore,
			removeChild,
			removeChildFromContainer,
		} = mutaion;
		
		function commitWork(current: Fiber | null, finishedWork: Fiber): void {
			switch (finishedWork.tag) {
				case ClassComponent: {
					return;
				}
				case HostComponent: {
					const instance: I = finishedWork.stateNode;
					if (instance != null) {
						// commit the work prepared earlier
						const newProps = finishedWork.memoizedProps;
						// for hydration we reuse the update path but we treat the oldProps
						const oldProps = current !== null ? current.memoizedProps : newProps;
						const type = finishedWork.type;
						const updatePayload = finishedWork.updateQueue:;
						finishedWork.updateQueue = null;
						if (updatePayload !== null) {
							commitUpdate(
								instance,
								updatePayload,
								type,
								oldProps,
								newProps,
								finishedWork
							);
						}
					}
					return;
				}
				case HostText: {
					const textInstance = finishedWork.stateNode;
					const newText = finishedWork.memoizedProps;
					
					const oldText: string = current !== null ? current.memoizeProps:newText;
					commitTextUpdate(textInstance, oldText, newText);
					return;
				}
				case HostRoot: {
					return;
				}
				default: {}
			}
		}
	}
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	