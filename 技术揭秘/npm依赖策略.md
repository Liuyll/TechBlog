## npm的依赖策略

考虑下面的情况:

应用Root安装了A和B两个依赖,其中：

+ A依赖CD
+ B依赖CE

此时：

![image-20200509141749852](/Users/liuyl/Library/Application Support/typora-user-images/image-20200509141749852.png)

很明显，C依赖包是可以复用的，但它没有被复用。

这就是npm v2版本的策略，它导致了很多问题：

+ node_modules 太大了，很占空间
+ 安装很慢，文件路径非常长，windows下难以删除
+ 找到正确的引用很复杂



#### v3

在npm v3后，npm把可复用的包提到最顶层的`root package.json`里来维护

那什么叫可复用的依赖包呢？

+ 包一致
+ 包版本一致

也就是说,库A依赖react16.9和库B依赖的react16.3不能共用

那么遇到这种冲突，npm会放一份在`root`的`node_modules`里，如果有其他库需要不同的react，就在库自己的`node_modules`里保存一份。



之所以能这么做，是因为包搜索机制：

库B里的代码:

```
import * as r from 'react'
```

实际上会先从库B的`node_modules`里先查找，此时因为版本号不同，所以保存了一份`react 16.3`,用的就是`react 16.3`版本，而库A的依赖已经放在`root`里，所以直接往父目录查找即可。



##### 合并版本

但实际上，我们很可以是这样写的版本号

```
libraryA: 
"react": "~16.9.1"

libraryB:
"react": "~16.3.1" 
```

很明显，这两个版本号是可以合并的，所以只会在根目录保存一份`16.9.1`



以`react-redux`为例子：

`react-redux`对`react`的版本控制：

```
"react": "^16.8.3"
```

它的`react`版本需求的描述:

> React Redux 7.1 requires **React 16.8.3 or later.**

可以看见，高版本的`react`，它是可以直接去`root`里去获取，如果`root`依赖的`react`是低版本的，那它就没法使用了。



#### v5 

##### 版本锁

v5版本有了一个全新的功能，版本锁:`package-lock.json`



这个版本锁有什么用呢？上面已经提到了，我们依赖的版本可能是个范围，但我们希望在任何地方安装出来的`node_modules`是不会变化的，所以我们需要一个`lock.json`来把每个依赖都写死，也就是说:

```
"react": "^16.8.3" 

=====> 此时root依赖react 16.8.9
=====> package-lock.json

"react": "16.8.9"
```



> 很明显，你应该把package-lock.json提交到版本库



#### 缓存

v5重写了缓存策略

##### 缓存策略

.npm会缓存每一个install下来的包，在install时会走以下逻辑

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020042513384724.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhaWhhb3hpbg==,size_16,color_FFFFFF,t_70)

v5后，直接使用install --offline强制使用缓存

##### 缓存结构



![image-20200912122445810](/Users/liuyl/Library/Application Support/typora-user-images/image-20200912122445810.png)

上面是npm的缓存目录

+ content-v2 一堆二进制文件，可以解压为需要的package
+ Index-v5 描述性文件，对content-v2里内容的索引





> 引用：
>
> [npm 如何处理依赖与依赖冲突](http://git.code.oa.com/rn-plus/rn-plus-px2rem.git )_
>
> [探索 JavaScript 中的依赖管理及循环依赖](https://juejin.im/post/5a6008c2f265da3e5033cd93)

