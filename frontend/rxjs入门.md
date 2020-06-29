## rxjs

rxjs本质上只是一个函数式工具库，它符合了函数响应式编程的规范而已。

入门rxjs需要知道几个必须的概念，整个rxjs也是围绕他们进行的。



### Observer

一个Observer是响应式编程的核心，另外一个核心(Subscriber)完全依赖于它，看一个示例:

```
let observer = Rxjs.Observer((observer) => {
	observer.next(1)
	observer.next(2)
	observer.complete()
})
```





### Subscriber

这是另外一个核心，我们通过subscriber来对监听的observer发出的事件进行响应

它符合一个规范：

```
let subscriber = observer.subscribe({
	next: () => {},
	complete: () => {},
	error: () => {} 
})
```



## 上手案例

### drag

一个拖拽活动是很复杂的，它涉及到以下三步：

1. 鼠标点击
2. 鼠标拖动
3. 鼠标松开

