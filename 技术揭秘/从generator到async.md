### generator和Promise

在`async/await`未成为标准之前，我们会看下这样的代码:

```javascript
function *run() {
  const h = Promise.resolve(...)
  // take advantage of h
  const f = Promise.resolve(...)
  // take advantage of f
}

function co(fn) {
  const start = fn()
  const run = start.next()
  return next(run)
  
  function next(cur) {
    if(cur.done) return
    if(cur.value instanceOf Promise) {
      cur.value.then(data => {
      	let nextCur = cur.next(data)  
        if(!nextCur.done) next(nextCur)
      })
    }
  }
}
```

相信使用过`co`的同学都能看出来它做了什么，这里简单提一下

1. 取出`Promise`的值
2. 传入`Promise`的值
3. 如果需要继续，那么递归调用



为了讲清楚Promise，我们回到es5:

#### thunkify

想象一下这样的场景：

```javascript
function run *(rs) {
	rs.forEach(path => {
		const h = yield read(path)
		// utilize h dosomething
	})
}

```

现在我们没有`Promise`,而原生的`readFile`函数明显是个异步函数，那么怎么配合`genrator`来控制流程呢？

想一想我们怎么把`readFile`改为`Promise`形式的？

```javascript
function readFilePromise(path) {
	return new Promise((resolve,reject) => {
		readFile(path,(err,data) => {
			if(err) return reject(err)
			resolve(data)
		})
	})
}
```

很好，我们利用这种形式来模拟`Promise`

```javascript
function thunkify(fn) {
	let thunkFn = function(...args) {
    return function(cb) {
      fn(...args,cb)
    }
  }
  return thunkFn.bind(fn)
}

const read = thunkify(readFile)
```



