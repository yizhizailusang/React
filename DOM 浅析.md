### DOM 操作慢之刨根问底篇  
最近在学习React，看到好多都在说dom操作消耗性能比较慢，要尽量少的操作DOM，但心里却不太明白，于是想进一步刨根问底一下，在网上查阅了些资料，因为比较DOM操作和渲染部分是由浏览器来处理的，比较抽象，理解不到位的地方还请指出~  

找到问题答案之前先来温习一下整个浏览器渲染网页的过程。大致如下：  
1. 解析HTML，并生成一颗DOM Tree  
2. 解析各种样式并结合DOM tree生成一颗Render Tree  
3. 对Render Tree 的各个节点计算布局信息，比如box的位置与尺寸（layout）  
4. 根据Render Tree 并利用浏览器的UI层进行绘制（paint）  

其中DOM Tree 和Render Tree 上的节点并非一一对应，比如一个display: none 的节点就只会在DOM Tree上，而不会出现在Render Tree上，因为这个节点不需要被绘制。  
![渲染过程图](https://images2015.cnblogs.com/news/1/201602/1-20160219104627050-1270673052.jpg)
paint 和 layout 是一个耗时的过程  
以下三种情况，会导致网页重新渲染。  
1. 修改DOM  
2. 修改样式表  
3. 用户事件（比如鼠标悬停、页面滚动、输入框键入文字、改变窗口大小）

重新渲染就需要重新布局和重新绘制。前者叫重排，后者叫重绘。 
需要注意的是重绘不一定需要重排，重排一定引起重绘。  

#### 性能优化 
重新渲染时非常消耗资源的，是导致网页性能低下的根本原因。
提高网页性能，就要降低重排和重绘的频率和成本，尽量较少触发重新渲染，现在浏览器已经很智能了，会尽量把所有的变动集中在一起，排成一个对列，在当前js的执行上下文完成后执行一次渲染，尽量避免多次渲染。
		
		div.style.color = 'blue';
		div.style.marginTop = '30px';
		
上面代码两个样式改动，但是浏览器只会触发一次重排和重绘。  
如果写的bad，就会触发两次重排和重绘。  

	div.style.color = 'blue';
	var margin = parseInt(div.style.marginTop);
	div.style.marginTop = (margin + 10) + 'px';
	
上面代码对div元素设置背景色后，第二行要求浏览器给出该元素的位置，所以浏览器不得不立即重排。  
所以从性能角度考虑，尽量不要把读操作和写操作，放在一个语句里面。  

#####不仅要减少操作DOM，还要减少去访问DOM的次数。  
在浏览器中，DOM和js的实现，并不在同一个区域，比如我们最熟悉的chrome，js引擎是V8，而DOM和渲染，靠的是Webcore引擎。也就是说，DOM和js是两个独立个体。  

	把DOM和JavaScript各自想象成一个岛屿，它们之间用收费桥梁连接。--《高性能JavaScript》  

#####提高性能的技巧  
1. DOM的多个操作应该放在一起。  
2. 如果某个样式是通过重排得到的，那么最好缓存结果。避免下次用到时候，浏览器又要重排。  
3. 不要一条条改变样式，而要通过改变class，或者csstext属性，一次性的改变样式。  
4. 尽量使用离线dom，而不是真实的网页DOM，来改变样式，比如Document Fragment。再比如，使用 cloneNode() 方法，在克隆的节点上进行操作，然后再用克隆的节点替换原始节点。  
5. position属性为absolute或fixed的元素，重排的开销会比较小，因为不用考虑它对其他元素的影响。  
6. 只在必要的时候，才将元素的display属性为可见，因为不可见的元素不影响重排和重绘。另外，visibility : hidden的元素只对重绘有影响，不影响重排。  
7. 使用虚拟DOM的脚本库，比如React等
8. 使用 window.requestAnimationFrame()、window.requestIdleCallback() 这两个方法调节重新渲染

