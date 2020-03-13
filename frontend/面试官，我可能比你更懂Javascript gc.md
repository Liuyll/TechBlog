> 我们此次聊的都是heap gc，栈gc是依靠ESP下移，无效内存复用的方法来回收
>
> 本文主要深入v8 gc的具体细节，涉及大量伪码和概念，不适合小白阅读

在聊gc之前，我们先聊一些必须知道的概念：

<b>The Generational Hypothesis(世代假说)</b> 

1. 大多数变量是才分配不久就会被清除的
2. 存在少数很长时间不死，甚至是伴随整个程序运行周期的变量



针对`The Generational Hypothesis`，js提出了这样的解决方案：

1. 对大多数变量执行新生代处理,我们把这部分回收叫做<b>副垃圾回收器</b>
2. 对持久存活变量做老生代处理,我们把这部分回收叫做<b>主垃圾回收器</b>



#### mutator

你无需知道`mutator`的具体概念，你仅需了解它带来的行为:

+ 生成新对象
+ 改变旧对象引用

你可以理解为，`mutator`是程序往下执行的过程，gc的存在就是为了应付`mutator`产生的垃圾



#### gc常用算法

##### 跟踪回收

+ 标记清除:

  具体细节后续章节会谈到

  它的好处是不用额外的内存，只需要对已有对象的`head`打上标识即可，最后进行统一的清理

+ 复制追踪：

  它需要一块新空间来进行复制存活对象，所以不需要进行统一清理，它的好处是速度快(一次遍历)，在垃圾比例大的情况下更具优势

事实上，javascript中新旧生代的gc全部采用了跟踪算法



##### 引用计数

跟踪算法需要单独的时间进行gc标记，引用计数缺不需要

它只需要标识每个对象的引用，在需要gc的时候干掉没有引用的对象即可

当然，它也存在`circular reference`等问题，这里具体不介绍引用计数



### V8堆内存

![image-20200313110807958](/Users/liuyl/Desktop/pic/image-20200313110807958.png)

> 本文更偏重gc的技术细节，广泛的抽象概念这里只做简单介绍

如上图所示，v8分为新/旧生代，我在世代假说里已经提过这种分代的好处

为了叙述方便，这里介绍一下新生代空间

+ 新生代分为了to/from 两个空间，分别对应上图的对象区域和空闲区域
+ 新生代只有1-8M的内存
+ 新生代空间只储存很快就消亡的对象

### 新生代gc之cheney

> cheney是scavenge算法的实现

> 上面已经提到过，本文旨在剖析技术细节，对普通概念可以参考其他文章

cheney的大致过程如下：

![image-20200313111821601](/Users/liuyl/Library/Application Support/typora-user-images/image-20200313111821601.png)



接下里是重头戏，我们看看cheney是怎么实现的？

###### 核心代码：copy

```c++
copying(){
    scan = $free = $to_start 
    for(r : $roots)
        *r = copy(*r)
		
		// scan用于搜索已复制到to空间对象的指针
		// free是前倾指针，用于标识头部
    // scan和free都是to空间的指针
    while(scan != $free) 
        for(child : children(scan))
            *child = copy(*child) 
        scan += children(scan).size
    
    swap($from_start, $to_start)
}

copy(obj){
    // scan catch up $free ? true : false
    if(is_pointer_to_heap(obj.forwarding, $to_start) == FALSE)
        copy_data($free, obj, obj.size) 
        obj.forwarding = $free
        $free += obj.size
    return obj.forwarding
}
```

