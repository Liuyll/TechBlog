## recipe

> 当produce只传入一个函数时，immer会返回一个函数。这个传入的修改函数就是recipe

#### 用法

```
producer = produce((draft,arg) => {
	draft.x = 1
})

producer(currentState,someArgs)
```



#### 妙用

<b>redux的reducer接受一个state，一个action，返回一个新的state，这完全符合recipe</b>

