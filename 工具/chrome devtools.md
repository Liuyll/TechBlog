## Performance

### 一些关键选修开启步骤

#### layers panel

1. 点击设置按钮,打开`capture setting`

![image-20200216032340906](/Users/liuyl/Library/Application Support/typora-user-images/image-20200216032340906.png)

2. 勾选Enable advanced paint instrumentation

![image-20200216032428886](/Users/liuyl/Library/Application Support/typora-user-images/image-20200216032428886.png)

3. 点击frames

   ![image-20200216032515223](/Users/liuyl/Library/Application Support/typora-user-images/image-20200216032515223.png)

4. 出现layers panel

![image-20200216032530997](/Users/liuyl/Library/Application Support/typora-user-images/image-20200216032530997.png)



#### 实时渲染

1. ctrl+shift+p 打开命令行
2. 输入 Show rendering
3. 勾选FPS meter



#### force gc

1. 点击memory 旁边的垃圾箱



### 几个性能指标

> full performance report tools: lighthouse

+ FP (first print)
+ DCL
+ L
+ LCP (largest contentful print)
+ TBT (total blocking time)
+ TTI (time to interactive)
+ FMP (first meanful paint) 首次有效绘制

[参考](https://blog.csdn.net/c_kite/article/details/104237256)



### 使用参考

[performance usage](http://jartto.wang/2017/08/28/how-to-optimize-marker-of-AMap/)