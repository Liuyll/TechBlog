## 默认隔离级别RR下的锁

我们先讨论默认隔离级别RR下的锁是怎么给的。



### gap year

在select的时候，RR为了解决幻读问题，使用了gap year(最近下行区间锁) + next year(最近上行区间锁)封锁了一个范围



```
select * from test where num = 200
```

在RR下，这是个快照读，它不会上锁。

但是在`Serialiable`下，它会默认添加`for update`加上`gap lock`和`next lock`，这样有时候看上去就会锁到正无穷到负无穷，看似就是锁表了。