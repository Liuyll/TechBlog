## kafka抗住流量的io秘密

kafka的写入过程

![image-20200615000120572](/Users/liuyl/Library/Application Support/typora-user-images/image-20200615000120572.png)

## 页缓存

#### Page Cache(OS cache)

这是操作系统本身的缓存机制

我们在写入磁盘时，需要先写入内存，即page Cache，然后再由操作系统决定写入磁盘的时机

其实，page cache是在disk cache和memory cache之间的一层。



[基础](https://tech.meituan.com/2017/05/19/about-desk-io.html)

#### 预读

在读取一个页面的时候，会额外读取少量页进入page Cache

##### 同步预读

如果新的页面不在page Cache(也就是不在预读页中)，此时表示非连续读取。

接下来就继续同步预读

##### 异步预读

如果新的页面位于page Cache中，则标识读取是连续读取。

那么接下里就开始异步预读



#### 回写

在page Cache中写入的数据会在适应时机写入到磁盘里



### 追加写

kafka的partition是通过追加写的机制来保持连续写入的



### 零拷贝

![image-20200620184522964](/Users/liuyl/Library/Application Support/typora-user-images/image-20200620184522964.png)

kafka利用零拷贝，把数据直接传输给socket，而不经过用户空间。

这样即使数据在磁盘上，也只是经过两次DMA传输。在完美情况下，page Cache将直接传输所有数据。



## 完美写入

![image-20200615000614642](/Users/liuyl/Library/Application Support/typora-user-images/image-20200615000614642.png)



一个完美的kafka集群是如上图所示的：

读写操作全部集中在`Page Cache`,此时的kafka集群是完全内存操作的，所以非常快速



## topic为什么要加入分区



![image-20200620131610591](/Users/liuyl/Library/Application Support/typora-user-images/image-20200620131610591.png)



当然，`partition`机制对用户是屏蔽的。在不主动设置key算法的前提下，用户只需要topic就能消费到对应的数据。而不用知道数据储存在哪里。

注意几个点:

+ 对每一个partition都是追加写入，比随机写入要快很多。所以不能添加太多分区，否则就基本上是随机写入了
+ kafka对每一个`partition`实现了ISR机制，保证了数据不丢失

+ topic划分的`partition`是物理划分的，在磁盘上不是连续的一块了





加入分区的几个重要原因：

+ 通过分区leader实现了消费数据的高并发性

  此时一个topic可以被多个消费者消费不同分区

+ 可以自动选择获取消息的分区，实现负载均衡





### topic的replication机制

为了保证消息不挂掉，把每个topic都放在不同的broker进行备份。以保证消息的可靠性。

需要注意的是，replication的个数 <= broker