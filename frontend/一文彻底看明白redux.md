## 目录

[TOC]

> 注:本文不会仔细分析redux-react

## redux的整体流程

#### 顺带提一下redux的流程

![dw topic](https://awps-assets.meituan.net/mit-x/blog-images-bundle-2017/adc95a4c.png)

+ 我们可以看到middleware作用的地方，是在action与reducer之间。因为reducer是pure function，所以一大堆把业务信息转化为pure function的中间件就出来了

+ 事实上只有reducer控制store，action触发reducer

+ store控制view是通过subscribe来订阅控制的，每次store改变，都会触发react组件触发的subscribe，将最新的store放进组件

  

### middleware究竟是什么？

##### first look code!

```javascript
const middlewareAPI: MiddlewareAPI = {
      getState: store.getState,
      dispatch: (action, ...args) => dispatch(action, ...args)
}
const chain = middlewares.map(middleware => middleware(middlewareAPI))
dispatch = compose<typeof dispatch>(...chain)(store.dispatch)
return dispatch
```

compose的写法很美妙(FIFO遍历)：

```javascript
compose = middlewares.reduce((a,b) => (...args) => b(a(...args)))
```

<b>redux的实现是FILO</b>

```javascript
compose = middlewares.reduce((a,b) => (...args) => a(b(...args)))
// FIFO和FILO的实现 取决于最内层函数是第一个中间件还是最后一个中间件
// 这里实现非常巧妙，递归倒换执行顺序，省去了reverse,可以学习一下
```

再次强调，`compose`的写法很美妙，我们详细分析一下

+ 每次reduce里b(...args)其实就是middleware的`next`,`store.dispatch`是最后一个中间件接受后的真实dispatch，此时action为pure
+ 以FILO的形式遍历栈，为每个中间件传入next
+ <b>重点</b>:返回第一个middleware的最终函数，即 `action => {}`



到这里，理一下dispatch的逻辑：

+ 遍历中间件，为每一个中间件curry一个middlewareAPI进去
+ 遍历中间件，为每一个中间件传入next，并返回第一个中间件
+ 此时dispatch就是串联好的中间件组的头部

所以middleware是<b>action -> reducers 这个过程中对数据的处理函数</b>

> 想一想thunk，saga是不是这样



> ```javascript
> 附:
> export interface Dispatch<A extends Action = AnyAction> {
>   <T extends A>(action: T, ...extraArgs: any[]): T
> }
> 
> export interface AnyAction(extends Action){
> 	type:T;
>   [extraProps: string]: any;
> }
> ```



### createStore做了什么？

##### createStore定义

```javascript
export default function createStore(
  reducer: Reducer<S, A>,
  preloadedState?: PreloadedState<S> | StoreEnhancer<Ext, StateExt>,
  enhancer?: StoreEnhancer<Ext, StateExt>
){
```

store有几个重要的方法

+ subscribe (redux-react就是在compose函数里为每个组件subscribe了patch，再由patch触发的组件更新)
+ dispatch



subscribe不能在dispatch时执行，所以两个一起讲

#####dispatch一些核心代码 

```javascript
//...
try {
  isDispatching = true
  currentState = currentReducer(currentState, action) // 这就是为什么reducer不能是异步
} finally {
	isDispatching = false
}
// try finally 只是为了清除isDispatching表示，错误依然由调用者处理
//...

// 对应下面subscribe的ensureCanMutateNextListeners
const listeners = (currentListeners = nextListeners)

// 触发订阅者响应,redux-react里rerender订阅组件
for (let i = 0; i < listeners.length; i++) {
  const listener = listeners[i]
  listener()
}
```



##### sub/unsubscribe一些核心代码

```javascript
if (isDispatching) throw new Error(...) // 首先不能在dispatch里subscribe，避免影响订阅队列

let isSubscribed = true
ensureCanMutateNextListeners() // 如果在下次dispatch之前再次新增订阅，不复制next缓冲
nextListeners.push(listener)

function ensureCanMutateNextListeners() {
  if (nextListeners === currentListeners) {
  	nextListeners = currentListeners.slice()
  }
}

// 重点提一下ensureCanMutateNextListeners函数
/*store维护了nextListener和currentListener两个类，以保证在dispatch时执行的是一个snapshot，*	而非nextListener(最新值)。这只是为了处理某种极端情况
*	该情况如下，某个listener里进行了unsubscribe,此时listeners数组会产生变化，for循环遍历将会出现问题
*/
```

[参考该issue](https://github.com/MrErHu/blog/issues/18)

> 事实上，subscribe运用了双缓冲技术，防止该次处理时订阅队列发生变化。这种技术在V8 GC 里cheney算法，以及fiber架构上都有广泛运用



### Tools函数

#### bindActionCreators

> [Action Creator](https://cn.redux.js.org/docs/Glossary.html#action-creator) 很简单，就是一个创建 action 的函数。不要混淆 action 和 action creator 这两个概念。Action 是一个信息的负载，而 action creator 是一个创建 action 的工厂。

react应用里，`bindActionCreators`的作用是让用户无redux感知的调用dispatch

