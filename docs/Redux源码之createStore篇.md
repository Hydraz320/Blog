# 开坑

最近想看看Redux，毕竟不大有那么多相关文章，加深下理解吧。

Redux+Saga+Immutable好舒服。

## utils/isPlainObject.js

这个方法就是检测是不是一个普通对象，换句话说是不是用new Object()或者{}生成的。检测方法就是用Object.getPrototypeOf方法，这里涉及到继承原理方面的东西，可以看这篇[总结](https://github.com/Hydraz320/FE.Note/blob/master/JS/%E7%BB%A7%E6%89%BF/oop-prototype.md)，下面的图讲得很清楚了。蛮有趣的一段代码，先获取对象的指针，然后循环去找该对象的__proto__直到到达null，这种情况下plain object必然还是指向Object.prototype，而非plain object的__proto__就可能指向某个类的原型了。

## createStore.js

源码注释真的好看，对各个概念的解释不要太精准且容易理解。

> createStore文件代码注释翻译
>
> 创建一个维护state tree的redux store。
> 唯一改变store中数据的方法就是调用`dispatch()`。
>
> 你的应用应当只有一个单一的store。为了指定state tree的不同部分如何与actions进行对应，你可以利用`combineReducers`来combine若干个reducers以生成一个单一的reducer function。
>
> @param {Function} reducer 是一个返回下一个state tree的函数，它会根据当前state tree和action来进行计算和处理。
>
> @param {any} [preloadedState] 初始state。你可以选择在一个同构应用中从服务端整合一个状态，或者从先前存储在用户会话(session)中的序列化的数据中取出。
>
> 如果你使用`combineReducers`来创建一个根reducer function，那么这个初始state必须与`combineReducers`创建的对象的key值一一对应。 
>
> @param {Function} [enhancer] store enhancer。 你可以使用如中间件、时间旅行、持久化等第三方功能来增强store。Redux自带的store enhancer只有`applyMiddleware()`。
>
> @returns {Store} 一个让你可以读取状态的、触发action的、订阅变化的redux store对象。

下面可以根据几个具体问题来引导着我们看代码。

### 问题一：为什么createStore可以只接收一个reducer函数作为参数，最后仍能够返回一个store对象呢？createStore最终返回的store对象是什么样子的呢？

简要回答：因为createStore内部会触发INIT action，然后会调用dispatch并在内部使用reducer进行更新，因为INIT action的type的值为`@@redux/INIT${randomString()}`，显然是个随机值，所以调用的reducer内部，用户对action.type的判断就几乎不可能匹配上，也即走了default路线，返回了reducer默认的initState。这么一来即使只传入reducer而没提供第二个初始值参数，依旧可以完成初始化。当然，如果给createStore提供了初始值preloadedState，在内部会赋给currentState，而currentState又会作为currentReducer函数的参数，而currentReducer也就是由reducer赋值而来。这样reducer就得到了初始值，完成后续的初始化流程。

createStore返回的store对象，也就是维护着一个初始state、同时具有5个API(dispatch、subscribe、getState、replaceReducer、[$$observable])的对象，通过这些API我们可以对state进行管理。其中，前三个自然是文档上见得很多的啦。后两个后面我们会研究，其中replaceReducer在我们内部项目中有所应用。

### 问题二：我们能否在reducer执行过程中再dispatch一个action，这样仿佛很任性？

简要回答：内部通过一个状态变量isDispatching作为锁来限制这一行为，当action被dispatch后，我们会调用reducer进行处理，但如果在reducer中再次dispatch，就会抛出Error，因为isDispatching还没有打开。这是一个很小但在代码中很常见的点。又见到在getState方法中也是有类似的检测，而getState方法本身极其简单，就是判断一下isDispatching，如果已经dispatch了一个action并且reducer计算结束了，就直接返回currentState对象。简单的一塌糊涂。。。

### 问题三：这样只是初始化了状态，能够更新状态，能够获取当前状态，那对更新进行订阅是什么样的？毕竟不可能仅仅孤立的去更新状态；
 
subscribe函数会接收监听器函数作为参数，然后照例是类型检测、isDispatching锁检查(这就保证了我们在reducer计算过程中也不能添加监听器)，接着一个重点是，每次订阅都会有一个isSubscribed标志。核心部分就是将listener给push进nextListeners数组中，并且返回一个取消订阅的函数，该函数使用了对应的isSubscribed标志形成闭包，首先会利用isSubscribed判断是否已经取消订阅，如果是就会return，避免重复取消。而后面则是找到对应的listener，将其从nextListeners中splice掉。这就是完整的订阅函数，也非常简单。

但是什么时候会执行listener呢？事实上回到dispatch中我们即可看到，有一个对listener数组的交替赋值(current = next这种)，并且遍历监听器去执行他们，这样就完成了订阅-->执行的过程。

除此之外，还有个中间过程ensureCanMutateNextListeners()，在订阅与返回的取消订阅方法中均有出现。其实这个东西看似简单考虑的却很充分。dispatch方法中我们循环执行所有的listener，如果在执行的过程中，我们又添加了新的订阅函数，现有的循环就会受到影响。因此，ensureCanMutateNextListeners就是为了保证在这种情况下，用nextListeners对currentListeners进行一次快照，然后将新添加的listener给push到这个快照里去，从而避免屏蔽订阅对当前循环的currentListeners产生影响。在下一次dispatch时，又会对listener们进行遍历，但在此之前，会有如下代码：

```javascript
    const listeners = (currentListeners = nextListeners)
    for (let i = 0; i < listeners.length; i++) {
      const listener = listeners[i]
      listener()
    }
```
可以看到，我们执行的listener队列，是从快照中取出的，这就自然得到了更新后的订阅队列。至此完成一个闭环，nice。

### 问题四：replaceReducer是干什么的？？？

这个函数代码非常简单，就是将接收到的nextReducer替换掉内部的currentReducer，并且dispatch一个REPLACE action，这里4.x的版本和早些版本的redux在此处dispatch一个INIT action不同。事实上，REPLACE action本质上也是一个随机数，只是与INIT action的值不同，可能是新版本的作者觉得还是对二者进行区分较好。但是，效果目前是一样的，同问题一中所讲的一样，起到了初始化reducer对应state树的作用。

这个东西通常也不会直接去用，毕竟我们都是加上连接库来使用redux的嘛。不过内部项目中有类似的代码，在内部分享的时候，可以结合着去聊聊。

注释对其解释的很清晰：

> 替换当前store用来计算状态的reducer。
>
> 如果你应用了code splitting并且你想动态载入一些reducer，你可能需要用到这个。 
> 
> 如果你针对redux应用了hot reloading机制，你也可能会需要用到它。

### 问题五：[$$observable]又是啥？

这个我也看不明白，先不深入去挖啦！有空再看吧我先去食堂吃饭了。
