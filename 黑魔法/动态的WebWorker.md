## 动态的WebWorker

前言：

加载其他脚本来执行webworker只在很少的业务场景里被使用，我们更希望webworker能作为一个可拔插式的方便工具



### 如何实现？

1. worker构造函数的参数可以接受一个ObjectUrl

2. blob可以由URL.creatrObjectURL转化为ObjectUrl

3. 字符串可以转化为blob

4. 函数可以转化为字符串，并且通过字符串拼接执行

   <b>所以步骤就是4-3-2-1</b>

### 代码

