## 概述

nodejs是有名的异步语言。那么nodejs的核心libuv究竟是如何运转的呢？又是如何和nodejs联系在一起的呢？



## 探索

实际上，在`uv__io_poll`这个阶段，如果I/O执行队列为空，那么libuv会在此阻塞到下一个loop时间，并且在这段时间里，可以被`epoll`唤醒，这是libuv保证cpu不空转的核心秘密。

关于epoll的具体细节，我们放在下一章来详细讲解。

本章主要分为几个小部分讲解：

+ nodejs与libuv的协调

+ libuv与epoll的协调
+ 实战: nodejs的回调触发过程

  ### nodejs如何与libuv的epoll联系起来

#### 观察者

> 注意，这里的观察者跟libuv里的uv_io观察者不一致，这是cc层的观察者，用来连接libuv和javascript

这是一个非常复杂的问题，我们的代码是javascript，通过`binding`执行cc层的胶水代码，变成了一个个原生调用。跨层之间必定通过一种数据结构来储存两层的传入与返回值。

这个结构体就是`AsyncWrap`。我们把它叫做观察者，当然，没有任何观察者直接使用它，而是使用它的子类，拿`FsReqWrap`为例

```
FSReqWrap -> ReqWrap -> AsyncWrap
```

接下来，我们把`callback，context，入参`等信息储存在这个观察者上



#### 通信

我们有了观察者后，就能利用这个中间者来进行通信了。

比如在fs里，我们会调用`native`的fs来进行真正的函数调用。

```
binding.open(pathModule._makeLong(path),
               stringToFlags(options.flag || 'r'),
               0o666,
               // 我们的观察者
               req);
```

需要注意的是，我们在调用`fs.read`时，中间还经过了一层异步的`fs.open`，并在`fs.open`的回调里调用了后续的`read`操作。

```
// deps/uv/src/unix/fs.c

int uv_fs_read(uv_loop_t* loop, uv_fs_t* req,
               uv_file file,
               const uv_buf_t bufs[],
               unsigned int nbufs,
               int64_t off,
               uv_fs_cb cb) {
  if (bufs == NULL || nbufs == 0)
    return -EINVAL;

  INIT(READ);
  req->file = file;

  req->nbufs = nbufs;
  req->bufs = req->bufsml;
  if (nbufs > ARRAY_SIZE(req->bufsml))
    req->bufs = uv__malloc(nbufs * sizeof(*bufs));

  if (req->bufs == NULL) {
    if (cb != NULL)
      uv__req_unregister(loop, req);
    return -ENOMEM;
  }
	
	// 复制结果
  memcpy(req->bufs, bufs, nbufs * sizeof(*bufs));

  req->off = off;
  do {                                                                        \
    if (cb != NULL) {                                                         \
      uv__work_submit(loop, &req->work_req, uv__fs_work, uv__fs_done);        \
      return 0;                                                               \
    }                                                                         \
    else {                                                                    \
      uv__fs_work(&req->work_req);                                            \
      return req->result;                                                     \
    }                                                                         \
  }                                                                           \
  while (0)
}
```

我们可以看到最重要的异步操作`uv__work_submit`将观察者传进libuv的`ThreadPool`。

至此，nodejs和libuv就完全关联上了。



#### 小试牛刀

写过nodejs addon的同学应该都知道，libuv暴露出了一个加入worker queue的函数`uv_queue_work`，这个函数可以把我们的同步任务加入变为异步回调的形式，这是如何实现的呢？

首先看下`uv_queue_work`的简单用法

```
int count_fibs() {
	const fib_count = 10000;
	uv_work_t reqs[fib_count];
	
	for(int i = 0;i < fib_count;i++) {
		uv_work_t cur_work = reqs[i];
		cur_work.data = i
		uv_queue_work(uv_get_current_loop(), &cur_work,fib,after_fib_cb)
	}
	// 不一定一轮跑的完，不要用Once
	uv_run(loop,UV_RUN_DEFAULT)
}

void fib(uv_work_t req) {
  //	count fib
}

void after_fib_cb(uv_work_t req) {
	int ret = *(int *)req.data;
	makeCallback(ret)
}
```

先看看`uv_queue_work`的定义