![img](https://sakura-1252236262.cos.ap-beijing.myqcloud.com/2018-08-31-163100.png)

![img](https://sakura-1252236262.cos.ap-beijing.myqcloud.com/2018-08-31-163127.png)

+ 所谓的`存活检查`事实上并没有进行"检查",而是通过`可达性分析`算法直接迭代复制了由root(以js的内存模型就是栈上变量)指向的节点和其子节点

+ 上图就是执行下面语句后的to空间，此时free指针领先scan

  ```c++
  for(r:roots){
  	*r = copy(*r)
  }
  ```

  



###### 迭代复制

![img](https://sakura-1252236262.cos.ap-beijing.myqcloud.com/2018-08-31-163619.png)

+ 上图是迭代复制根引用的子对象后的to空间

  ```c++
  while($free != scan){
    	for(child:children(scan)) {
        // $free right move
    		*child = copy(*child)
  		}
    	scan += children(scan).size
  }
  ```

+ 结合上图，我们可以清晰想象到结束条件发生在<b>scan指针搜索的对象没有子引用对象可迭代复制时</b>



###### 算法原理

事实上，scan和free指针最终一定会相等。

+ scan指针最初会落后free指针，是因为会先复制根块
+ scan指针逐渐追上free指针，是因为scan指针会遍历所有的复制块，以达到迭代复制子块的效果

至此，存活对象的检查就接受，`copying`函数执行完毕

![img](https://sakura-1252236262.cos.ap-beijing.myqcloud.com/2018-08-31-163705.png)

#####其他技术细节

+ 上述gc过程是副回收器进行的，它一旦扫描到对象是老生代里的，就会直接跳过，老生代对象由主回收器执行major gc
+ 存在一个问题，如果某个新生代对象唯一的被引用是老生代里的对象，那么它不会被复制

为解决上述问题，我们引入一个概念:

###### 记录集

在`mutator`过程中，为B对象添加了对A对象的引用，如果此时B对象为老生代对象，A对象为新生代对象，则将其B(发出引用的对象)加入记录集，进行副回收时，扫描记录集获得A引用。



### 新旧生代过渡

#### promotion 晋升

##### when?

+ 新生代目标对象已经历一次副回收器gc
+ `copy`过程中，`to`空间超过25%使用量

上述两种情况，新生代对象都会晋升为老生代对象

##### how?

我们需要了解一下晋升的过程，它实际跟`cheney`里的copy相似，这不是一个简单的指针空间移动，而是复制了一份新对象到老生代。

##### 记录集更新

上面已经提到过，记录集储存的并非是新生代对象，而是发起引用的老生代对象，新生代在复制到老生代时，引用也会同步转换至老生代。



### 老生代gc之标记-清除

> 不建议你以网上的方法理解标记-清除或者引用计数等gc，对gc应该以引擎的角度看，而不是位于代码解析的角度看

#### 标记-清除

在谈后续算法之前，我们必须先了解一下gc界的大牛----标记-清除算法

###### 一些常识

我们必须先知道，一个对象在内存中存在`head`和`field`两个域，`head`储存了它的信息(包括种类，大小等)，我们所有的标识等操作都是在`head`域进行的。你可以把`head`域看做是一个对象的索引。



##### 过程

1. 获取root reference(你可以理解为全局变量)
2. 通过root reference向下寻找可获取的引用，并进行标记
3. 再次遍历对象，找到无标记的垃圾进行回收

你不难发现，标记-清除算法是不可中断的，它是一个递归的过程。但为了保证其余工作正常，我们继续介绍另一个算法



#### tri-color marking(三色标记法)

如果你熟悉react的`async reconcile`过程，你应该能明白如何把递归的更新转化为迭代的更新，再利用加速更新标识来跳过已更新的fiber。三色增量标记的原理完全跟它相同。

`tri-color marking`将对象分为三个颜色：

+ 白色(未遍历)
+ 灰色(遍历部分，等待遍历子引用)
+ 黑色(子引用也全部被处理)

显然，在gc后只存在两种颜色:白或黑

此时，白色意味着垃圾，因为它从未被可达性算法遍历，而黑色就是存活对象

##### 技术细节

###### 颜色如何涂

1. 对根引用直接涂灰

2. 把根引用子节点涂灰(并不需要递归处理完子节点)，然后涂黑根引用

3. 对灰节点递归涂色

4. 串联白色节点，进行删除

###### 存在的问题

![image-20200313102753426](/Users/liuyl/Desktop/pic/image-20200313102753426.png)

我们先简单的介绍一下上述几幅图:

+ a: A是根引用，B是A的子引用，此时执行对应涂色1,2步骤
+ a-b间隙: 执行mutator，此时变量引用更改，B的子引用切换到A
+ b: 恢复标记阶段，A已被涂黑，无法进行查找，B对C的引用已经丧失
+ c: C对象明显是存活对象，此时却未被任何一次标记

不难发现，C对象被标记遗漏，这个问题很普遍，`mutator`动态修改了引用，导致无法被遍历到，解决它的思路有很多种，在`redux`里使用了`double listeners`双缓冲队列来监听事件触发移除时的队列变更，这里的处理思路与它很像，我们引用一个新概念

###### write barrier(写屏障)

直接看code

```c++
void write_barrier($obj,field,newField) {
  // obj是否是老生代对象，newField是否是新生代对象，obj是否存在于记录集
  if($obj >= $oldStart && newField <= $oldStart && $obj.ismemorable = false) {
    addMemorableSet($old)
    $obj.ismemorable = true
  }
  // 新引用直接涂灰
  if(!newField.mark) {
    newField.color = markingObj(newField,TRI_COLOR_MARKING_GREY)
    newField.mark = true
  }
  // 修改引用
  *yield = $newObj.yield
}

```

![image-20200313104259321](/Users/liuyl/Desktop/pic/image-20200313104259321.png)

> 处理的关键在于: 对`mutator`过程中变更的新引用直接涂灰，这个过程单独处理，不由父节点迭代而来。

###### 涂色阶段

增量标记其实分为了三个部分：

+ 根引用查找
+ 标记 (异步标记 marking -> mutator -> marking)
+ 清除 (同步清除)

> 确实，它跟react concurrent-mode的diff and commit是何其的相似啊



#### 标记-压缩

标记-压缩是标记-清除算法的衍生算法，旨在移除gc后的内存空隙，有兴趣可以自行了解



**引用：**

1. [v8 GC](http://eternalsakura13.com/2018/09/01/v8_GC/)
2. [可达性分析算法](https://www.jianshu.com/p/115899be002b)
3. [垃圾回收](https://www.kawabangga.com/posts/1336)