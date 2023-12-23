---
title: 关于 asyncio 的喃喃自语
date: 2018-01-30 00:22:04
tags:
    - Python
    - 异步编程
categories: 工具
---

大多数东西都是根据之前学习中的印象而来，这篇文章更多的是喃喃自语吧。

### 协程

在 Python 3.5 之前，协程在我眼中应该就是 `yield` 这个语法的同义，通过 `send`、`throw` 等方法来作为其交互，用于多进程中以提升效率，然而就 Python 的使用环境来说，其实接触到的机会并不是太多。

在 Python 3.5 之后，给了一种新的，现在看起来还是比较友好的选择

```python
async def hello():
    print("Hello world!")
    # asyncio.sleep 也是一个 coroutine
    # 异步调用asyncio.sleep(1):
    # 线程也可以从这儿拿到返回值（asyncio.sleep(1) 返回值为 None）
    r = await asyncio.sleep(1)
    print("Hello again!")
```

在 `I/O` 密集的地方，填入 `await` ，在 `def` 前面加上 `async`，不需要去适应至今让我还是十分不适的 `yield` / `yield from` 

<!-- more -->

### EventLoop

最初接触到这个概念应该是 `epoll` 中，然后是在 `Tornado` 的源码中，当然，`Tornado` 也是用了 `epoll`。

在 `EventLoop` 的世界中，它总是与 `thread` 共存，它只是负责接收事件，余下的由 `thread` 来解决，保证了并发，在最近阅读的 `Kafka` 源码中它看起来也是这么干的，不过它可能比起 `Tornado` 做得更加彻底，`Tornado` 整个应用只会保持一个 `EventLoop`，而 `Kafka` 为了保证高吞吐高并发，除了 `Accept` 之外，拥有多个类型注册器，又对应了不同类型的线程池，基本的读和写都不成问题了。

说道这里，才开始说了 `asyncio` 中的 `EventLoop`。如果你只是运行 demo 的话，你会发现获取 `event_loop` 的方法非常简单 `asyncio.get_event_loop()`，但这只是在主线程中，如果你在其他线程中再看

```python
import asyncio
import time, threading


# 新线程执行的代码:
def loop():
    print('thread %s is running...' % threading.current_thread().name)
    loop = asyncio.get_event_loop()
    time.sleep(1)
    print('thread %s ended.' % threading.current_thread().name)

if __name__ == '__main__':
    print('thread %s is running...' % threading.current_thread().name)
    t = threading.Thread(target=loop, name='LoopThread')
    t.start()
    t.join()
    print('thread %s ended.' % threading.current_thread().name)
```

获得的却是一个 `RuntimeError`

```
RuntimeError: There is no current event loop in thread 'LoopThread'.
```

需要的是这么做

1. `loop = asyncio.new_evet_loop()`
2. `asyncio.set_event_loop(loop)`

```python
# 新线程执行的代码:
def loop():
    print('thread %s is running...' % threading.current_thread().name)
    new_loop = asyncio.new_event_loop()
    asyncio.set_event_loop(new_loop)
    loop = asyncio.get_event_loop()
    time.sleep(1)
    print('thread %s ended.' % threading.current_thread().name)
```

其实是有种，多此一举的感觉。

那，`event_loop` 有什么用呢？

-  它是 `asyncio` 的起点，是执行所有事件的起点
-  通过 `loop.run_forever()` + `loop.call_*` 实现对事件的调度，最后关闭的时候调用 `loop.close()`
-  通过 `loop.add_reader()、loop.remove_reader()、loop.add_writer()、loop.remove_writer()` 来注册读写事件，其中有两个参数：文件描述符、`callback`，另外还有就是 `callback` 的参数，事件驱动嘛，多用于 socket。
-  通过使用 `run_in_executor` 来达到将耗时调用委托给线程池/进程池


```python
import asyncio
import time
from concurrent.futures import ProcessPoolExecutor


def cpu_bound_operation(x):
    print("cpu bounding")
    time.sleep(x)

async def func():
    # 线程池 / 进程池执行
    loop.run_in_executor(p, cpu_bound_operation, 5)
    print("doing something thing")


if __name__ == '__main__':
    loop = asyncio.get_event_loop()
    p = ProcessPoolExecutor(2)
    loop.run_until_complete(func())
    loop.run_until_complete(func())
    loop.run_until_complete(func())
    
"""
doing something thing
doing something thing
doing something thing
cpu bounding
cpu bounding

-- sleep 5 seconds --

cpu bounding
"""
```



### Future 与 Future

  真的，我第一次知道 `Future` 是在 Java 中，然后是在 `Tornado` 中，然后告诉我，`Python` 中还有两个（完全不兼容） `Future`，分别是 `asyncio.futures.Future`，`concurrent.futures.Future`，而且这两个都在 `asyncio` 中使用，真是感觉，这屎有毒。

>  例如，`asyncio.run_coroutine_threadsafe()` 将调度一个协程到在另一个线程中运行的事件循环，但它返回一个 `concurrent.futures.Future` 对象，而不是 `asyncio.futures.Future` 对象。 这是有道理的，因为只有 `concurrent.futures.Future` 对象是线程安全的。

你如果在阅读文档的时候你会发现，`loop.run_until_complete` 里面给出的参数就是 `future`，这里的 `future` 指的是 `asyncio.futures.Future`，文档会告诉你 `If the argument is a coroutine object, it is wrapped by ensure_future().` （我真的完全不想知道 `coroutine object` diff `coroutine`），简单来说我们从 `ensure_future()` 入手知道它大概封装了一个协程，然后读源码的时候发现它给你返回的是一个 `task`。

```Python
def ensure_future(coro_or_future, *, loop=None):
    """Wrap a coroutine or an awaitable in a future.

    If the argument is a Future, it is returned directly.
    """
    if futures.isfuture(coro_or_future):
        if loop is not None and loop is not coro_or_future._loop:
            raise ValueError('loop argument must agree with Future')
        return coro_or_future
    elif coroutines.iscoroutine(coro_or_future):
        if loop is None:
            loop = events.get_event_loop()
        task = loop.create_task(coro_or_future)
        if task._source_traceback:
            del task._source_traceback[-1]
        return task
    elif compat.PY35 and inspect.isawaitable(coro_or_future):
        return ensure_future(_wrap_awaitable(coro_or_future), loop=loop)
    else:
        raise TypeError('A Future, a coroutine or an awaitable is required')
```

这时候你大概能知道 `run_until_complete` 执行的是一个 `task` （当然，你如果去看文档的会知道，它能执行一个 task 列表）。

### 感想

讲真，Python 3 的 `asyncio` 在我对比了 `epoll` 和 `Java` 的异步机制之后我觉得是将一件本来就不是对初学者很友好的机制再次复杂化，你需要很多新的东西，比如：`call`，协程，`eventLoop`，`task`，`future`，`executor` 等等，以及延展的 `yield`，`generator` 等等。

幸好，社区已经有很多人为我们封装了基础的框架，优秀如 `aiohttp` 等，希望以后这类理解成本越来越低吧。

### 参考

[官方文档](https://docs.python.org/3/library/asyncio.html)

[雾里看花之 Python Asyncio](https://linux.cn/article-8051-1.html)