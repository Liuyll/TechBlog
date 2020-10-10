## 概述

本章主要介绍libuv最核心的loop以及任务加入loop的过程



## 探索

### 回顾

#### loop的初始化

loop的整体初始化在上一章已经讲过，忘记的读者可以回顾上一章，这里不再浪费篇幅。



### uv_loop_t

神奇的数据结构，它储存了一个loop的所有信息。并且可以被`uv_default_loop`来初始化得到。

```c
// typedef uv_loop_s uv_loop_t
struct uv_loop_s {
  /* User data - use this for whatever. */
  void* data;
  /* Loop reference counting. */
  unsigned int active_handles;
  void* handle_queue[2];
  union {
    void* unused;
    unsigned int count;
  } active_reqs;
  /* Internal storage for future extensions. */
  void* internal_fields;
  /* Internal flag to signal loop stop. */
  unsigned int stop_flag;
  // 不同平台各自私有的实现
  UV_LOOP_PRIVATE_FIELDS
};
```

我们可以看到，整个`uv_loop_t`分为两个部分：

+ 每种事件循环都有的共有成员(平台无关)

+ 不同平台独自拥有的私有成员`UV_LOOP_PRIVATE_FIELDS`(平台相关)

其中，共有成员里最重要的就是`handle_queue[2]`



#### handle_queue

顾名思义，`handle_queue`储存的是`handle`的队列，那`handle`又是什么呢？

我们都知道，在libuv里有著名的一幅图

```
   ┌───────────────────────────┐
┌─>│           timers          │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐  
│  │     pending callbacks     │ 
│  └─────────────┬─────────────┘      
│  ┌─────────────┴─────────────┐      
│  │       idle, prepare       │ 
│  └─────────────┬─────────────┘      ┌───────────────┐
│  ┌─────────────┴─────────────┐      │   incoming:   │
│  │           poll            │<─────┤  connections, │
│  └─────────────┬─────────────┘      │   data, etc.  │
│  ┌─────────────┴─────────────┐      └───────────────┘
│  │           check           │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │
   └───────────────────────────┘

```



其中几个需要检查几个重要的队列：

+ timer
+ pending 
+ idle / prepare
+ i/o
+ check

实际上，这几个队列里都是储存了相对应的`handle`结构体。



### 单次loop时间

跟`react`里的调度一样，我们在任何循环里都会设计到一个重要的问题，一个循环的时间究竟是多少呢？

在`react`里，它hack了`requestCallbackIdle`，使每次循环的可用的更新时间保持在30帧/s内，以压榨浏览器的资源。而nodejs里并没有帧这种概念，它不存在`main thread`来争抢线程使用时间，所以它只需要保证一个循环<b>尽可能久</b>就好了。

> 什么叫尽可能久？

我们都知道，timer是我们需要检查的第一个队列。也是用户感知最明显的队列，如果保证timer不超时的情况下，我们是否可以长期的维持在一个循环里。

事实上确实如此，libuv维持了一个最小堆的来储存`uv_handle_timer`，以最近的timer超时时间作为该次循环的时间。

### i/o poll

作为循环里最重要的一轮，i/o poll是libuv里不得不提的东西。我们所有的，需要调用native与os交互的异步操作都会在这里被触发。在第一章也提到过，i/o poll是通过`epoll` 的i/o多路复用式实现的非空转cpu等待。



跟着代码，我们来分析 i/o poll做了什么事情。

首先是注册所有的在`watcher_queue`里的描述符

