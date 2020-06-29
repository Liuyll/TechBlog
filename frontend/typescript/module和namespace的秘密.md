## module

#### 文件即模块

在ts里，有一个概念叫做文件即模块，也就是说,像下面这样:

```
import * as A from 'A'
```

我们同时默认引入了`A`这个`module`到当前模块里来。



#### 模块中模块

当然，一个模块中如果只能存在一个模块，那拆分的文件可能有点多。

我们可以在模块里声明其他模块：

```
export module AO {}
```

这样我们就能引入AO模块了



## namesapce

与module不同的是，一个`class`也可以是一个`module`



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

