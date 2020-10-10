## module

#### 文件即模块

在ts里，有一个概念叫做文件即模块，也就是说,像下面这样:

```
import * as A from 'A'
```

我们同时默认引入了`A`这个`module`到当前模块里来。

####扩展模块

上面提到了文件即模块，所以每个文件作为导出时就是一个默认模块。我们可以在其他文件里扩充它

```
// A.ts
class A {}
export default A
```

```
// B.ts
import A from 'A'
declare module 'A' {
	interface A {
		...
	}
}
```

这样可以动态的扩充A的实例属性里`class A`

同理，也可以扩展它的静态属性

```
declare module 'A' {
	namespace A {
		let z:number;
	}
}
```

唯一需要注意的是，必须先通过`import `来表示需要扩展模块。

如果直接`declare module 'xx'`这会新建一个模块

#### 模块中模块

当然，一个模块中如果只能存在一个模块，那拆分的文件可能有点多。

我们可以在模块里声明其他模块：

```
export module AO {}
```

这样我们就能引入AO模块了

不过我们希望用namespace来表示在文件里的代码分割，模块只作为一个且唯一一个向外暴露。



## namesapce

实际上，namespace除了不能作为一个文件以外。它的用法与module并没有任何区别



## 声明合并

#### 合并namespace和class以达到静态属性描述

```
// dynamic
interface A {}

// static
declare module A {
	declare let xxx
	declare function xxx()
}

// or 
namespace A {
	export let xxx
}

```



### 全局合并

```
declare global {
    interface Array<T> {
        toObservable(): Observable<T>;
    }
}
```

