# 开坑

最近想看看Redux，毕竟不大有那么多相关文章，加深下理解吧。

Redux+Saga+Immutable好舒服。

## src/utils/isPlainObject.js

这个方法就是检测是不是一个普通对象，换句话说是不是用new Object()或者{}生成的。检测方法就是用Object.getPrototypeOf方法，这里涉及到继承原理方面的东西，可以看https://github.com/Hydraz320/FE.Note/blob/master/JS/%E7%BB%A7%E6%89%BF/oop-prototype.md 下面的图讲得很清楚了。蛮有趣的一段代码，先获取对象的指针，然后循环去找该对象的__proto__直到到达null，这种情况下plain object必然还是指向Object.prototype，而非plain object的__proto__就可能指向某个类的原型了。

## createStore.js

源码注释真的好看，对各个概念的解释不要太精准且容易理解。

> createStore文件代码注释翻译

> 创建一个维护state tree的redux store。
> 唯一改变store中数据的方法就是调用`dispatch()`。

> 你的应用应当只有一个单一的store。为了指定state tree的不同部分如何与actions进行对应， 你可以利用`combineReducers`来combine若干个reducers以生成一个单一的reducer function。

> @param {Function} reducer 是一个返回下一个state tree的函数，它会根据当前state tree和action来进行计算和处理。

> @param {any} [preloadedState] 初始state。你可以选择在一个同构应用中从服务端获得一个状态，或者从先前存储在用户会话中的序列化的数据中取出。

> 如果你使用`combineReducers`来创建一个根reducer function，那么这个初始state必须与`combineReducers`创建的对象的key值一一对应。 

> @param {Function} [enhancer] store enhancer。 你可以使用如中间件、时间旅行、持久化等第三方功能来增强store。Redux自带的store enhancer只有`applyMiddleware()`。

> @returns {Store} 一个让你可以读取状态的、触发action的、订阅变化的redux store对象。
 

