### 先谈谈Hooks与Class 编程思想的区别

#### 从ComponentWillReceiveProps谈起

在用hooks实现ComponentWillReceiveProps这个API时，我们往往会想到以下步骤:

1. cwrp 做了什么？它拦截某个接收到的Prop进行响应后再做更新。

2. 在hooks里，什么能做Props的拦截？事实上，没有hooks提供这一功能。

3. hooks的思想里，一切都是响应依赖的。当依赖的Props更新时，数据自行变化。由此，我们知道实现该API的关键是`useMemo,useCallback`这种缓存型API

   

