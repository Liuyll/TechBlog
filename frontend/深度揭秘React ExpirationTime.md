### 场景引入

考虑以下场景

```javascript
setState()
// after 5ms
setState()
// after 1ms
setState()*5
// after 30ms
setState
```

到这里，你应该明白`batchUpdateStrategy(批量更新策略)`，考虑以下，如何合并<b>该合并</b>的`setState`

#### 基础数学

##### 抹平时间差

如果我们把<b>优先级单元</b>定义为:每n(ms)的更新视为同一更新，加入批量更新组里	

```javascript
unit = n
expirationTime = ms / unit | 0
```

通过这样的算法，抹平`unit`范围内的不同的更新时间差

##### 根据某个基值四舍五入

这个就不解释了，初中数学

```javascript
function ceiling(num: number, precision: number): number {
  // |0 取整
  return (((num / precision) | 0) + 1) * precision;
}
```



## 核心计算公式

`ExpirationTime`的计算方法其实就是融合上述的两种数学方法，目的是把在可接受范围内的更新设为同一个`ExpirationTime`，以达到批量更新的目的

```javascript
function computeExpirationBucket(
  currentTime,
  expirationInMs,
  bucketSizeMs,
): ExpirationTime {
  return (
    MAGIC_NUMBER_OFFSET -
    ceiling(
      MAGIC_NUMBER_OFFSET - currentTime + expirationInMs / UNIT_SIZE,
      bucketSizeMs / UNIT_SIZE,
    )
  );
}
```

>  核心代码在：react/packages/react-reconciler/src/ReactFiberExpirationTime.js

```javascript
 if (isBatchingInteractiveUpdates) {
        // high priority
        expirationTime = computeInteractiveExpiration(currentTime);
      } else {
        // low priority
        expirationTime = computeAsyncExpiration(currentTime);
 }
```



省略掉其他代码后的核心计算方式就是上面，事实上只有一个变量，我们不需要关注无聊的多余代码，只需要记住提炼出来的一列计算公式即可

```
const MAGIC_NUMBER_OFFSET = 2

(((currentTime - MAGIC_NUMBER_OFFSET + var / 10) | 0) + 1) * (var2 / 10)
// var和var2由priority和dev决定
export const HIGH_PRIORITY_EXPIRATION = __DEV__ ? 500 : 150;
export const HIGH_PRIORITY_BATCH_SIZE = 100;

export const LOW_PRIORITY_EXPIRATION = 5000;
export const LOW_PRIORITY_BATCH_SIZE = 250;

```

这里顺便解释一下 - 2是什么意思：2只是作为一个定义的常量，在传入的时候加入了，这里减去而已。

至于为什么要加一个常量？可能只是作为一个偏移量而已。

### 直接给出结论

HIGH PRIORITY 10ms 间距为同一expiration time

LOW PRIORITY 25ms 间距为同一expiration time

ps:个人觉得这块代码毫无业务重现性，大家可以当做娱乐来读，它的数学性还是很强的。

## 有什么用？

<b>expirationTime是batchUpdate的核心，两个调用时间距离极小的setState将会被批量更新</b>

> 上述表达或许有些许误解，`expirationTime`并不直接影响`batchUpdate`,但`setState`会开启一个`transaction`，`transaction`会批量处理优先级相同的一批更新(优先级由`expirationTime`决定)

看看setState你就知道了

```javascript
enqueueSetState(inst, payload, callback) {
    const fiber = getInstance(inst);
    const currentTime = requestCurrentTimeForUpdate();
    const suspenseConfig = requestCurrentSuspenseConfig();
    const expirationTime = computeExpirationForFiber(
      currentTime,
      fiber,
      suspenseConfig,
    );

    const update = createUpdate(expirationTime, suspenseConfig);
    update.payload = payload;
    if (callback !== undefined && callback !== null) {
      if (__DEV__) {
        warnOnInvalidCallback(callback, 'setState');
      }
      update.callback = callback;
    }

    enqueueUpdate(fiber, update);
    scheduleWork(fiber, expirationTime); // 就是在这里使用的
  },
```

## 细节

#### current 真的是 Performance.now()吗？

并不是的，current的取值由函数`requestCurrentTimeForUpdate`计算而来

##### requestCurrentTimeForUpdate

+ 同一批更新，`current`计算为一个相同量
+ 新一批的更新，重计算当前时间



## 相关API

[performance.now()](https://developer.mozilla.org/en-US/docs/Web/API/Performance/now)