```
while (!QUEUE_EMPTY(&loop->watcher_queue)) {
    q = QUEUE_HEAD(&loop->watcher_queue);
    QUEUE_REMOVE(q);
    QUEUE_INIT(q);

    w = QUEUE_DATA(q, uv__io_t, watcher_queue);
    
    if ((w->events & POLLIN) == 0 && (w->pevents & POLLIN) != 0) {
      filter = EVFILT_READ;
      fflags = 0;
      op = EV_ADD;

      if (w->cb == uv__fs_event) {
        // 根据类型确定filter和po
      }

      EV_SET(events + nevents, w->fd, filter, op, fflags, 0, 0);

      if (++nevents == ARRAY_SIZE(events)) {
        if (kevent(loop->backend_fd, events, nevents, NULL, 0, NULL))
          abort();
        nevents = 0;
      }
    }
    // 根据不同的events & type 有不同的判断，不具体讲了

    w->events = w->pevents;
  }
```

熟悉epoll的同学可以看linux版本的，这里是`freeBSD`的kqueue实现，不同大同小异。

接下里就是熟悉调度的同学很熟悉的调度了...

```
void uv__io_poll(uv_loop_t* loop, int timeout) {

  // 一堆变量声明
  ...
  
  // 观察者为0,代表无异步函数需要执行,事件循环结束,直接返回
  if (loop->nfds == 0) {
    return;
  }

  nevents = 0;

  // 注册所有事件
  while (!QUEUE_EMPTY(&loop->watcher_queue)) {
    q = QUEUE_HEAD(&loop->watcher_queue);
    QUEUE_REMOVE(q);
    QUEUE_INIT(q);

    w = QUEUE_DATA(q, uv__io_t, watcher_queue);
    // 这个filter是内核过滤器，op是任务类型
    EV_SET(events + nevents, w->fd, filter, op, fflags, 0, 0);
    }

		// 根据不同的w->events类型进行不同的EV_SET
		...
    w->events = w->pevents;
  }

  for (;; nevents = 0) {
    /* Only need to set the provider_entry_time if timeout != 0. The function
     * will return early if the loop isn't configured with UV_METRICS_IDLE_TIME.
     */
    if (timeout != 0)
      uv__metrics_set_provider_entry_time(loop);

    if (timeout != -1) {
      spec.tv_sec = timeout / 1000;
      spec.tv_nsec = (timeout % 1000) * 1000000;
    }

    if (pset != NULL)
      pthread_sigmask(SIG_BLOCK, pset, NULL);

    // 等待唤醒
    nfds = kevent(loop->backend_fd,
                  events,
                  nevents,
                  events,
                  ARRAY_SIZE(events),
                  // timeout == -1则无超时时间,否则设定poll轮剩余时间为超时时间
                  timeout == -1 ? NULL : &spec);

    if (pset != NULL)
      pthread_sigmask(SIG_UNBLOCK, pset, NULL);

    /* Update loop->time unconditionally. It's tempting to skip the update when
     * timeout == 0 (i.e. non-blocking poll) but there is no guarantee that the
     * operating system didn't reschedule our process while in the syscall.
     */
    SAVE_ERRNO(uv__update_time(loop));

    // 超时返回
    if (nfds == 0) {
      if (reset_timeout != 0) {
        timeout = user_timeout;
        reset_timeout = 0;
        if (timeout == -1)
          continue;
        if (timeout > 0)
          goto update_timeout;
      }

      assert(timeout != -1);
      return;
    }

    if (nfds == -1) {
      if (errno != EINTR)
        abort();

      if (reset_timeout != 0) {
        timeout = user_timeout;
        reset_timeout = 0;
      }

      if (timeout == 0)
        return;

      if (timeout == -1)
        continue;

      /* Interrupted by a signal. Update timeout and poll again. */
      goto update_timeout;
    }

    have_signals = 0;
    nevents = 0;

    assert(loop->watchers != NULL);
    loop->watchers[loop->nwatchers] = (void*) events;
    loop->watchers[loop->nwatchers + 1] = (void*) (uintptr_t) nfds;

    // 执行
    for (i = 0; i < nfds; i++) {
      ev = events + i;
      fd = ev->ident;
      /* Skip invalidated events, see uv__platform_invalidate_fd */
      if (fd == -1)
        continue;

      // 找到对应的watcher,这个在uv__io_start时被绑定对应的uv__io_t映射
      w = loop->watchers[fd];

      // 对应的fd被停止观察了,跳过
      if (w == NULL) {
        /* File descriptor that we've stopped watching, disarm it.
         * TODO: batch up. */
        struct kevent events[1];

        EV_SET(events + 0, fd, ev->filter, EV_DELETE, 0, 0, 0);
        if (kevent(loop->backend_fd, events, 1, NULL, 0, NULL))
          if (errno != EBADF && errno != ENOENT)
            abort();

        continue;
      }

      revents = 0;

      if (revents == 0)
        continue;

      /* Run signal watchers last.  This also affects child process watchers
       * because those are implemented in terms of signal watchers.
       */
      if (w == &loop->signal_io_watcher) {
        have_signals = 1;
      } else {
        uv__metrics_update_idle_time(loop);
        w->cb(loop, w, revents);
      }

      nevents++;
    }

    if (reset_timeout != 0) {
      timeout = user_timeout;
      reset_timeout = 0;
    }

    if (have_signals != 0) {
      uv__metrics_update_idle_time(loop);
      loop->signal_io_watcher.cb(loop, &loop->signal_io_watcher, POLLIN);
    }

    loop->watchers[loop->nwatchers] = NULL;
    loop->watchers[loop->nwatchers + 1] = NULL;

    if (have_signals != 0)
      return;  /* Event loop should cycle now so don't poll again. */

    if (nevents != 0) {
      if (nfds == ARRAY_SIZE(events) && --count != 0) {
        /* Poll for more events but don't block this time. */
        timeout = 0;
        continue;
      }
      return;
    }

    // 超时返回
    if (timeout == 0)
      return;

    // 未超时继续执行调度
    if (timeout == -1)
      continue;

update_timeout:
    assert(timeout > 0);

    diff = loop->time - base;
    if (diff >= (uint64_t) timeout)
      return;

    timeout -= diff;
  }
}

```

