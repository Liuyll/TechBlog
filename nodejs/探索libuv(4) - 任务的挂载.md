## 概述

本章主要剖析的是worker是如何处理任务，以及任务是如何提交到线程池的。



## 探索

### 独特的queue

libuv的很多操作都需要用到双向的`queue`，而它的queue实现又很有意思，我们先仔细研究下它。

```
typedef void *QUEUE[2]
```

可能大家会很奇怪，为什么不以常规的struct来描述队列，而是用一个指针数组。

实际上`QUEUE[0]`标识next，`QUEUE[1]`标识prev

```
#define QUEUE_NEXT(q)       (*(QUEUE **) &((*(q))[0]))
#define QUEUE_PREV(q)       (*(QUEUE **) &((*(q))[1]))
```

读到这里时，读者肯定会有大大的疑惑。这个跟直接取`((*(q))[0]`有什么区别吗？

`&((*(q))[0]))`取到了`q[0]`这个`void *`，那么如果想对它进行赋值呢？

我们简化以下场景，考虑以下代码

```
int i = 0;
(char) i = 2;
(*(char *)(&i)) = 3; // true
```

同理，我们需要对`void *`赋值的话，也需要`*((void **) &q)`来进行赋值。

当然取值就随便了，两种方法都是可以的。

所以这句话主要是为了方便赋值和取值两个操作。

#### 操作

想象读者都是熟悉链表操作的同学，这里不详解了

头插

```
#define QUEUE_ADD(h, n)                                                       
  do {                                                                        
    QUEUE_PREV_NEXT(h) = QUEUE_NEXT(n);                                       
    QUEUE_NEXT_PREV(n) = QUEUE_PREV(h);                                       
    QUEUE_PREV(h) = QUEUE_PREV(n);                                            
    QUEUE_PREV_NEXT(h) = (h);                                                 
  }                                                                           
  while (0)
```

尾插

```
#define QUEUE_INSERT_TAIL(h, q)                                               \
  do {                                                                        \
    QUEUE_NEXT(q) = (h);                                                      \
    QUEUE_PREV(q) = QUEUE_PREV(h);                                            \
    QUEUE_PREV_NEXT(q) = (q);                                                 \
    QUEUE_PREV(h) = (q);                                                      \
  }                                                                           \
  while (0)
```

#### 神奇的取值

你肯定发现了，`QUEUE`只有前驱和后继节点，但是没有数据储存域。它是如何储存数据的呢？

如果你熟悉`linux`内核的话，应该知道`linux`也定义了一个精简的双端队列。

```
struct list_head {
  struct list_head *next, *prev;
};
```

看似与我们的`QUEUE`完全类似啊，那在linux里是如何取值的呢？

实际上，我们需要把这个队列嵌入到一个结构体里，作为结构体的一个成员。然后通过手动计算偏移量去获得结构体的地址，然后再去取得对应的成员值。

类似这样

```
struct book {
  int sn;
  char name[NAMESIZE];
  int price;
  struct list_head node;   //在内核里，链表结构体放在最后
}
```

熟悉c语言的同学，应该有一定的思路了。我们只需要从`node`所在的位置，减去<b>结构体首地址到node成员的偏移量</b>就能获得结构体的首地址了

```
&struct = &node - (&node - &struct) 
```

bingo,简单的小学减法。

问题是，如何取得这个偏移量offset呢？

实际上，在c语言里，获取结构体成员的偏移量是很简单的。

```
struct Demo {
	int a; //0
	char b; // 4
	int c; //5
}

&((struct Demo*)0 -> b) // 4
```

是的，我们只需要把0地址强制转换为结构体指针，就能获取成员变量的偏移量。但显而易见，无法通过这种方法去取值。

至此，我们的`offsetof`函数也有了定义。它可以取到任何一个成员变量到node的偏差量。



系统以及把上述操作封装为`container_of`和`offsetof`，我们直接调用就行了

+ `container_of`：通过一个成员变量的地址求出结构体的首地址
+ `offsetof`：求出一个成员变量地址到结构体首地址的偏移

取值

```
#define QUEUE_DATA(ptr, type, field)                                          
  ((type *) ((char *) (ptr) - offsetof(type, field)))
```

好了，队列的操作就到这里。



### 提交任务

