## child_process

### shared socket

shared socket可以说是`child_process`类型的负载均衡服务器最最重要的东西，它运行多个子服务器共同监听一个端口，以完成负载均衡。

实际上，`shared socket`的底层是调用了`setsockopt`,并开启了`so_reuseport`选项

[reuseport](https://www.cnblogs.com/schips/p/12553321.html)实际上是操作系统底层来实现的端口复用，它允许某个服务器的地址正在`time_wait`时，其他服务器也能连接到这个地址。

当然，它也支持多个服务器在空闲状态时监听同个端口。

#### 负载均衡

我们都知道，tcp/ip连接的五元组是OS识别一个连接的唯一方法。

```
[protocol]:[remote_addr]:[remote_port]:[local_addr]:[local_port]
```

所以OS里绝不会允许两个相同的五元组出现。

虽然多个进程(worker)同时监听了一个端口，但在某个worker通过`connect`获取到这个连接时，其他的worker在连接时都会报错:`EADDRINUSE(给定的地址已被使用)`。此时，我们就实现了一个没有策略的负载均衡，让不同的服务器去处理请求。

#### 问题

很显然，`shared socket`是OS提供给我们的原生方法，它必然存在一些不适合场景的问题：

+ 负载均衡只能由操作系统来完成，而操作系统为了减少进程切换的消耗，倾向于同一进程一直处理
+ 



## cluster