上面是一个精简版的`uv__io_poll`函数，删除掉了对不同类型的fd执行的不同操作。

大概就是做的这几件事:

1. 监听`watcher_queue`的描述符
2. 在`timeout`允许范围内等待唤醒
3. 唤醒后执行对应任务
4. 更新剩余`timeout`
5. 任务执行完后重复`2,3,4`两个步骤
6. 超时后退出



#### 定时

任何调度算法最核心的都是每一轮的计时时间

poll的超时时间是通过`uv_backend_timeout`函数计算得出的

```
// return 0 终止，-1 无限
int uv_backend_timeout(const uv_loop_t* loop) {
	// 循环被上一轮循环中某个任务终止
  if (loop->stop_flag != 0)
    return 0;
	
	// 检查循环是否在激活状态
  if (!uv__has_active_handles(loop) && !uv__has_active_reqs(loop))
    return 0;
	
	// 没有能响应的async_handle
  if (!QUEUE_EMPTY(&loop->idle_handles))
    return 0;
	
	// 有上轮循环 io poll阶段还未执行的任务
  if (!QUEUE_EMPTY(&loop->pending_queue))
    return 0;
  
  // handle关闭
  if (loop->closing_handles)
    return 0;

  return uv__next_timeout(loop);
}

```

这个计算非常简单，主要是判断一些不该进行循环的情况。

但必须提一下这个条件判断

```
if (!QUEUE_EMPTY(&loop->pending_queue))
    return 0;
```

这个代表着我们上一轮`io poll`阶段有任务还没执行到，这个不能再拖延到下一轮调度了，需要立马执行，所以跳过本轮的`io poll`阶段。

我们继续看下正常的时间计算

