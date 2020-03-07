

## topics

### 概念

#### 备份

##### leader

#### leader

***ps:有了多实例，自然就有主从架构(分布式一般都是主从或者副本集架构)***

![leader架构](https://img-blog.csdn.net/20180626193308626?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0Nzk2OTgx/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

我们直接看上面这张图，这就是kafka的备份策略。看上去是不是很像mongo的读写分离部署架构呢？实际上有所差别。

kafka在Producer(Consumer)时为了保证消息不丢失，增加了follower节点(也就是我们说的备份节点)来保证消息同步，副本节点



让我们先来创建两个broker(kafka实例)

```
kafka-server-start -daemon properties1
kafka-server-start -daemon properties2
# 两个配置文件如何写，我已经提到过
```



我们执行一段代码：

```
kafka-topics --create --zookeeper 127.0.0.1:2181 --replication-factor 3 --partitions 1 --topic my-replicated-topic
```

报错了~

> Error while executing topic command : Replication factor: 3 larger than available brokers: 2.

我们看到了，备份的实质还是一个broker，于是我们把replication-factor改为2好了，因为我们只有两个broker

接下里看看我们的副本集在干嘛？

```
kafka-topics --describe --zookeeper localhost:2181 --topic my-replicated-topic
```



>show
>
>Topic:my-replicated-topic	PartitionCount:1	ReplicationFactor:2	Configs:
>	Topic: my-replicated-topic	Partition: 0	Leader: 0	Replicas: 0,1	Isr: 0,1

这个排版不是很好，不过我们还是能看到replicas为0和1，isr也为0和1	

##### ISR?

> ISR集合：ISR(In-Sync Replica)集合代表的是follow副本和leader副本消息相差不多的副本的集合。消息相差不到是一个比较模糊的概念。其实follow副本需要满足以下两个条件：
> 1：follow副本必须和zookeeper保持连接。
> 2：follow副本的最后的offset和leader中最新的数据之间的大小不能超过阈值。(也就是每个follow不能和leader副本消息相差太多)。
> Note：由于网络原因和宕机等原因免不了试follow副本会不能满足其中以上的条件（我理解的为任意一个条件），那么该follow副本将会被T出ISR集合中。举例：如果follow副本不满足上面条件2，此刻会被T出ISR集合，但是这个follow副本依然会进行数据的拉取，并且进行追赶，如果最新的offset和leader副本之间的数据量小于了阈值，那么该follow副本会从新加入到ISR集合中。

说白了，isr就是可以被信任的副本集



接下来我们试试副本集的功能

```
lsof -i:9092
kill -9 xxx 
# 关掉默认的leader producer
```

show

> Topic:my-replicated-topic	PartitionCount:1	ReplicationFactor:2	Configs:
> 	Topic: my-replicated-topic	Partition: 0	Leader: 1	Replicas: 0,1	Isr: 1

我们看到已经显示leader为1了，isr也为1了，因为0被踢出isr了，当然replicas还是不会变，当0上线的时候它就出来了。



## kafka实例

### Broker

老生常谈的一句话：一个broker就是一个kafka实例，kafka集群就是由一堆broker组成的。replications和partitions都要依赖这些broker



### 多实例

我们在做负载均衡时，首先就要开启kafka集群

#### 配置

```
broker.id=0
broker.id=2
# 我们通过broker.id来区分kafka，就像区分zookeeper一样
```

要看一个重要的属性，kafka将所有的log都存放于一个地方

```
log.dir=/usr/etc/logs/kafka/kafka-1-logs...kafka-n-logs
# 当然，不用指定每个kafka-logs的命名，系统会自动帮你创建
```



## Consumer

先让我们聊聊，consumer把消息藏在哪里了？

```
--zookeeper or --bootstrap-server
# zookeeper把消息存在zookeeper里，而后者把消息存在kafka里
# 新版kafka已经不支持--zookeeper了
```

#### 消费组

我个人认为，这是consumer最重要的概念。它跟partition完成了kafka的横向扩展，让负载均衡变得简单。

一些常识必须要先了解：

- 同一个消费组的成员不能消费同一个partition
- 一个消费者最多对应一个partition
- 不同消费组可以消费同一个partition
- 同一个消费组应该是同样逻辑的service





## Producer

***Of Course 有Consumer也就有Producer啦***

事实上，producer也采取和consumer一致的leader写入，follow备份的策略

#### 对多partition的producer，有哪几种写入方法？

```
# Partitioner type (default = 0, random = 1, cyclic = 2, keyed 3, custom = 4), default 0
# random 随机写入一个分片
# cyclic 轮询写入
# keyed 根据key策略写入

partitionerType: 2
```

#### 消息确认机制 

` requireAcks: 1,`



## Zookeeper

***这可是kafka最重要的东西啦！***

---



我们先看看zookeeper在背后为我们做了什么？

- 注册topics时，它记录了topic
- 注册producer时，它记录了producer
- 注册consumer时，它同样记录了consumer
- leader歇菜时，它发挥了重要作用

