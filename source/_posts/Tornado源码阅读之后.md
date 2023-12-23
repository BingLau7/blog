---
title: Tornado源码阅读之后
date: 2015-11-09 16:20:04
tags:
    - Python
    - Tornado
categories: 源码分析
---

大概是上个星期将Tornado的源码大概读了一遍，有这么几篇文章值得参考:

- [为什么要阅读Tornado的源码？](http://www.nowamagic.net/academy/detail/13321002)
- [[翻译]深入理解Tornado——一个异步web服务器](http://www.cnblogs.com/yiwenshengmei/archive/2011/06/08/understanding_tornado.html)

<!--more-->

主要仔细读的就`tcpsever.py`, `tcpclient.py`, `httpclient.py`, `httpserver.py`, `web.py`, `ioloop.py`。

所谓Tornado的高并发其实是`epoll`/`kqueue`的封装，对读写进行事件的封装。在全局仅有一个的IOLoop的实例中进行死循环来等待请求发送，这其实主要是避免了IO(网络连接)的等待，而我们在等到请求之后会执行`add_handler`方法，这其中其实就是我们在`Application`中所传入的类，这里面类使用了`__call__`进行包装，从而可以达到调用其类本身就可以调用其中的`RequestHandler`的对应方法`_execute()`， 这里面就是处理了我们的请求，并根据请求的方法分配到我们实现的子类自己所实现的`get`, `post`方法等，其中检查`xsrf`等也是在这里面开始进行。

其实这次阅读还是有一些没有深入的地方，例如`iostream.py`中对于数据流的处理，例如各种util也没有详细阅读。

下次计划准备阅读`Flask`及其相应的代码，毕竟在Python框架里面，我对这两个框架是最为熟悉的了，也可以与现在阅读的Tornado做一个横向对比。
