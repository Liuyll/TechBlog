## 前言

mysql的`master-slave`架构实际上也在其他很多地方都存在，我们可以把这种容错的措施分为两种：

+ `master-slave`主从架构
+ `ISR(replicaset)`副本集架构

一般来说，我们在主从架构上做读写分离，在副本集架构上做数据备份。

所以这里讲的`mysql`里的架构，实际上在很多相同的主从架构里都是一致的。



## 同步

`mysql`的主从同步实际上是采用独特的`binlog`日志来进行的，每进行一个事务，都会把信息写入到`binlog`里。此时采用同步的方式通知从节点拉取`binlog`。

从节点有两个线程：

+ `binlog`拉取线程，该线程拉取`binlog`并写入到本地的`relay log`里
+ 本地复现线程，该线程通过循环读取`relay-log`复现`master`线程的`binlog`操作

![图片](https://img-blog.csdn.net/20180313225542529)



### 写入与同步故障

通过`WAL`写入最大的麻烦就是硬件突然失能导致的同步故障。假如`master`进程还未同步`redo`日志到磁盘上就结束了事务，此时电脑断电，内存数据丢失，那么这条数据就丢失了。

`innodb`引入了配置项来处理`redo`日志写入磁盘的时间`innodb_flush_log_at_trx_commit`:

+ 0 ：每秒 write cache & flush disk

+ 1 ：每次commit都 write cache & flush disk

+ 2 ：每次commit都 write cache，然后根据innodb_flush_log_at_timeout（默认为1s）时间 flush disk

实际上为1时最安全，但性能最坏。为2比较均衡。

> 当然，也可以在代码层面提升1的性能，让多次操作只进行一次`commit`即可。



同时，我们上面提到了主从同步的关键是`binlog`，那么写入`binlog`到磁盘的时间也尤为关键。

我们知道mysql默认的是写入到`redo log`，再在事务提交的时候，一股脑把所有的`redo log`提交到`bin_log`等待同步。



mysql提供了`sync_binlog`来处理：

+ 0：由文件系统处理`binlog`何时进入到磁盘
+ 1：每次`commit`均提交`binlog`到磁盘

为1时能保证最多只有一个事务的数据丢失



### 同步方案

`mysql`的`binlog`分为几种形式：

+ SBR(基于sql语句的日志)：它记录每个sql语句
+ RBR(基于行变化的日志)：它以每个行变化为单位产生日志，一条sql语句可能会产生多个行变化
+ MBR：混合SBR和RBR的方式



### WAL

实际上，上述所说的同步机制都是对当前的机制进行分析，它的原理实际上还是WAL。

像mysql这种数据库，写入数据需要在磁盘上寻址。我们希望有一种机制可以不阻塞用户的行为(写入后查询等)，然后又不丢失写入的数据。此时，我们引入一份冗余的日志来表示数据库的操作行为，然后由一个后台线程异步把行为同步到磁盘（同步的时机可以被配置）。



我们前几篇文章主要是讲到了怎么写`redo log`和`undo log`，但没有讲清楚WAL的实现细节。

我们先考虑一个细节：

+ `redo log`写入到磁盘的时机(这个操作通常被称为`checkpoint`)。



#### 组提交

实际上，在多个事务并发执行的时候，刷盘操作会成为一个瓶颈。最好是把多个刷盘操作排队变成一次性刷多个的行为，这种优化叫做组提交(group commit)。

我们在`commit`阶段刷盘`redo log`时，将各个事务合并成一个直接刷到磁盘上。



当然我们可以聊到一些实现的细节。

##### 可以进行组提交的事务

在`redo log`里，每个`redo log`都会有唯一一个`LSN`(`log sequence number`)，以保证`redo log`不会重复。而每一个事务会产生一个或多个的`redo log`，当一个事务获取到锁时，它可以提交比它的最大`LSN`小的所有事务。



#### `redo log` checkpoint

这是个很有价值的问题，实际上`redo log`并不是一个无限大的文件，它必须要在合理的时机被删除，以保证后续的更新能顺利的写入。

可以想到几种必须被`flush`的场景：

+ 内存紧张，必须清除部分缓存页(mysql并没有直接使用操作系统的`page cache`，而是通过自己实现的`buffer pool`来控制)。
+ `redo log`到达配置的极限，此时无法再进行写入。

这两种情况是最紧张的，我们必须要进行刷页。

> 刷盘的过程会经历多道缓存，包括innodb自己的buffer pool，还有vfs的几道缓存。具体可以[参考](https://www.cnblogs.com/zengkefu/p/5674586.html)





### 2PC

我们的主从同步(主备同步)都依靠的是`bin_log`，而我们的写入依赖的却是`WAL`的`redo log`。这两个操作不是原子性的操作，很可能出现以下情况:

+ `bin_log`落盘而`redo log`未落盘，此时从领先于主
+ `redo log`落盘而`bin_log`未落盘，此时从落后于主

显然两种情况都是因为落盘时物理机出现故障导致的，我们需要2pc把两个操作封装为原子性。



按照2pc设计的通用思路，我们设计不同的状态

1. before prepare: `redo log`和`binlog`均在内存
2. prepare: 回滚段为`prepare`，刷盘
3. commit: 回滚段重置为`commit`，刷盘`binlog`
4. end: 完成事务



按照这个状态转移，我们很容易发现在`3`的时候可能出现问题：回滚段重置状态跟`binlog`落盘无法用原子操作来完成

>为解决这个问题，引入binlog的一个完整性判断方法
>
>- statement 格式的 binlog，最后会有 COMMIT；
>- row 格式的 binlog，最后会有一个 XID event

实际上，在恢复的过程中，会比较`binlog`的`xid`和`redo log`的`xid`，如果`binlog`的`xid`不存在，那么意味着落盘失败。