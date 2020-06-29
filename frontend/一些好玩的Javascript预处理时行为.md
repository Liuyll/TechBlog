### 聊聊引擎

为照顾不熟悉编译原理的同学，我们先回顾一下常规语言的执行过程

#### 编译型语言

1. 词法分析(token化)
2. 语法分析
3. 形成AST
4. 编译器优化代码
5. 翻译为机器语言
6. 形成二进制可执行文件

#### 解释型语言

> 很多同学认为解释型语言是没有`Compiler(编译器)`而只存在`Interceptor(解释器)`，事实上这是一个很大很大误区，其实解释型语言同样要经过`tokenize -> lexical analysis -> syntactic parsing -> generate AST tree` 这样一个过程，只是说解释型语言不生成可执行的文件，它在需要时根据`AST`来动态执行。



#### 术语

为方便叙述，我们先介绍一下一些常见的术语:

+ TurboFan JIT编译器(v8)
+ Ignition 解释器(v8)
+ JIT 即时编译 动态将字节码转化为机器码，它旨在节省内存

+ 机器码 cpu可直接运行，行数大
+ 字节码 高度抽象的代码，但cpu依然不能直接执行，行数少



#### AST

必须提及一下AST，这是我们后面叙述的核心

```javascript
function a() {
	let a = 6
  {
    let b = 7
  }
}
```

一个这么简单的函数，在被翻译为`ast`后却格外的复杂