```
int uv_queue_work(uv_loop_t* loop,
                  uv_work_t* req,
                  uv_work_cb work_cb,
                  uv_after_work_cb after_work_cb) {
  assert(work_cb != null);

  uv__req_init(loop, req, UV_WORK);
  req->loop = loop;
  req->work_cb = work_cb;
  req->after_work_cb = after_work_cb;
  uv__work_submit(loop,
                  &req->work_req,
                  UV__WORK_CPU,
                  uv__queue_work,
                  uv__queue_done);
  return 0;
}
```

很简单，它就是把我们要做的事情用req封装后，再通过`uv__work_submit`传递给我们的线程池

从上面我们能看到一个关键的线索`uv_work_t`，那么这个结构体是什么？

```
struct uv_work_s {
  UV_REQ_FIELDS
  uv_loop_t* loop;
  uv_work_cb work_cb;
  uv_after_work_cb after_work_cb;
  UV_WORK_PRIVATE_FIELDS
};
```

> c语言里面继承很有意思，都是通过结构体包含另外一个宏定义来实现的

`UV_WORK_PRIVATE_FIELDS`实际上是`struct uv__work`的宏定义

```
struct uv__work {
  void (*work)(struct uv__work *w);
  void (*done)(struct uv__work *w, int status);
  struct uv_loop_s* loop;
  // wq == work_queue
  void* wq[2];
};
```



这个结构体封装了一切我们需要的信息，并且它将会一直被传递，最终被下一个章节所讲的回调触发函数触发。



#### 如何调用回调

上面已经讲明白了我们的回调挂载在`AsyncWrap`的`complete`属性上，但还是不清楚回调在什么时机被执行。

为了通俗的讲明白，我们先看看`libuv`官方提供的demo。

```
int main() {
    uv_loop_t *loop = malloc(sizeof(uv_loop_t));
    uv_loop_init(loop);

    printf("Now quitting.\n");
    uv_run(loop, UV_RUN_DEFAULT);

    uv_loop_close(loop);
    free(loop);
    return 0;
}
```

其中，`uv_loop_init`就是我们实现回调的关键，它内部调用了`uv_async_init`来注册事件循环和回调函数。

##### 在nodejs中，libuv的初始化

当然，libuv的主线程初始化还是没有上面demo这么简单。

node会在启动的时候调用`uv_default_loop`来初始化

```
// node/src/node.cc

NodeMainInstance main_instance(&params,
                                   uv_default_loop(),
                                   per_process::v8_platform.Platform(),
                                   result.args,
                                   result.exec_args,
                                   indexes);
```

不过这个函数内部确实就做了调用`uv_loop_init`这样一件事。这里只是把`nodejs`的入口放出来。



##### 多线程运行

事实上，在每一个worker执行完自己的任务的时候，会调用`uv_async_send`来通知主线程。

而主线程在最开始的使用，就会像上面demo一样调用`uv_loop_init`，那么在收到`uv_async_send`的信号的时候，它会调用默认的`uv_work_done`来执行回调。

`uv_loop_init`实际上是初始化了一个`uv_async_t`异步句柄，使它有能力被其他线程唤醒。它调用`uv__async_start`构造一个管道，并赋予`uv__async_io`的唤醒回调函数

```
static void uv__async_io(uv_loop_t* loop, uv__io_t* w, unsigned int events) {
  char buf[1024];
  ssize_t r;
  QUEUE queue;
  QUEUE* q;
  uv_async_t* h;

  assert(w == &loop->async_io_watcher);

  // 这个 for 循环用来确认是否有 uv_async_send 调用 
  // 方法是判断写入是否为1
  for (;;) {
    ...
  }
 
  // 双缓冲
  QUEUE_MOVE(&loop->async_handles, &queue);
  while (!QUEUE_EMPTY(&queue)) {
    q = QUEUE_HEAD(&queue);
    h = QUEUE_DATA(q, uv_async_t, queue);

    QUEUE_REMOVE(q);
    // 重新把异步句柄插到循环，下次循环时触发
    QUEUE_INSERT_TAIL(&loop->async_handles, q);

    // 确认这个 async_handle 是否真的被触发
    if (cmpxchgi(&h->pending, 1, 0) == 0)
      continue;

    if (h->async_cb == NULL)
      continue;
    
    // uv_work_done
    h->async_cb(h);
  }
}
```