在第一章已经提到过，任务的提交是通过`uv_work_submit`来进行的.

```
void uv__work_submit(uv_loop_t* loop,
                     struct uv__work* w,
                     enum uv__work_kind kind,
                     void (*work)(struct uv__work* w),
                     void (*done)(struct uv__work* w, int status)) {
  // 上篇主要讲的这里 初始化线程池等
  uv_once(&once, init_once);
  w->loop = loop;
  w->work = work;
  w->done = done;
  post(&w->wq, kind);
}
```

继续探究`post`函数

```c
static void post(QUEUE* q, enum uv__work_kind kind) {
  // 因为存在队列插入操作 需要加锁
  uv_mutex_lock(&mutex);
  if (kind == UV__WORK_SLOW_IO) {
    //慢i/o 跳过...
  }
	
  // 插入uv__work -> wq 到 loop -> wq
  QUEUE_INSERT_TAIL(&wq, q);
  // 跳过slow I/O
}
```

可以发现，`post`把`uv_work`上的`wq`挂载到了全局的`wq`上，那么worker在取任务的时候，是否就不需要从`uv_work -> wq`上取了呢？

我们看下worker的工作函数

不过，我们这里先不看对slow i/o任务的处理。快慢i/o的调度，我们后续单独来讲。

```
static void worker(void* arg) {
  struct uv__work* w;
  QUEUE* q;
  int is_slow_work;

  uv_sem_post((uv_sem_t*) arg);
  arg = NULL;

  uv_mutex_lock(&mutex);
  	for (;;) {
  		...
    	// 利用条件变量和互斥锁，等待快i/o任务进入
    }

    q = QUEUE_HEAD(&wq);

    QUEUE_REMOVE(q);
    QUEUE_INIT(q);  /* Signal uv_cancel() that the work req is executing. */

    uv_mutex_unlock(&mutex);
		
		// 从全局wq上取一个任务下来
    w = QUEUE_DATA(q, struct uv__work, wq);
    w->work(w);

    uv_mutex_lock(&w->loop->wq_mutex);
    w->work = NULL;  /* Signal uv_cancel() that the work req is done
                        executing. */
		// 把执行完的w->wq 再放进wq队列等待回调
    QUEUE_INSERT_TAIL(&w->loop->wq, &w->wq);
    // 唤醒主线程
    uv_async_send(&w->loop->wq_async);
    uv_mutex_unlock(&w->loop->wq_mutex);

    /* Lock `mutex` since that is expected at the start of the next
     * iteration. */
    uv_mutex_lock(&mutex);
  }
}
```

确实如我们所料，这里是从全局的`wq`上取下来执行，而不是从`w`上来取。

#### wq

根据前面几章的讲解，读者有可能混淆出现的几个wq

+ loop -> wq：`uv__work_done`触发回调用，worker在执行完任务后会把对应的`uv__work -> wq`插到这里面
+ wq：仅在`init_thread`时被初始化，线程池都共用这个`wq`
+ req -> wq：这个是`uv__work`上的wq，是需要执行的工作队列，会被插入到全局的`wq`里



#### 快慢i/o调度

上面省略了对快慢i/o的调度策略，这里详细来讲一下。

```
 while (QUEUE_EMPTY(&wq) ||
           (QUEUE_HEAD(&wq) == &run_slow_work_message &&
            QUEUE_NEXT(&run_slow_work_message) == &wq &&
            slow_io_work_running >= slow_work_thread_threshold())) {
      idle_threads += 1;
      uv_cond_wait(&cond, &mutex);
      idle_threads -= 1;
    }
```

这里假设读者都很清晰`unix`下的条件变量和互斥锁的用法。`uv_cond_wait`实际是`pthread_cond_wait`的一层封装，这里不再继续深入了。

我们剖析一下`worker`等待的条件:

1. `wq`队列空了，表示没有任务需要执行
2. 下一个任务是慢i/o任务，且正在运行的慢任务数量达到慢i/o任务阈值

那么这个`run_slow_work_message`又是什么呢？

重新回到我们的`post`函数里(完整版)

