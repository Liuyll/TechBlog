### 看一下寻常中间件的实现

```javascript
// ice-store

function compose(middlewares:middleware[],ctx:Ctx):Promise<any>{
  function gonext(middleware,next){
    return async () => middleware(ctx,next)
  }
  let next = () => Promise.resolve()
	middlewares.reverse().forEach((middleware) => {
    next = gonext(middleware,next)
    //next = async () => middleware(ctx,next)
  })
	return await next()
}
```





### 从Redux说起

+ 看一段redux的compose

```javascript
const compose = middlewares.reduceRight((a,b) => (...args) => b(a(...args)))
```

+ 看一段redux的applyMiddleware

  先Review一下middleware的写法
  
  `middleware = store => next => action => {}`
  
  ```javascript
  function applyMiddleware(store,middlewares){
    let chain = []
    const middlewareAPI = {
      getStore:'',
      dispatch:(action) => dispatch(action)
    }
  
    chain = middlewares.map(middleware => middleware(middlewareAPI))
    const dispatch = compose(...chain)(store.dispatch)
    // store.dispatch 作为最后一个middleware的next传入，标识已经执行完前面所有的middleware了
    // 而执行完后返回的 action => {} 函数将作为next返回到前一个函数
  }
  
  
  ```

### koa呢？

+ Review

`middleware: (ctx,next) => {}`

```javascript
context = ... // koa provide
const chain = []
// change to ctx => next => () => {} 
// 此时完全和redux相同
chain = middlewares.map(middleware => (next) => () => middlware(ctx,next))
compose = (...fns) => fns.reduceRight((a,b) => (lastNext) => b(a(lastNext)))
emit = compose(...chain)(() => Promise.resolve()) 
emit()
```

当然，这是一个粗暴的解法，koa的中间件是完全异步的，它怎么实现的呢？

#### 一个前提

我们知道，`new Promise(cb)`里的cb如果返回了一个`promise`，那必须等待内部Promise的状态全部resolve后才会继续执行，koa利用了这点

```javascript
// fn = (ctx,next) => {}
// koa-compose:

next = Promise.resolve()
ctx = createCtx()
middlewares.reverse().forEach(middleware => {
  // 此时，middleware如果返回一个Promise，那么进入Promise执行链，直到该链完成
  next = () => Promise.resolve(middleware(ctx,next))
})

```

现在中间件有两种写法

```javascript
// 1.
async (ctx,next) => {
  next()
}

// 2.
async (ctx.next) => {
  return next()
}
```

确实，`1,2`种写法在常规运行下都能正确执行。如果你已经了解上述的中间件写法的本意，那么不难发现，`next`只是执行后续`middleware`的启动器而已。

但是如果出现了错误呢？

```javascript
async function errorCatch(ctx,next) {
	  try {
      await next()
    } catch(e) {
      console.log(e)
    }
}

// 1. not catch
async (ctx,next) => {
  next()
}

// 2. catch
async (ctx,next) => next()

async (ctx.next) => {
  throw new Error()
  return next()
}
```

很明显，不返回`next()`导致Promise的链断掉了，下面是一个简单的模拟情况

```javascript
Promise.resolve(_ => {
  // can
  return Promise.resolve().then(_ => new Error())
  // not can
  Promise.resolve().then(_ => new Error())
}).catch(e => e)
```

如果你不明白原理，不妨看下Promise的简单实现，事实上每个then返回一个全新的Promise，数据来自于then内部的返回值