```
int uv__next_timeout(const uv_loop_t* loop) {
  const struct heap_node* heap_node;
  const uv_timer_t* handle;
  uint64_t diff;
	
	// timer是个最小堆
  heap_node = heap_min(timer_heap(loop));
  // 没有在运行的timer handle，此时不需要为循环计时
  if (heap_node == NULL)
    return -1; /* block indefinitely */

  handle = container_of(heap_node, uv_timer_t, heap_node);
  // 最近的定时器已经超时了，此时立马进入下一轮循环
  if (handle->timeout <= loop->time)
    return 0;
  
  // 返回剩余时间，INT_MAX作为最大时间打底
  diff = handle->timeout - loop->time;
  if (diff > INT_MAX)
    diff = INT_MAX;

  return (int) diff;
}
```

整体逻辑非常清晰，相信读者都有能力看懂。



### 令人震惊的process.nextTick

事实上，任何microTask都不是由libuv来控制的，而是交给v8接管。但这里必须要独立的讲nextTick，而且是令人震惊的nextTick

#### 触发？

我们先来观察一下它的触发时机

```
void InternalCallbackScope::Close() {
  if (!tick_info->has_scheduled()) {
    env_->isolate()->RunMicrotasks();
  }
	
	// tick_callback_function 是在前一个函数被设置进来的js回调，也就是nextTick的回调
  if (env_->tick_callback_function()->Call(process, 0, nullptr).IsEmpty()) {
    env_->tick_info()->set_has_thrown(true);
    failed_ = true;
  }
}
```

我们可以看到，`nextTick`回调的触发是在`InternalCallbackScope`这样一个类被`Close`的时候触发

这个`Close`的行为可以被追溯到`InternalMakeCallback`被触发的时候

```
// src/node.cc
MaybeLocal<Value> InternalMakeCallback(Environment* env,
                                       Local<Object> recv,
                                       const Local<Function> callback,
                                       int argc,
                                       Local<Value> argv[],
                                       async_context asyncContext) {
  InternalCallbackScope scope(env, recv, asyncContext);
  scope.Close();//Close会调用_tickCall
}
```

在第一章node和libuv的结合中，我们提到过`InternalMakeCallback`实际上就是往`asyncWrap`的子类上设置回调的方法。所以每一个handle结束后，都会触发到这个方法。执行其对应的回调，并且触发我们的`_tickCallback`。



#### 小试牛刀？

```
setTimeout(function () {
    console.log(1);
    process.nextTick(function () {
        console.log(2);
        process.nextTick(function () {
            console.log(2);
            process.nextTick(function () {
                console.log(2);
            });
        });
    });
});
process.nextTick(function () {
    console.log(3);
    setTimeout(function () {
        console.log(4);
    })
});

```

它的最终输出是`3,1,2,2,2,4`，如果你明白`nextTick`触发的时机，并且清楚`timeout`由`timeWrap`封装，你就能明白为什么是这样输出的。



#### microTask？

```
setTimeout(function () {
    console.log(1);
    process.nextTick(function () {
        console.log(2);
        Promise.resolve(5).then((v) => console.log(v))
        process.nextTick(function () {
            console.log(2);
            process.nextTick(function () {
                console.log(2);
            });
        });
    });
});
process.nextTick(function () {
    console.log(3);
    setTimeout(function () {
        console.log(4);
    })
});

```

考虑上面的情况，我们添加了一个`Promise`，它的输出是什么呢？

别慌，我们先不要去试。刚才我们提到过`InternalCallbackScope`的关闭会触发`_tickCallback`，那么这个又是什么呢？

```
function _tickCallback() {
    let tock;
    do {
      while (tock = shift()) {
        // ... emit async_hooks
        if (tock.args === undefined)
          callback();//执行调用process.nextTick()时放置进来的callback
        else
          Reflect.apply(callback, undefined, tock.args);//执行调用process.nextTick()时放置进来的callback
 				// 链式执行
        emitAfter(asyncId);
      }
      
      runMicrotasks();//microtasks将会在此时执行
    } while (head.top !== head.bottom || emitPromiseRejectionWarnings());
    tickInfo[kHasPromiseRejections] = 0;
}
```

相信大家能明白了，`nextTick`阻塞了`microTask`的执行。所以上面的输出顺序大家可以写出来了。