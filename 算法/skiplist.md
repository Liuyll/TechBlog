## 前言

本章讲解跳表的那些事。



## 跳表的结构

![图片](https://img-blog.csdn.net/20180115125211841?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvREVSUkFOVENN/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

跳表是`O(logn)`复杂度的有序表查询结构。

跳表有以下特征：

+ 最大32层的储存（消耗额外空间）
+ n层元素必在n-1层存在



通过跳表的结构图，我们可以轻易的定义一个跳表节点的数据结构

```
struct SkipListNode {
	data int
	layer int
	up SkipList
	down SkipList
	left SkipList
	right SkipList
}
```

实际上，有些跳表的实现采用`forward和backround`来标识左右节点，两者是没有任何区别的。



当然，知道了跳表节点的结构，我们还必须定义跳表本身的哨兵节点(即meta节点)。

```
struct SkipList {
	start SkipListNode
	end SkipListNode
	layer int
	layer_length int
	
	r random
}
```

有几个需要注意的点：

+ 每层的跳表里，都存放一个随机序列`r`，这个`r`被用来决定`put`一个节点时，是否往上层节点添加。
+ `meta`节点本身不储存周围节点信息，依靠`start`和`end`两个节点通往上下层节点。



## 插入

跳表的插入是最复杂的操作，但对于`RBT`来说，也是非常非常的简单了。

它的pipeline大致如下：

1. 当前层(初始为0)找到对应位置(前驱节点小于，后继节点大于)后插入。
2. 当层`SkipList`的`r`是否大于(或小于)`0.5`，大于则往上层插入，反之停止。
3. 重复`1,2`步骤，直到停止或创建新层。
4. 如果要在顶端插入新层，那么执行新层创建程序。新层创建后终止，不再重复`2`。



```
public Integer put(String key, Integer value) {

    SkipListEntry p, q;
    int i = 0;

    // 查找适合插入的位子
    p = findEntry(key);

    // 如果跳跃表中存在含有key值的节点，则进行value的修改操作即可完成
    if(p.key.equals(key)) {
        Integer oldValue = p.value;
        p.value = value;
        return oldValue;
    }

    // 如果跳跃表中不存在含有key值的节点，则进行新增操作
    q = new SkipListEntry(key, value);
    q.left = p;
    q.right = p.right;
    p.right.left = q;
    p.right = q;

    // 再使用随机数决定是否要向更高level攀升
    while(r.nextDouble() < 0.5) {

        // 如果新元素的级别已经达到跳跃表的最大高度，则新建空白层
        if(i >= h) {
            addEmptyLevel();
        }

        // 从p向左扫描含有高层节点的节点
        while(p.up == null) {
            p = p.left;
        }
        p = p.up;

        // 新增和q指针指向的节点含有相同key值的节点对象
        // 这里需要注意的是除底层节点之外的节点对象是不需要value值的
        SkipListEntry z = new SkipListEntry(key, null);

        z.left = p;
        z.right = p.right;
        p.right.left = z;
        p.right = z;

        z.down = q;
        q.up = z;

        q = z;
        i = i + 1;
    }

    n = n + 1;

    // 返回null，没有旧节点的value值
    return null;
}

private void addEmptyLevel() {

    SkipListEntry p1, p2;

    p1 = new SkipListEntry(SkipListEntry.negInf, null);
    p2 = new SkipListEntry(SkipListEntry.posInf, null);

    p1.right = p2;
    p1.down = head;

    p2.left = p1;
    p2.down = tail;

    head.up = p1;
    tail.up = p2;

    head = p1;
    tail = p2;

    h = h + 1;
}
```



## 跳表的变种

实际上，在`redis`里面，`sorted set`为了支持一些业务功能，对跳表进行了变种的改动。

它可能需要知道一个元素具体在整张表的排名，为此引入了`span`这个字段。



## 参考

+ [跳跃表Skip List的原理和实现(Java)](https://blog.csdn.net/DERRANTCM/article/details/79063312)
+ [Redis 的底层数据结构(跳跃表)](https://juejin.im/post/6844903962429112334)

