###requestIdleCallback 后台任务调度

####前言  
在学js的最初，我们就已经了解到js是单线程的，它只有执行完一段代码后，才能执行另外的代码，在平时，这其实并不会受到影响，但是当你需要一些高频的操作时呢，比如你使用js来完成一段动画，监听input 的输入来频繁的操作DOM，scroll的滚动监听等，这个时候，我们多么希望，把这些计算量特别的功能，直接另开线程去处理。 
 
####1. Worker  
说到多线程，可能你能想到worker，是的，worker是一个多线程的功能，但worker有个很大的限制，就是worker只能进行一些单纯的js计算，不能牵扯DOM，而在js很多动态效果中，有特别多的地方，是进行DOM操作的，DOM操作才是最消耗性能的地方，这个时候worker就显得有点鸡肋，所以虽然workder方法已经出来多年，依然不火的原因。  

####2. requestIdeCallback  
在很久以来，前端开发者们，都希望可以通过一种方法，来了解到，当前的事件处理队列的情况，是否浏览器处于空闲状态，还是正在处理一个很复杂计算量很大的代码，比如：数据上报，数据分析，客户端模板渲染，数据预加载等，这些操作，更适合在浏览器空闲的时候进行处理，也就是一个概念中的后台处理程序。  
如果当前正在处理很复杂的逻辑，而又要处理上述的逻辑的话，那么事件处理队列将变得异常庞大，这个时候，浏览器甚至会出现假死的状态，而在以往的时候，当我们猜测某些操作，会出现这种情况的时候，通常使用setTimeout来做一个延时的处理，甚至把一个很复杂的逻辑拆分为多个模块，然后setTimeout分片执行，虽然可以缓解但是还不能彻底解决，因为开发者不知道每段程序会执行多长时间。而用户随时会操作浏览器，导致无法高效的利用浏览器的性能。  

requestIdleCallback 2016年是在chrome49版本中新兴的API。  2017年年初，w3c也有了该api的草案：requestIdleCallback后台任务调度。  

它是一个后台任务，只有在浏览器空闲的时候才会被执行，也就是说，如果现在正在执行一个requestAnimationFrame的动画，而这个动画消耗了特别的浏览器性能，那么在动画接卸气，requestIdleCallback才会被执行，而对于一些性能不好的浏览器，requestAnimationFrame动画，可能占据了特别多的内存，那么这个时候，requestIdleCallback就可能在整个动画执行完毕之后，才会被执行，这样至少对于动画来说，会有更好的体验，而不会因为中间加入其它操作，而导致动画不流畅。

