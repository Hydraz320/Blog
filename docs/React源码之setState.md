# React源码之setState

在我们对组件的状态进行修改时几乎一定会用到的就是setState方法，而较为常见的使用方式就是传入一个对象。从官网的文档中可以看到：这种形式的setState()是异步的，在同一个循环(cycle)中调用多次会被batch到一起，并且在一个cycle中调用多次increment最终却只会进行一次。那么，setState方法究竟在框架内部做了哪些工作导致了文档中所描述的情况，我们又可从中学习到哪些更好的实践呢？

## 代码调试

这里先通过打debugger和阅读源码，给出在一个handleClick里调用setState后代码的流动情况来理解这个过程。
首先，当触发onClick事件时，会来到ReactEventListener.js文件中(这里先略过内部的事件机制)，由dispatchEvent派发事件，其中有如下代码：

![dispatchEvent](https://github.com/Hydraz320/Blog/blob/master/images/React%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E7%B3%BB%E5%88%97/3.png?raw=true)

可以看到，React在事件派发的时候调用了ReacUpdates.batchedUpdates方法，而这一方法来自ReactDefaultBatchingStrategy. batchedUpdates，

![batchedUpdates](https://github.com/Hydraz320/Blog/blob/master/images/React%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E7%B3%BB%E5%88%97/4.png?raw=true)

这一方法算是比较核心的一个方法，甚至在通常所说的“生命周期”的钩子方法执行前也会先执行batchedUpdates(这里在后面的例子分析)。对于当前的情形，因为尚未处于批量更新的状态，alreadyBatchingUpdates这一bool变量为false，于是会走向事务transaction.perform。因此，在经历一系列事件处理后，调用栈进入到ReactComponent.prototype.setState进行状态的改变。

但是有一点非常重要的是，尽管alreadyBatchingUpdates为false，但isBatchingUpdates却被修改为true，而这会影响到后续的执行流程(作为dirtyComponents处理)。

setState接受了一个partialState参数，代表了部分发生改变的状态。在setState内部则是会调用enqueueSetState方法：

![enqueueSetState](https://github.com/Hydraz320/Blog/blob/master/images/React%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E7%B3%BB%E5%88%97/5.png?raw=true)

其中，updater是被注入进来的，先不去深究，而enqueueSetState方法在ReactUpdateQueue.js文件中，内部调用ReactUpdates.enqueueUpdate，再次进入函数阅读，就可以看到核心的一段代码：

![](https://github.com/Hydraz320/Blog/blob/master/images/React%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E7%B3%BB%E5%88%97/6.png?raw=true)

若isBatchingUpdates为 true，则把当前状态等待更新的组件(即调用setState的组件，类型其实是ReactCompositeComponentWrapper，是自定义组件类的一个包装，其实就是通过_assign方法在原型上添加了一个方法)放入 dirtyComponents 数组中；否则会调用batchUpdates处理所有队列中的更新，甚至我们可以看到，该方法会在内部继续调用enqueueUpdate。也就是说，尽管使用时就是用一个setState，但实际更新的情况却会根据是否处于“批量更新”的状态来决定是先“批量更新”还是push进dirtyComponents等待后续的处理。

如果batchedUpdates，那么就是直接开始去更新了，这里会push到dirtyComponents，然后事务触发close(本例中是closeAll，但其实也是close…)才取更新视图。

![](https://github.com/Hydraz320/Blog/blob/master/images/React%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E7%B3%BB%E5%88%97/7.png?raw=true)

![](https://github.com/Hydraz320/Blog/blob/master/images/React%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E7%B3%BB%E5%88%97/8.png?raw=true)

上面红框的两个事务非常多见，在进入closeAll之后会进入到flushBatchedUpdates，可以从下图看到，函数在循环中将dirtyComponents中的自定义组件wrapper挨个取出，并执行了关键一步，是的又是事务，并在这一步完成了视图的更新。之后就是一些释放工作了。

![](https://github.com/Hydraz320/Blog/blob/master/images/React%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E7%B3%BB%E5%88%97/9.png?raw=true)

然而，还需要继续进入runBatchedUpdates，这一函数我就不截图了，它首先对要更新的dirtyComponent进行了排序，比如说有时更新了外层组件也会影响到内部组件因而需要先更新内部组件；然后继续调用ReactReconciler.performUpdateIfNecessary，但其实内部是这样的：

![](https://github.com/Hydraz320/Blog/blob/master/images/React%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E7%B3%BB%E5%88%97/10.png?raw=true)

internalInstance，看到现在应该猜得到是ReactCompositeComponent(Wrapper)，果然，ReactCompositeComponent早就定义了performUpdateIfNecessary方法，此处会由updateComponent方法来进行组件更新。可以看到，从onClick事件开始到现在，已经终于来到ReactCompositeComponent这一自定义组件的地盘。

![](https://github.com/Hydraz320/Blog/blob/master/images/React%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E7%B3%BB%E5%88%97/11.png?raw=true)

updateComponent函数还挺长的，这里介绍其主要思想(但其实是最终更新的核心)。首先提供了componentWillReceiveProps钩子，并通过this._processPendingState计算了新的状态，然后提供了shouldComponentUpdate钩子，并且会在未提供的情况下做shallowEqual比较。最终由_performComponentUpdate执行更新。
其实到这里我还是蛮崩溃的，想不到到这都还没更新？！_performComponentUpdate中提供了componentWillUpdate钩子(其实看到这里我终于开始明白，所谓的生命周期，不过是在被设计好的更新、挂载的调用栈上给出那么几个钩子让开发者去定义去使用而已，有啥玄乎啊……)之后从参数列表拿到先前计算好的nextState(这里其实就是setState的那个对象参数了)，然后再去调用_updateRenderedComponent。

在函数内部出现的几个变量需要说明的是，_renderedComponent是ReactDOMComponent类型的，什么意思呢？可以说，这是React加工出来的“伪”DOM元素，它拥有变成真实DOM所需的标签、子元素、设置的事件回调等等。

继续调用ReactReconciler.receiveComponent之后，又会跳到ReactDOMComponent. receiveComponent(发现这个套路没……)然后跳到ReactDOMComponent. updateComponent，可以看到下图中，先更新了属性，然后更新子DOM，也就是说更新本身是个递归过程，

![](https://github.com/Hydraz320/Blog/blob/master/images/React%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E7%B3%BB%E5%88%97/12.png?raw=true)

然后执行进去后，在这一句发生了变化：

![](https://github.com/Hydraz320/Blog/blob/master/images/React%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E7%B3%BB%E5%88%97/13.png?raw=true)

而实现的过程就是由调和器ReactChildReconciler.unmountChildren去掉子元素后重新挂载新元素导致的。

事实上，底层涉及到的操作都和这几个事务有关，在下一个例子中会看到它们是如何影响setState的行为的。


## 实例分析
这个例子是从别人的分析中看来的，以此为基础也加深了我的理解和思考。这里分析如下(图片有点长。。。)：

![](https://github.com/Hydraz320/Blog/blob/master/images/React%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E7%B3%BB%E5%88%97/14.png?raw=true)

上面代码的结果是0 0 1 1 3 4。总的来说这个意思就是： 在ComponentWillMount、ComponentDidMount里，会先触发事务进入batchedUpdates执行状态，状态的更新都会被push进dirtyComponent并且只执行一次。这里划掉是因为这个理解不准确。正如前面所提到的文档描述，所谓的只执行一次，是因为每一个setState都没有立即更新，所以在同一个cycle里，this.state.val都是一样的，而this.state.val+1也是如此。这样一来，连续几个setState只是在反复令val等于同一个值而已，并且push到dirtyComponents里没有立即执行。所以，每一个setState都执行了，而并非文档中所说的“只执行一次increment”。而在setTimeout里，调用栈则简单得多，最终会立即更新，而打印出的值也是正常的。

结合平时的使用，我们可以尽量避免在同一个cycle中多次使用setState，同时，对于需要使用前一个状态的情形，使用setState的updater function form，如下所示：

![](https://github.com/Hydraz320/Blog/blob/master/images/React%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E7%B3%BB%E5%88%97/15.png?raw=true)

## 最后
尽管在对源码的理解上缺乏深度，还停留在发现是这样而不能理解这样设计的道理的层次上，但是我仍会坚持下去，希望在以后的学习中会有更深刻的体会和更显著的提升。
此外，这篇文章我会保留，当我有更好的体会时，做出更好的修改。
