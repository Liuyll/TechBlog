> 注：该文不谈vdom的跨平台优势，只关心vdom的性能优势

## 原生慢吗？

### 原生真的慢吗？

不慢，任何对等操作下，原生都会快于vdom，因为vdom会走无用的diff。



### 原生真的不慢吗？

在一次render下，原生确实不慢。

但多次render呢？我举个例子：1000个元素的更新，10次render，每次100个元素和1次render，每次1000个元素，两者真的速度相同吗？

***上面提到过，原生dom操作在对等条件下一定快于vdom，但这里并非对等操作***



我们不得不提到，浏览器内核是如何重置innerHTML的更新的。

#### 内核

> 先附上一张Aggregated Time的查看方式表
>
> [详情](https://developers.google.com/web/tools/chrome-devtools/evaluate-performance/performance-reference?hl=zh-CN)
>
> ![image-20191203200936123](/Users/liuyl/Library/Application Support/typora-user-images/image-20191203200936123.png)

---

![img](https://pic4.zhimg.com/80/93799d4434552f68d7c8fcf9a39bdf6b_hd.jpg)

> 该图引用自知乎

上图里可以清晰看到，脚本执行的时间远远小于<b>因脚本执行而进行的其他操作的时间</b>

---

![img](https://pic3.zhimg.com/80/3ffd05a65cdafc445226abfb79a04be3_hd.jpg)

> 这是一次解析1000个元素的情况，loading时间减少10倍



##### 为什么？

因为innerHTML置换字符串很简单，但是建立dom上下文就复杂了。10次解析，就会建立10个上下文，而这个上下文是完全可以通过batch来复用的。



##### innerHTML和textContent

修改innerHTML后，浏览器会干掉style，这导致rendering时间上升，并非最小dom更新策略。



## 为什么快？

+ DIFF
+ BATCHUPDATE
+ 收集 + 对比 



我上文提到过，更新相同数量的元素下，多次render的时间耗费极大(Loading事件消费)。所以批处理 + diff算法完成了更新的收集，也就是说我可以在一个render周期里渲染全部元素了。(<b>不过,Schedule算法证明了这不是理想情况，这会导致帧卡顿</b>,但这并非在我们的讨论范围内)

> 注：不要误解，非vdom也可以进行batchUpdate，不管是vue的依赖标记还是react的expirationTime都可以在非vdom的环境下实现，但由于无法掌握整体的dom tree，无法进行diff算法



#### 列表复用

还有一个重要的原因是列表元素的复用，在绑定dom到依赖的年代，我们很难做列表元素的复用，还是需要框架层进行判断是否是可复用元素。



## 为什么慢？

+ diff算法的遍历同样需要时间
+ 双fiber的对比修改策略(vue里体现为依赖收集)耗费时间。

+ diff算法的副作用：考虑下这种情况

![img](https://static001.infoq.cn/resource/image/e1/e3/e1a32e640909e276d6f9c6ac9c1da4e3.png)

> diff算法在比较同一节点无果后直接删除或者新建，这是它完成O(n)复杂度的最重要假设之一
>
> 为了一次遍历vdom tree就完成更新，它没办法复用A节点