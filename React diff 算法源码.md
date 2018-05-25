###React Diff 源码剖析
![](https://pic1.zhimg.com/74a86fbcc8bb4ad74e19b72a72b26c56_1200x500.jpg)

React 以迅雷不及掩耳之势占据了前端市场，本文通过剖析React源码，理解其内部的实现原理，知其然更要知其所以然。  
React diff 作为 Virtual DOM 的加速器，其算法上的改进优化是React整个界面渲染的基础，以及性能提高的保障，同时也是React源码中最神秘，不可思议的部分，本文从源码入手，深入解析React diff的奥秘。  

####前言
React 中最值得称道的部分莫过于Virtual DOM 与 diff 的完美结合，特别是其高效的diff 算法， 让用户可以无需估计性能问题而‘任性自由’的刷新页面，让开发者也可以无需关心VirtualDOM 背后的运作原理，因为React Diff 会帮助我们计算Virtual DOM 中真正变化的部分，并只针对该部分进行实际DOM操作，而非重新渲染整个页面，从而保证了每次操作更新后页面的高效渲染，因为virtual DOM 与Diff 是保证React 性能口碑的关键。  

传统diff 算法  
计算一颗树形结构转换成另一颗树形结构的最少操作，是一个复杂且值得研究的问题。传统diff算法通过循环递归对节点进行一次对比，效率极低，算法复杂度达到了O(n^3), 其中n是树中节点的总数。很可怕的一个计算数量。因此React需要制定一个高效稳定的diff算法。  
那么，React Diff是如何实现的呢？  

#### 详解React Diff  
传统diff算法的复杂度为 O(n^3), 显然这是无法满足性能要求的。React 大胆制定策略， 将O(n^3)复杂度的问题转换成O(n) 复杂度的问题。 
 
**Diff 策略**  
1. Web UI 中DOM节点跨层级的移动操作特别少，可以忽略不计。 （tree diff）  
2. 拥有相同类似的两个组件将会生成相似的属性结构，拥有不同类的两个组件将会生成不同的树形结构。（component diff）  
3. 对弈同一个 层级的一组子节点，他们可以通过唯一id进行区分(element diff)  

基于以上三个前提策略， React 分别对tree Diff、 Component Diff、以及element Diff进行算法优化，事实也证明这个前提策略合理切准确。  

####tree diff   
React 针对这一策略，通过 updateDepth 对virtual DOm 树进行层级控制，只会对相同颜色方框内的DOM节点进行比较，即同一个父节点下的所有子节点。当发现节点已经不存在，则该节点及子节点会被完全删除掉，不会再进一步比较。这样只需要对树进行一次便利，便能完成整个DOM树的比较。  
![dom diff](https://pic3.zhimg.com/80/0c08dbb6b1e0745780de4d208ad51d34_hd.jpg)
  
	updateChildren: function (nextNestedChildrenElments, transaction, context) {
		updateDepth++;
		var errorThrown = true;
		try {
			this._updateChildren(nextNestedChildrenElements, transaction, context);
			errorThrown = false;
	}
	finally {
		updateDepth--;
		if (!updateDepth) {
			if (errorThrown) {
				clearQueue();
			}
			else {
				processQueue();
			};
		};
	}
