上面的注释已经指出了,挂载在`async_cb`上的回调函数是`uv__work_done`，下面看下这个函数做了什么

```
// libuv/src/threadpool.c

void uv__work_done(uv_async_t* handle) {
  struct uv__work* w;
  uv_loop_t* loop;
  QUEUE* q;
  QUEUE wq;
  int err;

  loop = container_of(handle, uv_loop_t, wq_async);
  // 上互斥锁取队列
  uv_mutex_lock(&loop->wq_mutex);
  QUEUE_MOVE(&loop->wq, &wq);
  uv_mutex_unlock(&loop->wq_mutex);

  while (!QUEUE_EMPTY(&wq)) {
    q = QUEUE_HEAD(&wq);
    QUEUE_REMOVE(q);

    w = container_of(q, struct uv__work, wq);
    err = (w->work == uv__cancelled) ? UV_ECANCELED : 0;
    // 执行回调
    w->done(w, err);
  }
}
```

请注意，`uv__work_done`函数不断的取`loop -> wq`队列上的`req`来执行回调

这个`w->done`就是我们执行回调的阶段，不过这不是js的回调，这是`uv__work_submit`时注册的回调。我们看下`fs.read`时的`w->done`

```
static void uv__fs_done(struct uv__work* w, int status) {
  uv_fs_t* req;

	// 得到该观察者
  req = container_of(w, uv_fs_t, work_req);
  // 取消在循环里的注册
  uv__req_unregister(req->loop, req);

  if (status == UV_ECANCELED) {
    assert(req->result == 0);
    req->result = UV_ECANCELED;
  }
  
  // 准备执行js回调
  req->cb(req);
}
```

当然，这里的`req->cb`是观察者上的回调，这个回调是胶水层注册上去的。相当于js回调的一层封装。



#### epoll如何和libuv联系起来

我们上面已经提到了，`uv_async_send`可以唤醒我们的主线程，但依然不知道唤醒的是哪个阶段，以及是怎么唤醒的。

我们翻看事件循环的外层`while`函数`uv_run`(这个可以换一篇来讲，它是libuv的总循环)

值得注意的是，`uv_run`有很多阶段，但我们目前只需要关心最重要的`uv__io_poll`即可

```
 if (no_epoll_wait != 0 || (sigmask != 0 && no_epoll_pwait == 0)) {
 			// io多路复用
      nfds = epoll_pwait(loop->backend_fd,
                         events,
                         ARRAY_SIZE(events),
                         timeout,
                         &sigset);
      if (nfds == -1 && errno == ENOSYS) {
        uv__store_relaxed(&no_epoll_pwait_cached, 1);
        no_epoll_pwait = 1;
      }
    } else {
    	// io多路复用
      nfds = epoll_wait(loop->backend_fd,
                        events,
                        ARRAY_SIZE(events),
                        timeout);
      if (nfds == -1 && errno == ENOSYS) {
        uv__store_relaxed(&no_epoll_wait_cached, 1);
        no_epoll_wait = 1;
      }
    }
```

熟悉`epoll`的同学应该就知道上面的`epoll_wait`就是我们的核心。

当然不熟悉`epoll`也没关系，我们只需要知道epoll可以监听很多状态，一旦这个状态达到触发要求，epoll就会得到响应。

那么我们的`uv_async_send`是不是也是触发了这几个状态之一呢？

```
r = write(fd, buf, len);
```

如你所料，`uv_async_send`函数向缓冲区写入了一字节，进入上面讲述的`epoll`流程，最后我们的`epoll_wait`收到响应，又进入上文提到的`w->done`阶段，进入回调环节。

至此，epoll就完全和libuv联系起来了。

> 唤醒具体是在`uv_run`的`uv_io_poll`的`kevent(unix)`或`epoll.wait(linux)`这个epoll等待阶段。



## 参考

[**libuv线程池和主线程通信原理**](https://cnodejs.org/topic/5e36feb4267721420912ad1c)

[Node - 异步IO和事件循环](https://juejin.im/post/6844904093459152903)

