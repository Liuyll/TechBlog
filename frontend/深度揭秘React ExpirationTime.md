## 核心计算公式

```javascript
function ceiling(num: number, precision: number): number {
  return (((num / precision) | 0) + 1) * precision;
}

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

*核心代码在：react/packages/react-reconciler/src/ReactFiberExpirationTime.js*

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
(((currentTime - 2 + var / 10) | 0) + 1) * (var2 / 10)
// var和var2由priority和dev决定
export const HIGH_PRIORITY_EXPIRATION = __DEV__ ? 500 : 150;
export const HIGH_PRIORITY_BATCH_SIZE = 100;

export const LOW_PRIORITY_EXPIRATION = 5000;
export const LOW_PRIORITY_BATCH_SIZE = 250;

```

这里顺便解释一下 - 2是什么意思：2只是作为一个定义的常量，在传入的时候加入了，这里减去而已。

至于为什么要加一个常量？可能只是作为一个偏移量而已。

### 直接给出结论

HIGH PRIORITY 10ms

LOW PRIORITY 25ms

ps:个人觉得这块代码毫无业务重现性，大家可以当做娱乐来读，它的数学性还是很强的。

## 有什么用？

<b>expirationTime是batchUpdate的核心，两个调用时间距离极小的setState将会被批量更新</b>

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



## SO？

react的一切都是为了整体服务，我会在其他地方详细的讲enqueneUpdate

而scheduleWork呢？我说过了，react一切都是为了整体服务，scheduleWork可不能单看一个函数就能明白的。



## 相关API

[performance.now()](https://developer.mozilla.org/en-US/docs/Web/API/Performance/now)

