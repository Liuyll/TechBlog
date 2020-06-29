### 前言

在实际开发过程中，我们经常把数据fetch直接写到组件里面。对大型的项目可能会把数据写在一个单独的`request`文件里进行维护，但是数据整体没有模块化，维护起来还是很困难。

我们希望在每一个模块里维护一个模块化的取数文件，就像下面这样：

![image-20200430182556225](/Users/liuyl/Library/Application Support/typora-user-images/image-20200430182556225.png)

在获取数据的时候，步骤也变成了:

```
// normal
fetch(args).then(data => {
	// check
	// transform
	setState(data)
})

// redux/db
props.dispatch(getDataAction(args))
```

当然，如果你熟悉redux的话，你已经明白它的整体流程了。

```
redux -> fetch -> data return redux -> rerender component 
```



我们要做的事情，就是造一个符合这样写法的轮子出来。



### middleware

#### async action

众所周知的是，`reducer`接受的`action`必须是一个数值。当然，一个函数`action`也可以通过`redux-thunk`这样的中间件被使用。

`redux-thunk`的思想在这里很有用处，因为`fetch`的过程肯定是异步的，我们简单看下它的源码

```
const createThunk = store => next => action => {
	if(action is function){
		return args => async (dispatch,getState) => {
			let data = await action()
			dispatch(data)
		}
	} else next(action)
}
```

当然，如果你不了解`middleware`的工作模式，可以去看下相关`compose`的文章，原理非常简单，这里不讲解了。



不过，我们需要对每一个`action`获取的数据都进行相同的流程检查，比如返回状态是否在200和300之间等，这部分重复的工作，我们依然可以靠中间件解决。

#### check result

其实这里有两种检查的方法:

+ 使用`fetch`库，并在`fetch`时进行检查，如`axios`。
+ 在中间件里检查，如果返回值不符合，则不进入`reducer`流程，或发送一个标识`status:fail`的`action`

当然，在`redux-db`里，实现了自己的`fetch`库，同时也实现了可流程化控制的`middlewares`



#### 实现

##### 流程控制

最常见的流程控制方法是`generator`

```
function *control() {
	yield before()
	yield process()
	yield after()
}
```

我们的`action`是这样的

```

```



参考这样的思路，我们也可以在中间件里进行这样的流程控制:

1. 请求
2. 状态处理
3. 上报监控
4. 其他业务逻辑

<b>获取</b>

```
fetchControlMiddleware = store => next => action => {
	const state = action['fetch']
}
```

