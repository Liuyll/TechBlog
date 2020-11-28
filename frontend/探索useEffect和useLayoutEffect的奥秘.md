> 在开发 react-alive 的时候，出现了一个bug，一番排查后，最小复现代码如下:

```javascript
// father:
useEffect(() => {
	console.log('i am father effect')
})

useLayoutEffect(() => {
	console.log('i am father layout effect')
})

// son:
useEffect(() => {
	console.log('i am son effect')
})

useLayoutEffect(() => {
	console.log('i am son layout effect')
})
```

ps:我们知道,`useLayoutEffect`是同步执行的，而`useEffect`是hack过后的`requestIdleCallback`执行的，`requestIdleCallback`的最小间隔为16.66ms，也就是浏览器按照60帧的方式进行渲染的间隔



所以打印结果是:

```
i am son layout effect // son mounted
i am father layout effect // father mounted
i am son effect // son idle exec
i am father effect // father idle exec
```



