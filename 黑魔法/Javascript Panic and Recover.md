#### 行为

照顾没有写过`golang`的小伙伴，先看看`panic and recover`的行为

```javascript
function fn1(){
    defer(() => {
        let err = recover()
        if(err) if(err.message == 'fuck') {
          // dosomething throw or catch
        }
    })
  
    throw new Error('fuck')
    return 1
}
let protectFn1 = Protect(fn1)
protectFn1()
```

> 我们这里不讨论<b>不同语言错误捕获机制好坏</b>这样的哲学问题，只是单纯考虑一下js如何实现这样的语法？



### 实现

#### 先回顾一些技术啦

##### try / finally

其实对于很多前端同学来说，`try/finally`几乎没有见过，更没有使用过。不过如果你喜爱阅读源码，应该能在很多框架里看到这样的语法。

```javascript
// react batchUpdate 伪码
try {
	isTransaction = true
	flushUpdateQueue()
} finally {
	isTransaction = false
}
```

> 几乎所有依赖`transaction`的library都大量使用该语法

我们探讨一下它的特点

```javascript
function A() {
	let a = 1
	try {
		return a
	} finally {
		a = 5
	}
}

A() // 1
```

如果你熟悉`golang`，那么你应该见过这样的场景

```go
func b() (r int) {
    defer func() {
        r++ // return 1
    }
    return r
}

func b() int {
    defer func() {
        r++ // return 0
    }
    var r int
    return r
}
```

很幸运，js利用`try/finally`也能轻松模仿这样的值，只不过它没办法支持有名返回值，不过这也恰好是我们不需要的。

> 后续会提到它的用处

#### 重构词法作用域

> 未来我会专门开一片文章讲解js底层的一些东西，所以这里只简单提下

为什么我们需要重建词法作用域?

考虑一下以下场景：

```javascript
function A() {console.log(b)}
function B() {
  let b = 1
  A()
}
let b = 9
B() // 9
```

当然，只要你稍微属性js底层，这样的低级面试题就难不倒你。不过如果要求你<b>必须打印出1</b>，又该怎么办呢？

```javascript
function B(A) {
  let b = 1
	let copyA = () => eval(`(${A.toString()})()`)
	copyA()
}
B(A) // 1
```

是的，我们很巧的利用`toString`实现了词法作用域重构。

> 类似的利用场景非常多，web worker 的webpack loader也利用了这样的方案

有了这个东西，我们就可以直接`defer`，而不运行`defer`了

#### 意外

不巧的是，重构词法作用域后，又出现了新的问题

```javascript
// B.module
export function B(A) {
  let b = 1
	let copyA = () => eval(`(${A.toString()})()`)
	copyA()
}
```

```javascript
// A.module
import { B } from './B'
let variable
function A() {
  console.log(variable)
}
B(A)
// error:variable is not defined
```

问题很容易定位，就是重构词法作用域以后，新的词法作用域到了`B`函数那里，它的全局对象也只能到`B.module`,而`variable`是在`A.module`里声明的，自然出现`not defined `的错误

当然，我们还是可以通过重构词法作用域的方法重构全局环境，并且，它有多种方法可以实现



#### eval执行作用域探究

> 我们将探究<b>nodejs</b>环境下，`eval`的行为，浏览器的行为可以[参考](https://blog.csdn.net/cuixiping/article/details/4823119)

在此之前，我们来踩下小坑

```javascript
eval('let a = 7')
eval('var b = 7')
console.log(a) // error
console.log(b) // 7
```

这两种行为有什么区别？希望你熟悉`执行上下文`和`块级作用域`的真正细节，`eval`会开启新的执行上下文，`let`声明的变量将形成块级作用域，此时无法在外部访问

为什么要踩这个坑？因为我们需要通过`eval`来声明函数

```javascript
function A(){}
eval(A.toString())
```

如果`A`是依旧序列化的函数，那么一个真实的`A`函数将会被声明出来

##### eval的执行作用域

```javascript
function Test() {
	eval('var a = 7')
	global.eval('var b = 7')
}
console.log(a) // error
console.log(b) // 7
```

bingo,我们找到了

#### 实现

当然，有上面那些技术还不够。上面的例子已经指出了，重构词法作用域必须要用户配合传入函数才能实现。所以我们务必这样做

```javascript
function WrapEnvironment(fn) {
  const copyFn = () => eval(`(${fn.toString()})()`)
	copyFn()
}
```

我们可以在`WrapEnvironment`里做一些其他的事情，比如错误捕获

```javascript
function WrapEnvironment(fn) {
  const copyFn = () => eval(`(${fn.toString()})()`)
	try {
		copyFn()
	} catch(e) {
		// balabala
	}
}
```

##### recover

这还不够，我们不仅要捕获错误，还要处理错误。

利用刚才提到的词法作用域重构，我们可以在用户无感知的情况下调用`recover`

```javascript
function WrapEnvironment(fn) {
	const copyFn = () => eval(`(${fn.toString()})()`)
	try {
		copyFn()
	} catch(e) {
		// balabala
	}
  
  function recover() {
    if(repo.err) try {
      return repo.err
    } finally {
      repo.err = null
    }
  }
}
```



##### panic and defer

这些都可以





