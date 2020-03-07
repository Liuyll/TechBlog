## 先谈谈LHS，RHS

<b>ps:lhs = Left Hand Side</b>

+ 不管是隐式LHS还是显式LHS，它都遵从一点，RHS把右手边的变量取值(栈中地址为引用就返回引用，为值就返回值)赋值给左边变量指向的地址

+ LHS之前可能经历符号表查询，`eg:var a = 1`

+ LHS和RHS都拥有作用域栈上寻的特点

+ ***不成功的RHS将会带来referenceError(let const带来的特殊错误不计入讨论)，而不成功的LHS则会在非严格模式下创建一个全局变量***

  ```
   // let,const
   Error:Cannot access 'b' before initialization
  ```



## Python是怎么做的？

#### 只拥有Object和Name

这是Python最神奇的特点，它不存在LHS或RHS的查找赋值过程，而是一种绑定过程。

`a = 1 # object 1 => a  `



一个例子:

```
for x in range(3):
		print(id(x))
```

***事实上，python并没有指针一说，它的左值更类似于cpp里的引用***



