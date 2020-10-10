## 概述

上一篇已经讲了nodejs怎么与libuv进行通信的，也提到了`epoll`在这个通信过程中扮演的角色与作用。这章主要是讲其中的细节



## 探索

### epoll高效的原因

#### epoll是否是轮询的？

答案是肯定的，epoll依然需要通过调用`schedule_timeout`来进行用户态的轮询，不过它轮询的队列从`select`的`O(all fd)`变化到了`O(已完成的fd)`，但是完成队列的成员添加则不是用户态进行的，而是注册进内核的回调函数执行的结果。

#### rbtree

+ 创建:

  我们在调用`epoll_create`的时候，会在`epoll`专用的cache区域里创建一个`rbtree`以储存后续注册的fd

+ 添加:

  在调用`epoll_ctl`注册监听事件时，会首先在`rbtree`上判断是否存在相同的fd，并决定是否加入树

#### 节省内核拷贝

注意，上面提到`rbtree`的创建并不是在用户态的。而是在内核空间的数据结构，我们通过`epoll_ctl`往其中加入句柄，是不再需要内核进行复制的。

而通过`select`方式，内核需要拷贝所有需要监听的fd到内核里面，这回消耗巨大的时间。

#### complete queue

epoll把触发监听的fd放入到一个`complete queue`里，由用户轮询这个队列，判断是否需要进行处理。



#### 边缘触发(ET)

epoll的边缘触发在缓冲区满，缓冲区非满，缓存区空，缓冲区非空几个阶段触发。

对应的水平触发(LT)就会在任何有数据进入的时候被触发。

那实际上，ET比LT的效率要高很多，因为很多时候是不需要响应的。



### 回顾

上一章讲过了，epoll是依靠监听一些事件来触发。`libuv`里用`write`一个字节的方式填充进缓冲区触发epoll，那我们看下这个操作的全过程。



### 初始化loop

初始化函数非常大，我们一步一步在注释里分析

```
int uv_loop_init(uv_loop_t* loop) {
  uv__loop_internal_fields_t* lfields;
  void* saved_data;
  int err;


  saved_data = loop->data;
  memset(loop, 0, sizeof(*loop));
  loop->data = saved_data;

  lfields = (uv__loop_internal_fields_t*) uv__calloc(1, sizeof(*lfields));
  if (lfields == NULL)
    return UV_ENOMEM;
  loop->internal_fields = lfields;

  err = uv_mutex_init(&lfields->loop_metrics.lock);
  if (err)
    goto fail_metrics_mutex_init;

  heap_init((struct heap*) &loop->timer_heap);
  // worker工作结束后，会把对应的req->work_req加入到其中，libuv直接遍历这里执行回调
  QUEUE_INIT(&loop->wq);
  // idle
  QUEUE_INIT(&loop->idle_handles);
  // 作用在i/o poll阶段，储存了异步观察者
  QUEUE_INIT(&loop->async_handles);
  // check
  QUEUE_INIT(&loop->check_handles);
  // preapare
  QUEUE_INIT(&loop->prepare_handles);
  QUEUE_INIT(&loop->handle_queue);

  loop->active_handles = 0;
  loop->active_reqs.count = 0;
  loop->nfds = 0;
  loop->watchers = NULL;
  loop->nwatchers = 0;
  QUEUE_INIT(&loop->pending_queue);
  QUEUE_INIT(&loop->watcher_queue);

  loop->closing_handles = NULL;
  uv__update_time(loop);
  loop->async_io_watcher.fd = -1;
  loop->async_wfd = -1;
  loop->signal_pipefd[0] = -1;
  loop->signal_pipefd[1] = -1;
  loop->backend_fd = -1;
  loop->emfile_fd = -1;

  loop->timer_counter = 0;
  loop->stop_flag = 0;

  err = uv__platform_loop_init(loop);
  if (err)
    goto fail_platform_init;

  uv__signal_global_once_init();
  err = uv_signal_init(loop, &loop->child_watcher);
  if (err)
    goto fail_signal_init;

  uv__handle_unref(&loop->child_watcher);
  loop->child_watcher.flags |= UV_HANDLE_INTERNAL;
  QUEUE_INIT(&loop->process_handles);

  err = uv_rwlock_init(&loop->cloexec_lock);
  if (err)
    goto fail_rwlock_init;

  err = uv_mutex_init(&loop->wq_mutex);
  if (err)
    goto fail_mutex_init;

  err = uv_async_init(loop, &loop->wq_async, uv__work_done);
  if (err)
    goto fail_async_init;

  uv__handle_unref(&loop->wq_async);
  loop->wq_async.flags |= UV_HANDLE_INTERNAL;

  return 0;
}
```

