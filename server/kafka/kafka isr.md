## kafka isr

isr机制是kafka保证高可靠的最关键之处，我们单独来一篇来分析它。



### AR

AR代表所有的副本集合

#### OSR

当一个副本落后于leader太多的时候，它会被踢出ISR（避免影响同步策略），加入到OSR里。

所以OSR是一个落后（当前不可用）的副本集。



### 非完全异步/非完全同步

isr最重要的一个特点就是非完全异步，也并非完全同步。

我们说mysql是`semi-synchronized`(半同步)机制的主从复制。意味着mysql的master节点需要主动向从节点同步信息(只需接受到一个slave节点即算复制完成)

#### 异步同步

就是说master节点写入后，将日志记录到binlog，然后由slave节点去pull这个log。至于什么时候复制呢，是slave节点来决定的。所以master节点根本不需要等待slave节点同步完。



#### acks

acks实际上不是控制副本集同步，而是控制Producer响应一条消息的策略

+ 0：完全异步（连master都不保证）
+ 1：master同步(半同步)（保证master）
+ 2：完全同步（全部副本响应，最小两个）

对应broker本身，都是需要所有的follower响应才能算commit



### 选举策略



### 消息commit

上面提到过了，require.acks = 2时，ISR的每个follow都响应后，此消息才能被提交。

这个响应可能是两种：同步成功或被踢出ISR

同步成功非常好理解，那被踢出ISR才是kafka要实现的重点

实际上，被踢出ISR会有以下两种情况：

+ 落后消息大于阈值:replica.lag.max.messages 
+ 心跳时间大于阈值:replica.lag.time.max

实际上，最重要的是需要一套检查策略，知道每个副本是否符合了上述两个条件。



### replicaset leader

实际上，我们并没有要求一个`partition`只能有一个`replica`，也可以实现一个分区对应多个副本的情况。

此时副本集需要选举一个`leader`副本，分区`leader`只需要跟该副本集的`leader`节点同步即可。



### copy on writes

`read on writes`代表着写入后直接被读取到。

实际上，kafka为了保证读一致性，只能读到isr里最落后的一个副本LEO（最落后就能保证其他任何副本消息都不会落后于它）

> LEO()

#### HW(HighWaterMark)

hw意味着



