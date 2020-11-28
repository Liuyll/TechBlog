> 本文将跳出基于条件假设的diff算法，并将探讨牺牲一定的时间复杂度，但达到最优解的diff方案。

#### 场景引入1

加入我们不考虑`vdom`的场景，而是从最简单的数组变化开始

```javascript
oldArray = [1,3,4,5]
newArray = [3,5,4,8]
```

设计一种算法，找到`oldArray`到`newArray`的最简路径

#### 概念

你必须知道一些概念和一些简单的算法

+ Longest Increasing Subsequence(LIS)

  [leetcode](https://leetcode-cn.com/problems/longest-increasing-subsequence/)

  > 为方便叙述，下文简称为LIS

+ 基于O(n2)的LIS算法

  这是一个基础的`动态规划`算法，它只需要常规套路就能解决

  <b>规定:dp[n]代表第n个数时最大的子序列</b>

  ```javascript
  let target = [8,2,7,4,5,10]
  let dp = []
  for(let i = 0;i < target.length;i++) {
    dp[i] = [target[i]]
    for(let j = 0;j < i;j++) {
      // 严格上升
      if(target[j] < target[i] && dp[j].length + 1 > dp[i].length) {
        dp[i] = dp[j].concat(target[i])
      }
    }
  }
  ```

+ 基于O(nlogn)的LIS算法

  我们重新定义dp[n]为<b>长度为n的上升子序列上升最小趋势队列</b>

  > 什么是上升最小趋势队列？
  >
  > 举个例子: target = `[2,4,10,7,6]`
  >
  > 则LIS有`[2,4,7]` || `[2,4,6]`
  >
  > 此时，`[2,4,6]`的上升趋势比`[2,4,7]`更迟缓，所以为上升最小趋势队列
  >
  > 我们为了贪心寻找更长的上升队列，所以越小的上升趋势，最终得到的上升队列越大

  此时，dp内部完全有序上升，符合`二分查找`的条件

  ```javascript
  for(let i = 0;i < target.length;i++) {
    let cur = target[i]
    if(cur > dp[dp.length - 1]) dp.push(cur)
    else {
      let [left,right] = [0,dp.length - 1]
      let insert = right
      while(left <= right) {
        let middle = left + ((right - left) >> 1)
        if(dp[middle] >= cur) {
          right = middle - 1
          insert = middle
        } else left = middle + 2
      }
      dp[insert] = cur
    }
  }
  ```

  

#### 场景引入2

假设有这样一个场景

```javascript
oldChild:['a','b','c','d']
newChild:['d','a','b','c']
```

通过对`newChild`的一次遍历，我们能轻松构建这样这样一个`map`

```javascript
newNodeMap = {
	'd':0,
	'a':1,
	'b':2,
	'c':3
}
```

不妨看看`newChild`在`oldChild`的位置

我们构建`newChildToOldChildIndex`来表示`newChild`对应`index`元素在`oldChild`的位置，我们规定，若新增元素，则`index = -1`

```javascript
newChildToOldChildIndex = [3,0,1,2] // LIS = [0,1,2]
```

你不妨按照上述步骤，移动一下不同的节点，再重新构建`newNodeMap`和`newChildToChildIndex`再来找找规律

比如:

```javascript
newChild:['b','c','a','d']
newNodeMap = {
	'd':3,
	'a':2,
	'b':0,
	'c':1
}
newChildToOldChildIndex = [1,2,0,3] // LIS = [2，3]
```

<b>结论:</b>

> 对于任意子节点的移动，最小元素操作方案是保持最大上升子序列(LIS)不变,再进行插入和删除



### 优化

> 仅对reconcileArray的情况进行某些边缘情况做理想条件下的优化，进行算法级别的优化和snabbdom/react 这类基于假设的vdom各有千秋。

依旧是第一个场景:

```javascript
const LIS = 											[0,1,2]
const newChildToOldChildIndex = [3,0,1,2]
```

为了让你看出算法的本质，我特地把`LIS`数组添加了一些缩进。

我们搞清楚`子序列`和`上升`这两个关键字，应该就能明白从后外前遍历更加轻松，接下里步骤如下:

1. 从后向前遍历`LIS`和`newChildToOldChildIndex`
2. 两者如果相同，不做任何操作，代表复用
3. `LIS`已为空或`newChildToOldChildIndex`与`LIS`对应元素不同，则直接复制当前元素到对应的`LIS`元素之前
4. 遍历完成

```javascript
// 在进行算法之前，如果LIS长度小于newChildToOldChildIndex，应该进行一些空元素的填充

let last
for(let i = newChildToOldChildIndex.length - 1;i > 0;) {
  let c1 = newChildToOldChildIndex.length[i]
  let c2 = LIS[i]
  
  if(c1 === c2) {
    i--
    last = i
  } else {
    LIS.splice(last-1,0,c1)
  }
}
```

#### 新增元素插入

```javascript
const LIS = 												 [0,1,2]
const newChildToOldChildIndex = [3,0,-1,1,2]
```



### react怎么做的？

React无法接受`O(n2) | O(nlogn)`的时间复杂度，它基于这样一种假设做了LIS:

<b>假设:绝大多数的子元素移动都是左右移动，而非首尾颠倒</b>

基于上述假设，react把第一个元素作为`上升序列`的起点，然后遍历`newChildren`:

1. 可上升的元素保持不变
2. 不可上升的元素直接插入

```javascript
const newChildToOldChildIndex = [4,1,2,5]
// LIS = [1,2,5]
// React LIS = [4,5]
```

附上源码

```javascript
function placeChild(
    newFiber: Fiber,
    lastPlacedIndex: number,
    newIndex: number,
  ): number {
    newFiber.index = newIndex;
    const current = newFiber.alternate;
    if (current !== null) {
      const oldIndex = current.index;
      if (oldIndex < lastPlacedIndex) {
        // This is a move.
        newFiber.effectTag = Placement;
        return lastPlacedIndex;
      } else {
        // This item can stay in place.
        return oldIndex;
      }
      // alternate不存在，对应上述 -1 index
    } else {
      // This is an insertion.
      newFiber.effectTag = Placement;
      return lastPlacedIndex;
    }
  }
```

#### 场景复现

存在以下两种`change`:

```javascript
origin = Array.from(Array(1e4),(_,i) => i+1)
first_change = () => {
  [origin[0],origin[1]] = [origin[1],origin[0]]
}
secend_change = () => {
  [origin[0],origin[1e4-1]] = [origin[1e4-1],origin[0]]
}
```

两次change的时间消耗是一致的吗？

告诉你答案，两种`change`时间消耗差距巨大

> 自己测试一下，会深入你对diff array optimize的理解

