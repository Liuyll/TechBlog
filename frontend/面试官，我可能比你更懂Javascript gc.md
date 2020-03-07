### scavenge

##### cheney 

###### 存活检查

​	其实`存活检查`这是术语有些令人误解，我最初看到这个词的时候，也以为是可达性分析或者引用计数一样的检查方案。可是转念一想，如果在新生代调用上述的存活性检查算法，那分代GC就没必要存在了。

  

我们看看cheney是怎么实现的？

###### 核心代码：copy

```
copying(){
    scan = $free = $to_start 
    for(r : $roots)
        *r = copy(*r)
		
		// scan是用于搜索复制完成对象的指针
		// free是指向块开头的指针
    while(scan != $free) 
        for(child : children(scan))
            *child = copy(*child) 
        scan += scan.size
    
    swap($from_start, $to_start)
}
```

![img](https://sakura-1252236262.cos.ap-beijing.myqcloud.com/2018-08-31-163100.png)

![img](https://sakura-1252236262.cos.ap-beijing.myqcloud.com/2018-08-31-163127.png)

​	可见所谓的`存活检查`事实上并没有进行"检查",而是通过`可达性分析`算法直接迭代复制了由root(以js的内存模型就是栈上变量)指向的节点和其子节点

###### 迭代复制

![img](https://sakura-1252236262.cos.ap-beijing.myqcloud.com/2018-08-31-163619.png)

###### 算法原理

事实上，scan和free指针最终一定会相等。

+ scan指针最初会落后free指针，是因为会先复制根块
+ scan指针逐渐追上free指针，是因为scan指针会遍历所有的复制块，以达到迭代复制子块的效果

至此，存活对象的检查就接受，`copying`函数执行完毕

![img](https://sakura-1252236262.cos.ap-beijing.myqcloud.com/2018-08-31-163705.png)









**引用：**

1. [v8 GC](http://eternalsakura13.com/2018/09/01/v8_GC/)
2. [可达性分析算法](https://www.jianshu.com/p/115899be002b)