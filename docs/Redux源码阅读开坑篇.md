# 开坑

最近想看看Redux，毕竟不大有那么多相关文章，加深下理解吧。

Redux+Saga+Immutable好舒服。

## src/utils/isPlainObject.js

这个方法就是检测是不是一个普通对象，换句话说是不是用new Object()或者{}生成的。检测方法就是用Object.getPrototypeOf方法，这里涉及到继承原理方面的东西，可以看https://github.com/Hydraz320/FE.Note/blob/master/JS/%E7%BB%A7%E6%89%BF/oop-prototype.md 下面的图讲得很清楚了。蛮有趣的一段代码，先获取对象的指针，然后循环去找该对象的__proto__直到到达null，这种情况下plain object必然还是指向Object.prototype，而非plain object的__proto__就可能指向某个类的原型了。

## createStore.js

源码注释真的好看，对各个概念的解释不要太精准且容易理解。

> createStore文件代码注释
> Creates a Redux store that holds the state tree.
> The only way to change the data in the store is to call `dispatch()` on it.
>
> There should only be a single store in your app. To specify how different
> parts of the state tree respond to actions, you may combine several reducers
> into a single reducer function by using `combineReducers`.
>
> @param {Function} reducer A function that returns the next state tree, given
> the current state tree and the action to handle.
>
> @param {any} [preloadedState] The initial state. You may optionally specify it
> to hydrate the state from the server in universal apps, or to restore a
> previously serialized user session.
> If you use `combineReducers` to produce the root reducer function, this must be
> an object with the same shape as `combineReducers` keys.
>
> @param {Function} [enhancer] The store enhancer. You may optionally specify it
> to enhance the store with third-party capabilities such as middleware,
> time travel, persistence, etc. The only store enhancer that ships with Redux
> is `applyMiddleware()`.
>
> @returns {Store} A Redux store that lets you read the state, dispatch actions
> and subscribe to changes.
 
hahah