```
static void post(QUEUE* q, enum uv__work_kind kind) {
  uv_mutex_lock(&mutex);
  if (kind == UV__WORK_SLOW_IO) {
  	// 插入慢/io 队列
    QUEUE_INSERT_TAIL(&slow_io_pending_wq, q);
    
    // 慢/io 已经进入调度状态，不再通过post函数调度。而是通过正在调度中的慢i/o函数进行调度，逻辑下面会讲解
    if (!QUEUE_EMPTY(&run_slow_work_message)) {
      uv_mutex_unlock(&mutex);
      return;
    }
    // 插入刚才提到的message，标识慢i/o调度
    q = &run_slow_work_message;
  }

  QUEUE_INSERT_TAIL(&wq, q);
  // 非慢i/o 调度中，唤醒worker准备执行慢i/o调度
  if (idle_threads > 0)
    uv_cond_signal(&cond);
  uv_mutex_unlock(&mutex);
}
```



下面进入慢i/o 的执行过程，也就是跳出while循环的部分

```
  if (q == &run_slow_work_message) {
      // 再次判断慢i/o 任务数量是否超过阈值
      if (slow_io_work_running >= slow_work_thread_threshold()) {
        QUEUE_INSERT_TAIL(&wq, q);
        continue;
      }

      // 在执行前便被uv__work_cancel关掉任务，此时跳过本轮
      if (QUEUE_EMPTY(&slow_io_pending_wq))
        continue;

      is_slow_work = 1;
      slow_io_work_running++;

      q = QUEUE_HEAD(&slow_io_pending_wq);
      QUEUE_REMOVE(q);
      QUEUE_INIT(q);

			// 如果还有慢i/o 任务，再次往wq里加入slow_message，让其他worker进行调度
      if (!QUEUE_EMPTY(&slow_io_pending_wq)) {
        QUEUE_INSERT_TAIL(&wq, &run_slow_work_message);
        if (idle_threads > 0)
          uv_cond_signal(&cond);
      }
  }
```

可以看到，上面代码的最后几行就是调度上面`post`函数的这个部分

```
if (!QUEUE_EMPTY(&run_slow_work_message)) {
      uv_mutex_unlock(&mutex);
      return;
    }
```

执行完上面的函数后，`slow i/o`和`fast i/o`再也没有任何区别，都由线程池以普通任务的方式接管了。





#### 整体逻辑

我们总结一下整体的运行流畅：

1. 构造一个`AsyncReq`的子类`req`

2. 将`req`的`work_req`通过`uv__work_submit`传递给线程池

3. 通过`post`将`work_req`上的工作队列`wq`传递给`loop`的整体`wq`里

4. `loop`唤醒没有在工作的`worker`顺序执行`wq`上的任务

5. `wq`上的任务每执行完一个，调用`uv__async_send`唤醒loop线程

6. `uv_async_io`检查是哪个`async_handle`被唤醒了，并检查唤醒符是否符合

7. 如果符合唤醒条件，`uv__work_done`被调用。它遍历`loop`上的`wq`，顺序执行回调队列

   

#### req怎么初始化

上面讲了第一步的构造一个`req`,那这个是如何构造的呢？通过`fs.open`，我们来看看它的实现细节。

看过源码的同学肯定知道，`open`对应的是`FSReqWrap`

```
FSReqBase* req_wrap_async = GetReqWrap(args, 3);
```

紧接着会进入到`GetReqWrap`的这段逻辑

```
if (value->IsObject()) {
    return Unwrap<FSReqBase>(value.As<v8::Object>());
  }
```

我们看看`Unwrap`这个内联函数

```
static inline T* Unwrap(v8::Local<v8::Object> handle) {
    // Cast to ObjectWrap before casting to T.  A direct cast from void
    // to T won't work right when T has more than one base class.
    void* ptr = handle->GetAlignedPointerFromInternalField(0);
    ObjectWrap* wrap = static_cast<ObjectWrap*>(ptr);
    return static_cast<T*>(wrap);
  }
```

解释一下，这里的`handle->GetAlignedPointerFromInternalField`是取到内部的值

对不熟悉c++的同学，这里简单的提一下`static_cast`的作用

实际上对任何`OOP`语言来说，子类强转为父类(向上转型)都是非常安全的。而父类要转化为子类(向下转型)都是需要编译器检查的，c++提供了两种强转:

+ static_cast:  编译阶段处理
+ dynamic_cast:  RTTI，需要虚表支持

这个函数的作用是把handle转化为`T`的类型，也就是我们需要的`FSReqWrap`

