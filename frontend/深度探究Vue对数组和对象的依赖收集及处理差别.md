## 探究一：DependArray

关键代码：

```javascript
// defineReactive get()
if (Dep.target) {
   dep.depend()
   if (childOb) {
     // 这个dep直接给watcher，object/array 都会传该dep
     childOb.dep.depend()
     if (Array.isArray(value)) {
       // 这是第二个dep，array独有，对每个数组元素都向watcher发送了dep
       // 为了给数组的每一项都为对应的watcher发送dep，
       // 考虑以下情况：Vue.$set(this.books[0],key,value)
       dependArray(value)
     }
   }
 }

```

```javascript
function dependArray (value: Array<any>) {
  for (let e, i = 0, l = value.length; i < l; i++) {
    e = value[i]
    e && e.__ob__ && e.__ob__.dep.depend()
    if (Array.isArray(e)) {
      dependArray(e)
    }
  }
}
```

**清晰可见，当defineReactive的对象是数组时，调用dependArray,而dependArray又继续递归**



### dependArray又做了什么？

```
 e && e.__ob__ && e.__ob__.dep.depend()
```

无非就是找到array[i]对应的watcher，也就是```__ob__```，然后添加该依赖即可

<b>这个ob其实是构造watcher时挂载上去的</b>

```javascript
constructor (value: any) {
    this.value = value
    this.dep = new Dep()
    this.vmCount = 0
    def(value, '__ob__', this)
    if (Array.isArray(value)) {
      if (hasProto) {
        protoAugment(value, arrayMethods)
      } else {
        copyAugment(value, arrayMethods, arrayKeys)
      }
      // 下面会详细说这个函数
      this.observeArray(value)
    } else {
      this.walk(value)
    }
  }
```



### 为什么要两个dep？

首先要注意一下，在构造watcher时对数组的依赖观察

```javascript
observeArray (items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
}
```

```javascript
function observe (value: any, asRootData: ?boolean): Observer | void {
  // 普通数据类型不响应
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  let ob: Observer | void
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else if (
    shouldObserve &&
    !isServerRendering() &&
    // 不是服务端渲染就打上依赖
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```

可见对Array和object的处理差异已经出来了，<b>array是不对普通数据类型进行观察的，而object则要对每个key进行响应式添加</b>

```javascript
简单来说，对obj的处理如下
function handleObj(obj){
	let keys = Object.keys(obj)
	let i = keys.length - 1
	while(i--){
		defineReactive(obj[keys])
	}
}
```

ps:事实上，如果把数组改为object的响应方法，也能做到不用$set自动响应更新，但是defineProperty完全无法处理length的直接变化，所以vue3会采用proxy，想必到时候array将会采用全新的依赖收集方案



<b>我们仍未解决为何需要两个dep的问题</b>

不过很快，我们就翻到了$set的源码(observer/index function set)关于array部分的处理

```javascript
 export function set (target: Array<any> | Object, key: any, val: any): any {
  if (process.env.NODE_ENV !== 'production' &&
    (isUndef(target) || isPrimitive(target))
  ) {
    warn(`Cannot set reactive property on undefined, null, or primitive value: ${(target: any)}`)
  }
  if (Array.isArray(target) && isValidArrayIndex(key)) {
    target.length = Math.max(target.length, key)
    target.splice(key, 1, val)
    return val
  }
  if (key in target && !(key in Object.prototype)) {
    target[key] = val
    return val
  }
  const ob = (target: any).__ob__
  if (target._isVue || (ob && ob.vmCount)) {
    process.env.NODE_ENV !== 'production' && warn(
      'Avoid adding reactive properties to a Vue instance or its root $data ' +
      'at runtime - declare it upfront in the data option.'
    )
    return val
  }
  if (!ob) {
    target[key] = val
    return val
  }
  defineReactive(ob.value, key, val)
  // 通知数组的watcher求值，这个watcher是dependArray打上的第二层watcher
  ob.dep.notify()
  return val
}
```

以及 array monkey patch

```javascript
methodsToPatch.forEach(function (method) {
  // cache original method
  const original = arrayProto[method]
  // def把func作为val绑定在array的原型上，所以this指向的是array自己
  def(arrayMethods, method, function mutator (...args) {
    const result = original.apply(this, args)
    const ob = this.__ob__
    let inserted
    switch (method) {
      case 'push':
      case 'unshift':
        inserted = args
        break
      case 'splice':
        inserted = args.slice(2)
        break
    }
    if (inserted) ob.observeArray(inserted)
    // notify change 
    ob.dep.notify()
    return result
  })
})
```

so，你看到什么了吗？

<b>两个dep只是为了set方法做准备的，set一旦触发，立马对新的对象做依赖收集，并且触发老array的观察者更新，如果没有子watcher，那set就不能嵌套更新数组里的对象元素了</b>



### 说人话！！！

好的，让我用通俗易懂的方法来解释一下。

1. dependArray把数组里面所有对象元素递归的收集起来，并且把自身作为依赖添加进他们的__ob__里面

2. set只patch了一件事，在更新后通知所有的sub更新了，这时通过dependArray找到的那些对象都被通知到了。

3. 我们看一下vue里对象的样子

   ```
   {
   	a:1,
   	b:{
   		__ob__:{dep...}
   		a:1
   	},
   	__ob__:{dep...}
   }
   ```



## 总结

1. 只有Object的key有资格被defineReactive处理
2. array里面的object也可以被响应更新，并不需要set，因为它自身自带依赖收集
3. array可以实现响应更新，但是因为length无法处理，所以放弃了原生api的实现，转而向观察者模式实现
4. vue作为数据的array不是原生array
5. 综上可见，vue3以proxy api实现的依赖收集必将出现大的改动，因为proxy 完全有能力继承一个原生array，完美处理length