而动画结束后，浏览器的事件队列中，没有其他的操作时，这个时候再来执行requestIdleCallback 的方法。甚至为更精准的提供这些信息，requestIdleCallback 的回调函数中，会传入一个deadline对象，该对象中有方法，可以对事件执行的事件消耗进行一个预估。这样前端开发就可以通过deadline对象来预估这个回调函数，大概需要消耗的时间，进而做出正确的操作，保证不会影响到其他的动画等体验性。  
看看requestIdle的兼容性如何  
![兼容](http://www.zhangyunling.com/study/2017/20170114/caniuse.png)  

接下来让我们来学习下requestIdleCallback 更详细的用法。 
#####2.1 默认简单的使用  

	let requestId = requestIdleCallback(cb);
	

这里的requestId与setTimeout 的返回值一样，是一个标识，如果在之后，希望清理回调，可以直接使用cancelIdleCallback(requestId) 即可，关于这一点，与计时器是完全相同。  
&ensp; 前面说过requestIdle 回调函数中会传入一个默认的deadline对象，它是IdleDeadline构造函数的一个实例，该构造函数，只支持两个属性。  
	
	didTimeout: Boolean // 是否超时触发，只读
	timeRemaining: function // 该帧剩余可用时间 

其中包括一个属性didTimeout,和一个方法timeRemaining.  
deadline就是这样一个基于requestIdCallback 的IdleDeadline的实例，默认情况下，它也只包含这两个可用的数据。  
##### 3.2 didTimeout  
说到这个属性的话，就不得不提一下requestIdleCallback 的第二个参数，他是一个可配置的对象，只支持一个参数timeout，如果一帧内，一直没有空闲的时间可以执行requestIdleCallback的回调函数的话，那么当达到timeout设置的超时时间，requestIdleCallback的表现就和setTimeout的表现一致了。

下面来看一下例子：  

	//cb这里不加了，细节在前面的示例链接中
	function rIC(){
	    requestId = requestIdleCallback(cb,{
	        timeout : 100
	    });
	}
	
	function _cb1(){
	    _reset();
	
	    var i = 0,
	        len = 5;
	
	    function _s(){
	
	        if(i < len){
	            add("i="+i);
	            rAF(_s);
	            i++;
	        }
	    }
	
	    //requestIdleCallback的调用
	    rIC();
	
	    //rAF(_s);
	    _s();
	    //这里的两种调用方式，会对rIC有什么区别呢？
	    //在本篇文章的最后，会进行一下说明。
	
	}
	
	
	function _cb2(){
	    _reset();
	
	    var i = 0,
	        len = 10000;
	
	    rIC();
	
	    add("循环了10000次，1000的倍数才会显示！");
	    for(i;i<len;i++){
	        if(i%1000 == 0){
	            add("i="+i);
	        }
	
	        //用来阻塞时间
	        console.log("i="+i);
	    }
	
	}

那么timeout 是必须要设置的吗，这就要看具体情况了，如果这个方法，必须要在某段时间内执行，那么可以设置timeout，如果没有这个必要，那么可以完全不用管这个参数的，直接简单的调用即可。  

3.3timeRemaining  
在回调函数传入的参数deadline对象中，唯一的一个方法是： timeRemaining(),它是用来获取当前一帧还有多长时间结束的。  
如何理解这个呢，先来看一张图，该图时取自w3c官方文档。
  
![](http://www.zhangyunling.com/study/2017/20170114/image01.png)  

该图中的frame#1，frame#2就是两帧，每个帧持续时间是 1000 / 60 = 16.666ms,而在每一帧的内部，TASK和rendering只花费了一部分时间，并没有占据整个帧，那么这个时候，如图中idle period 的部分就是空闲时间，而每个帧中的空闲时间，根据该帧中处理事情的多少，复杂程度等，消耗不等，所以空闲时间也不等。     
而对于每一个headline.timeRemaining()返回值，就是如图中，IdleCallback 到所在帧结尾的时间（ms级）。  
对于该属性，在前面的实例中，就有用到了，所以这第三次实战，就继续使用前面的示例吧。  
前面的示例，考虑的情况并不多，如果帧中处理的东西特别多呢？比如使用rAF(requestAnimationFrame) 处理，但是其每次处理，都消耗特别多，无法在一帧内执行呢？  
所以这里来看如下示例：  
不管如何说，都不是实战来说的明白，所以继续实战： [timeRemaining](http://www.zhangyunling.com/study/2017/20170114/index-pc-4.html)  
在第四次实战中，rIC回调，都是计算量特别大的回调，但是得到的效果却截然不同。  
不做任何处理，在一次回调中处理，结果直接阻塞掉原有的rAF的处理，这样就会导致rAF的动画出现卡顿的情况，如果这样的话，那么rIC就变得有些鸡肋了。  
从实例运行结果可以看出，rIC的一个回调处理，直接就消耗了305毫秒的时间，对于rAF的循环处理模块形成了致命的阻塞，如果rAF是动画的话，那么在这个时候，动画就会卡住305ms，用户会比较明显的感知。  
那么如何避免上述问题呢，这就要靠本节中的timeRemaining方法了，它可以获取每一帧的剩余时间，而我们的这些处理，只需要在剩余空闲时间处理就可以了，不需要实时性的处理，甚至，如果每帧的空余时间短，我都接受多占用几个帧的空余时间，分段来处理。  

由运行结果可以看出，在rIC消耗过多的时候，rAF依然可以正常执行，rIC只有在rAF的空隙中处理事情。  
3.3 cancelIdleCallback  
既然可以设置一个requestIdleCallback 的回调，那么也可以取消掉一个，它就是对应requestIdleCallback的cancelIdleCallback方法，其调用方法特别简单，与setTimeout完全相同。  
3.4 备注事项  
1. timeRemaining 有一个特性需要注意一下，那就是它获取到的值，最大不会超过50ms，也就是说，就算没有使用rAF的循环模式，在代码中，没有任何这样的循环的话，timeRemaining的取值，也不会大于50ms，这个可以自己做一个简单的实例。这是一个w3c的标准。  
![](http://www.zhangyunling.com/study/2017/20170114/image00.png)

2. requestIdleCallback 和requestAnimationFrame 执行有一个相同的特性，不管当前帧，是否有空闲时间，它的最早执行时间，都是在下一帧开始。  
3. 





























	
	















