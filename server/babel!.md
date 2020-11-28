## babel-traverse

## api

[参考](https://xxxxxmiss.github.io/2018/12/27/babel-traverse-api/#%E4%BF%AE%E6%94%B9%E5%BD%93%E5%89%8D%E8%8A%82%E7%82%B9%E7%9B%B8%E5%85%B3](https://xxxxxmiss.github.io/2018/12/27/babel-traverse-api/#修改当前节点相关))

### babel-types

`cloneDeep`

深克隆types

`clone`





### reference / bindings and scopes

#### bindings

bindings包含了`scope`下所有的变量声明

如:

```
// Scope1
var a = 1
let b = 2

let c = b // b is referenced
// bingings = [a,b]
```

##### property

+ referencePaths: 被引用时的path



#### scope是什么

scope是变量所处的作用域，在这个作用域下，它有一些可以引用的变量:

```
// ScopeGlobal
let ref
function ScopeA() {
		// this a binding in ScopeA
 		ref // this a reference in ScopeGlobal.ref
}
```

> 在**词法区块(block)\**中，由于新建变量、函数、类、函数参数等创建的标识符，都属于这个区块作用域. 这些标识符也称为\**绑定(Binding)**，而对这些绑定的使用称为**引用(Reference)**

> 注意，scope的binding只有变量或函数声明，而没有变量赋值

#### scope-api

+ 链表结构

##### parent

+ 功能api

  ##### `getBinding`

  ##### `hasBinding`

  ##### `hasOwnBinding`

  ```
  function scopeOnce() {
    let test = 3
    test
    
    function scopeTwo() {
      test = 4
      let c = {
      	d: test // path
      }
    }
  }
  
  path.scope.hasOwnBinding('test') // false
  path.scope.hasBinding('test') // true
  ```

  可以看出来差别



+ parent

  `parentHasBinding`



+ 自身

  `traverse`

  

### scope和path

![image-20200430000215846](/Users/liuyl/Library/Application Support/typora-user-images/image-20200430000215846.png)

Scope保有对path的引用



#### path

##### property

+ container 容器，包含了所有同级的节点

  相应api：`getSibling`

+ node

+ scope

##### api

###### get

path.get('name').node === path.node.name



###### replaceWith

为防止call by sharing，

用节点直接replaceWith,而不要用path.node = ...



###### replaceWithMuitiple



###### insertBefore

注意，插入的必须是通过`types`构建的`node`节点

```
path.insertAfter(t.variableDeclaration(
	'let', // kind
	[t.variableDeclarator(t.Identifier('ab'),argsInit)]) // 查看types文档
)
```

插入后的结果:

```
// before:
let ab1 = 5;

// after:
let ab1 = 5,
		ab = 10;;
```



## 概念

### container

```
let variable = 'abc'
```

此时`variable`的`container`是`let`的这个`VariableDefinition`

可以调用:

+ pushContainer
+ container.push
+ unshiftCoutainer

等方法插入