> 你可以尝试在这里[ast](https://astexplorer.net/)把一个函数编译为AST，并且建议初学者选择esprima/babylon7

#### how to token?

我们简单举个例子

`let name = 'liuyl'`

上述语句会被token化为几个不同的属性:

+ let 关键字
+ name 标识符
+ = 操作行为
+ 'liuyl' Literal

###变量提升

js是JIT(即时编译)语言，它的执行有两个过程：

1. 编译
2. 执行

在第一次编译javascript的过程中，变量扫描和提升被执行 

#### 技术细节

##### 执行上下文

> 任何一段代码在执行的时候都有它的自己的执行上下文

![执行上下文](/code/tech_blog/图片/下载.png)

+ js引擎在进行编译的过程中，会创建一个根执行上下文，它作为所有没有自己的执行上下文的代码的执行上下文

+ 除了根执行上下文，也可能存在自己独立的执行上下文，比如进入到一个函数中，它会切换到函数上下文(函数仅在第一次执行时被`TurboFan`编译成机器码)

+ 此时抽取可提升的变量(var,function)，并分配内存，将其写入到`VaribableEnvironment`里，但不修改硬编码

  你可以把执行上下文看成以下结构

  ```javascript
  function fn(){
  	console.log(b) // undefined
  	var a = 7
  	var b = 6
  }
  var b = 10
  
  // -----------> console.log(b) 语句对应的执行上下文
  global ExecuteContext {
  	b -> undefind
  	fn ExecuteContext {
  		a -> undefined
  		b -> undefined
      outer -> global
  	}
  }
  
  // ------------> var b = 6 语句对应的执行上下文
  global ExecuteContext {
  	b -> undefind
  	fn ExecuteContext {
  		a -> 7
  		b -> 6
      outer -> global
  	}
  }
  ```

+ 按照上述步骤，js引擎绑定每个语句对应的执行上下文

+ 生成字节码

+ 生成机器码

  

<b>注:以下几种环境会生成单独的执行上下文:</b>

+ 全局
+ 函数
+ try
+ catch
+ eval



在讨论`Variable Environment`和`Laxical Environment`之前，我们先想几个问题：

1. `let`声明的变量存在变量提升吗？
2. 暂时性死区的原理是什么？
3. 变量提升跟块级作用域冲突吗？
4. this的实质是什么？

##### outer 与 词法作用域

考虑以下情况：

```javascript
function foo() {
	var name = 'cba'
	bar()
}
function bar() {
	console.log(name)
}

var name = 'abc'
foo() // 'abc'
```

如果你已经熟悉上文提到的执行上下文，你可能会陷入一个误区：

+ bar的执行上下文是`global`还是`foo`？
+ bar的执行上下文随着它执行的位置改变而改变吗？

之前有提到过，一个函数只在初次执行时被编译。也就是说，它在执行时已经过了一次完整的编译阶段，它已经存在固定的执行上下文，这个执行上下文的确定就是词法作用域。

词法作用域的确定方法：<b>定义在哪，执行上下文就在哪</b>



顺便提一下，闭包也是利用了词法作用域进行的执行上下文绑定

##### 变量环境 Variable Environment

上面已经提到过，`var`和`function`在编译阶段都会放入`Variable Environment`

##### this

我们先看一个例子

```javascript
let a = {
  _name:'abc',
  printName(){
    return _name // cba
    return this._name // abc
    return function() {
      return this._name // undefined
    }
  },
  getName:this._ame // undefined
}

let name = 'cba'
```

如果你已经明白执行上下文，应该就很清楚以下几点：

+ `getName`的执行上下文是`global`，`this`自然指向`window`

+ `printName`作为函数，有自己的执行上下文，这个执行上下文是可以指定的：

  + bind
  + 对象`.`调用

  + 构造函数调用 (事实上，它也只是借用构造函数使用`call`来进行绑定的)
  + this不会继承`outer`的this，this跟执行上下文链完全是两码事

明白了这个，你就能轻易的解答出那些弱智的面试题：

```javascript
var length = 10
function getLength() {
  console.log(this.length)
}
let obj = {
  b(){
    arguments[0]()
    // execute context: arugments
    // arguments.length = ?
  }
}

obj.b(getLength,1,2,3,4) // 5
```

层层分析执行上下文，你就能轻易获得答案。

##### 词法环境 Lexical Environment 

ES6引入了`let`和`const`两个全新的变量定义以弥补js没有块级上下文的缺陷。

在刚才的例子里，一个变量在编译阶段被分配内存并储存至执行上下文的`Variable Environment`，它是不可能存在块级作用域的。

那么js是如何既实现变量提升又实现块级作用域的呢？

##### 块级作用域

答案就是`Lexical Environment`，它是过程如下：

1. 编译阶段遇到`let`,`const`变量定义，把它移入当前执行上下文的词法环境，每个块级上下文都有自己的词法环境，一旦离开块级上下文，那么自动弹出该块的词法作用域。

   你可以理解为一个这样的数据结构:

   ```javascript
   let globalEnv = 6
   {
   	let a = 7
   	let b = 8
   	{
   		let d = 9
   		let e = 10
   	}
   }
   
   // 当执行到let e = 10时的词法环境
   LexicalEnvironment = [global,{a:7,b:8,{d:9,e:10}}]
   
   // 当执行完let e = 10时的词法环境
   LexicalEnvironment = [global,{a:7,b:8}]
   ```

2. 变量寻找时，优先从`Lexical Environment`查找，最后才去`Variable Environment`查找

3. 只有执行到一个块级作用域时，才进行处理



到此，我们能解决上面抛出的几个问题了：

+ 1.2 :let声明的变量确实也存在变量提升，但它被放入`Lexical Environment`里，不展现出变量提升的行为，反而会出现暂时性死区的语法错误。
+ 3 变量提升跟块级作用域确实冲突，所以js引入词法环境来做兼容

-----------------------------------------------------------------练习-----------------------------------------------------------------------------------

来练习一下吧

```javascript
eval('let a = 7')
eval('var b = 7')
console.log(a) // error
console.log(b) // 7
```

这两种行为有什么区别？希望你已经熟悉`执行上下文`和`块级作用域`的真正细节，`eval`会开启新的执行上下文，`let`声明的变量将形成块级作用域，此时无法在外部访问

### 变量冲突

```javascript
// case1:
function a(){}
console.log(a) // f
var a
console.log(a) // undefined
```

```javascript
// case2:
var a = 0
console.log(a) // 0
function a(){}
console.log(a) // 0
```

为什么交换函数和变量声明的位置就会出现不一样的行为？

再看下更奇怪的情况

```javascript
// case3:
var a 
console.log(a) // f
function a(){}
console.log(a) // f

// 可见，当a未被赋值时，a不进行任何变量覆盖
```

事实上，JavaScript在预处理时会进行变量扫描，顺序如下:

1. 扫描函数声明，若遇冲突，则覆盖
2. 扫描变量声明，若遇冲突，则忽略

看这样一种情况:

```javascript
console.log(a) // f
var a;
function a(){}:
```

我们可以总结出它的行为:

1. var 声明重要性小于函数声明
2. var 初始化(执行到赋值语句) 大于任何声明(因为函数声明是提前的，执行到赋值时，变量直接被覆盖了)
3. var 声明只做保底的undefined，若遇到任何冲突，它都被覆盖

当然，这只是它的表现行为，为什么会这么表现？上面一章已经把原理讲得很清楚了。

