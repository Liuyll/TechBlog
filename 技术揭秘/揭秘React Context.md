### Context的特性

#### 为何Consumer不受SCU的控制？

![image-20200416203008141](/Users/liuyl/Library/Application Support/typora-user-images/image-20200416203008141.png)



看一段在`beginWork`里的代码

```
if (
	!hasLegacyContextChanged() &&
	updateExpirationTime < renderExpirationTime
) {
	switch (workInProgress.tag) { 
		case Context.Provider: {
			
		}
  }
	return bailoutOnAlreadyFinishedWork(
    current,
    workInProgress,
    renderExpirationTime,
  );
}

// bailoutOnAlreadyFinishedWork
if(childExpirationTime > renderExpirationTime) {
	// clone child fiber to nextUnitOfWork
 	cloneChildFibers(current, workInProgress);
 	return workInProgress.child;
} else {
	// skip
	return null
}
```

由源码已经能够看出，只要`ContextProvider`将`childExpirationTime`的优先级提升至大于`renderExpirationTime`就能让子组件即使在父组件被`SCU`的情况下也能更新，它的提升方法就是`propagateContextChange`



![preview](https://pic3.zhimg.com/v2-f22be3b6edd168c86a72a2d324e4e2c2_r.jpg)