有几个细节需要我们了解的。

#### 关于一些queue

+ pending_queue 这个是等待`pending`阶段来执行的任务队列，储存的是不走`epoll`的任务。如tcp里遭遇了`ECONNREFUSED`时，就会把错误回调拉入到`pending_queue`里。正常情况下，一般是走`io poll`逻辑的
+ watcher_queue 这个是等待`io poll`阶段来监听的事件队列

### 创建pipe端口

上面提到了，epoll依靠监听读事件来唤醒主线程。在此之前，必须要了解一下`unix`下的pipe是怎么运行的(实际上，libuv在linux下使用的是eventfd句柄)

一个简单的pipe demo，通过pipe来实现父子进程的通信

```c
int fd[2]
write_fd = &fd[1]
read_fd = &fd[0]

if(pipe(fd) != -1)...
pid = fork()
if(pid == 0) {
  close(read_fd)
  write(write_fd,...)
}
close(write_fd)
char buf[10]
read(read_fd,buf,...)
```

有了这个基础，我们知道pipe可以在内核空间生成读写两个端口，通过这两个端口进行通信。我们不直接看这两个端口的创建时机，先看看worker线程怎么唤醒主线程

上面一章已经提到过了，worker线程通过`uv_async_send`里的`uv__async_send`来唤醒

```
static void uv__async_send(uv_loop_t* loop) {
  const void* buf;
  ssize_t len;
  int fd;
  int r;

  buf = "";
  len = 1;
  
  // 写端fd来自loop里的async_wfd
  fd = loop->async_wfd;

  do
    r = write(fd, buf, len);
  while (r == -1 && errno == EINTR);

  if (r == len)
    return;

  if (r == -1)
    if (errno == EAGAIN || errno == EWOULDBLOCK)
      return;

  abort();
}
```

这个函数很清晰，就是向`write_fd`写入了一个字节的数据。`write_fd`被绑定在`loop->async_wfd`里

接下来，我们就到了最核心的部分。这个读写句柄是什么时候创建的呢？事实上，在我们的事件循环初始化的时候，它就被创建了，并且绑定在我们的`loop`这个结构体上了。

1. 首先是`uv_async_t`这个异步句柄的初始化。

   注意，这个异步句柄默认是由libuv在创建`default loop`时构造的，回调函数是`uv_async_io`。实际上，你也可以创建自己的异步句柄。

```
int uv_async_init(uv_loop_t* loop, uv_async_t* handle, uv_async_cb async_cb) {
  int err;

  err = uv_async_start(loop);
  if (err)
    return err;

  uv__handle_init(loop, (uv_handle_t*)handle, UV_ASYNC);
  
  // 绑定回调
  handle->async_cb = async_cb;
  handle->pending = 0;

  QUEUE_INSERT_TAIL(&loop->async_handles, &handle->queue);
  uv__handle_start(handle);

  return 0;
}
```

2. 接着看下`uv_async_start`

   这个函数是绑定读写端口到`loop`上

   ```
   static int uv__async_start(uv_loop_t* loop) {
     int pipefd[2];
     int err;
   
   	// unix下
     err = uv__make_pipe(pipefd, UV__F_NONBLOCK);
   
     // 初始化async_io_watcher
     uv__io_init(&loop->async_io_watcher, uv__async_io, pipefd[0]);
     uv__io_start(loop, &loop->async_io_watcher, POLLIN);
   
     // 储存写端口
     loop->async_wfd = pipefd[1];
   
     return 0;
   }
   ```

   `uv__io_init`是简单的初始化函数，它把读端口绑定在`uv_io_t (loop -> async_io_watcher)`上面。

   接下来最重要的是`uv__io_start`，这个函数把`io_watcher`上的观察者队列放在了`loop`的`watcher_queue`上面。并且映射读端口到`loop`的观察者端口列表上。后续在`uv_async_io`上会检查触发者是哪个读端口，进而找到对应的`uv__io_t`

   ```
   void uv__io_start(uv_loop_t* loop, uv__io_t* w, unsigned int events) {
     w->pevents |= events;
     maybe_resize(loop, w->fd + 1);
   
     if (QUEUE_EMPTY(&w->watcher_queue))
       QUEUE_INSERT_TAIL(&loop->watcher_queue, &w->watcher_queue);
   
     if (loop->watchers[w->fd] == NULL) {
       loop->watchers[w->fd] = w;
       loop->nfds++;
     }
   }
   ```

   我们为什么需要一个`uv_async_io`来检查触发者呢？

   是因为我们可能有多个事件循环，就对应了多个`uv_async_handle`。此时一个事件循环被触发，不应该导致其他的事件循环同时被唤醒。

   

   