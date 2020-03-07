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
console.log(a) // 0

// 可见，当a未被赋值时，a不进行任何变量覆盖
```

事实上，JavaScript在预处理时会进行变量扫描，顺序如下:

1. 扫描函数声明，若遇冲突，则覆盖
2. 扫描变量声明，若遇冲突，则忽略

<b>冲突情况:</b>

+ 若var定义的变量不初始化，则不覆盖函数(若无同名函数声明，则为undefined)
+ 在初始化var变量后边定义的函数，将被忽略
+ var初始化的时机是解释器解释该行语句的时机