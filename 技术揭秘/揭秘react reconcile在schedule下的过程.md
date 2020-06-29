1. scheduleWork

2. scheduleWorkToRoot:

   此过程遍历造成更新的fiber到fiberroot节点，更新沿途节点自身的`expirationTime`，并更新上层fiber的`childExpirationTime`，拥有`childExpirationTime`表示子节点造成更新。

3. requestWork | scheduleUpdateOnFiber

4. addRootToSchedule:

   顾名思义，这个函数把fiberroot拉入调度



#### 在react legacy-event里的setState流程

![preview](https://pic3.zhimg.com/v2-225868f3e3e590bbab45336a6f3af9e6_r.jpg)

1. dispatchInteractiveEvent(这里是入口，不同事件触发类型不同)
2. dispatchEvent -> executeAllListeners
3. setState -> enqueueSetState