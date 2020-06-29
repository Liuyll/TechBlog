![image-20200323112612901](/Users/liuyl/Library/Application Support/typora-user-images/image-20200323112612901.png)

从这张图里，我们能看到几个重要的点:

1. render开始时，会动态创建`fiberRoot`，`fiberRoot`会储存真实dom和用于操作的`rootFiber`,并形成fiber树。
2. 任何更新都会从`fiberRoot`开始
3. `ExpirationTime`标识是否有更新
4. 调度是通过`workLoop`这样一个函数持续进行，最小单位的调度是在`performUnitOfWork`里执行



#### 几个超级重要的函数

##### markUpdateTimeFromFiberToRoot(fiber,expirationTime)

+ 把更新层层上报至`fiberRoot`,(注意是`fiberRoot`而非`rootFiber`)

  ![image-20200323141328984](/Users/liuyl/Library/Application Support/typora-user-images/image-20200323141328984.png)

+ 更新`rootFiber`路径中全部节点的`expirationTime`(`current && workInProcess`)

  ```javascript
  // 若最大子优先级依然小于当前优先级，则提升优先级 
  if (alternate !== null && alternate.childExpirationTime < expirationTime) {
     alternate.childExpirationTime = expirationTime;
   }
  ```

+ 更新`fiberRoot`最新的优先级

##### checkForInterruption

+ 判断是否有高优先级任务进入，如果有，则打断当前调度

##### ensureRootIsScheduled

该函数确保`fiberRoot`正在调度中，并更新一些调度变量

+ 判断任务是否过期，若过期则立刻执行

  ```javascript
  if (lastExpiredTime !== NoWork) {
      root.callbackExpirationTime = Sync;
      root.callbackPriority = ImmediatePriority;
      root.callbackNode = scheduleSyncCallback(
        performSyncWorkOnRoot.bind(null, root),
      );
    }
  ```

+ 比较新旧任务优先级

+ 根据优先级加入到`taskQueue`

#####  scheduleInteractions

+ `sync`和`async`任务不直接通过该函数来区别，但它接收`expirationTime`作为参数，并交由下层函数做处理
+ 初次挂载时，进行一些初始化工作

##### scheduleWorkOnFiber

###### scheduleWorkOnParentPath



##### scheduleUpdateOnFiber

###### markUpdateTimeFromFiberToRoot

###### markRootUpdatedAtTime



##### beginWork

+ 替换memoizedProps



## commit

### commitRootImpl

> 这是commitRoot实现的函数，也是commit的全过程

#### commit的子阶段

+ beforeMutation

+ mutation 
+ layout === componentDidMount
+ effect





