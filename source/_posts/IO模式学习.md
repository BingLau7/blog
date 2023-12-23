---
title: IO模式学习
date: 2016-01-21 15:07:41
tags:
    - IO
    - 异步编程
categories: 基础原理
---

### 问题

1. 同步与异步，阻塞与非阻塞有什么区别，举例说明
2. 什么是多路复用

### 问题引入

这是我们通常所写的程序

```
def normal():
    for i in range(10):
        print(i)

if __name__ == '__main__':
    print('begin')
    normal()
    print('end')

#result
begin
0
1
2
3
4
5
6
7
8
9
end

```

通常我们的输入需要等到上条输入结束之后才能进行，这也许是我们通常最想要得到的结果，但是有某些时候如果我们所需要的结果没有前后文影响的情况下，我们可能更希望它不是这么死板地来执行的，而是异步执行，最典型的一个例子就是访问网站，网站在进行渲染的时候用到了大量的IO操作，而其他用户不可能等到一个用户渲染完之后在进行渲染，否则会造成比较差的用户体验。

事实上，所有的IO操作（如数据库查询，读写文件等）都回造成阻塞，它们都会让我们无法利用到IO执行这一期间的计算机资源。

为了解决这个问题，计算机引入了一些**IO模式**的区别。

<!--more-->

### Linux下常见IO模式介绍

首先是同步IO与异步IO介绍：

**同步I/O操作**：实际的I/O操作将导致请求进程阻塞，直到I/O操作完成。

**异步I/O操作(AIO)**：实际的I/O操作不导致请求进程阻塞。

接下来是Linux中常见的IO模式介绍：
1. **阻塞式I/O：**应用进程调用I/O操作时阻塞，只有等待要操作的数据准备好，并复制到应用进程的缓冲区中才返回。
2. **非阻塞式I/O：**当应用进程要调用的I/O操作会导致该进程进入阻塞状态时，该I/O调用返回一个错误，一般情况下，应用进程需要利用轮询的方式来检测某个操作是否就绪。数据就绪后，实际的I/O操作会等待数据复制到应用进程的缓冲区中以后才返回。
3. **I/O多路复用：**阻塞发生在select/poll的系统调用上，而不是阻塞在实际的I/O系统调用上。select/poll发现有数据就绪后，通过实际的I/O操作将数据复制到应用进程的缓冲区中。
4. **异步I/O：**应用进程通知内核开始一个异步I/O操作，并让内核在整个操作（包含将数据从内核复制到应该进程的缓冲区）完成后通知应用进程。

如图所示：
![IO模式](/uploads/blog_img/IO_modal.jpg)

从上面图中可以看出，该图把I/O操作分为两个阶段，第一阶段等待数据可用，第二阶段将数据从内核复制到用户空间。前三种模型的区别在于第一阶段（阻塞式I/O阻塞在I/O操作上，非阻塞式I/O轮询，I/O复用阻塞在select/poll或者epoll上），第二阶段都是一样的，即这里的阻塞不阻塞体现在第一阶段，从这方面来说I/O复用类型也可以归类到阻塞式I/O，它与阻塞式I/O的区别在于阻塞的系统调用不同。而异步I/O的两个阶段都不会阻塞进程。

可以看出，I/O模式中1，2，3都属于同步IO的操作，因为其在等待数据的时候都有I/O操作完成之前都会被阻塞。而只有4属于异步IO操作(AIO)。

在Linux下有两种称为AIO的的接口。一个是由glibc提供，是由多线程来模拟：数据等待和数据复制的工作，由glibc创建线程来完成。数据复制完成后，执行I/O操作的线程通过回调函数的方式通知应用线程（严格来讲，这种方式不能算真正的AIO，因为用来执行实际I/O操作的线程还是阻塞在I/O操作上，只不过从应用进程的角度来看是异步方式的）。另一种是由内核提供的Kernel AIO，可以做到真正的内核异步通知（这种方式对读写方式，写入大小及偏移都有严格的要求），并且不支持网络I/O，其实现原理本质上与下面要介绍的IOCP类似。

还有一种称为IOCP（Input/Output Completion Port）的AIO。从实现原理上讲，IOCP做完I/O操作后，将结果封装成完成包（completion packet）入队到完成端口的队列（FIFO）中去，应用线程从队列中读取到完成消息后，处理后续逻辑。从这方面来讲，IOCP类似生产者-消费者模型：生产者为内核，收到应用线程的I/O请求后，等待数据可用，并将结果数据复制到应用线程指定的缓冲区中后，然后入队一个完成消息；消费者为应用线程，一开始向内核提交I/O请求，并在队列上等待内核的完成消息（只不过，IOCP对同时可运行的消费者有限制），收到完成消息后，进行后续处理。

### Python的实现

#### Python非阻塞IO

介绍`Select`模块：该模块可以访问大多数操作系统中的`select()`和`poll()`函数， Linux2.5+支持的epoll()和大多数BSD支持的kqueue()。

`select()`方法：该方法监听多个文件描述符的数组，当其返回的时候，系统内核就会将数组中已经就绪的文件描述符修改其标志位，使得进程可以获得这些文件描述符并进行相应的操作。注意，改方法在单个进程内监听的文件描述符数量存在限制，在Linux下一般是1024。

`poll()`方法：与`select()`几乎一样，但是不存在数量上的限制。

`poll`和`select`同样存在一个缺点就是，包含大量文件描述符的数组被整体复制于用户态和内核的地址空间之间，而不论这些文件描述符是否就绪，它的开销随着文件描述符数量的增加而线性增大。另外，`select()`和`poll()`将就绪的文件描述符告诉进程后，如果进程没有对其进行IO操作，那么下次调用`select()`和`poll()`的时候 将再次报告这些文件描述符，所以它们一般不会丢失就绪的消息，这种方式称为水平触发（Level Triggered）。不过`poll`并不适用于windows平台。

Python中的`select`模块以列表形式接受四个参数，分别是需要监控的可读文件对象，可写文件对象，产生异常的文件对象和超时设置(可省略)，当监控的对象发生变化时，select会返回发生变化的对象列表。

以下是`select.select()`的官方文档:

> `select.select(rlist, wlist, xlist[, timeout])`  
> This is a straightforward interface to the Unix select() system call. The first three arguments are sequences of ‘waitable objects’: either integers representing file descriptors or objects with a parameterless method named fileno() returning such an integer:  
>
> <br />
> `rlist`: wait until ready for reading  
> `wlist`: wait until ready for writing  
> `xlist`: wait for an “exceptional condition” (see the manual page for what your system considers such a condition)  
> <br />
> Empty sequences are allowed, but acceptance of three empty sequences is platform-dependent. (It is known to work on Unix but not on Windows.) The optional timeout argument specifies a time-out as a floating point number in seconds. When the timeout argument is omitted the function blocks until at least one file descriptor is ready. A time-out value of zero specifies a poll and never blocks.  
> <br />
>
> The return value is a triple of lists of objects that are ready: subsets of the first three arguments. When the time-out is reached without a file descriptor becoming ready, three empty lists are returned.  
> <br />
>
> Among the acceptable object types in the sequences are Python file objects (e.g. sys.stdin, or objects returned by open() or os.popen()), socket objects returned by socket.socket(). You may also define a wrapper class yourself, as long as it has an appropriate fileno() method (that really returns a file descriptor, not just a random integer).

我们此处使用网上得到的一个聊天室来讲解：

```
#!/usr/bin/env python
#encoding:utf-8
import socket
import select
import sys
import signal
class ChatServer():
  def __init__(self,host,port,timeout=10,backlog=5):
    #记录连接的客户端数量
    self.clients = 0
    #存储连接的客户端socket和地址对应的字典
    self.clientmap = {}
    #存储连接的客户端socket
    self.outputs = []
    #建立socket
    self.server=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    self.server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    self.server.bind((host, port))
    self.server.listen(backlog)
    #增加信号处理
    signal.signal(signal.SIGINT, self.sighandler)

  def sighandler(self):
    sys.stdout.write("Shutdown Server......\n")
    #向已经连接客户端发送关系信息，并主动关闭socket
    for output in self.outputs:
      output.send("Shutdown Server")
      output.close()
    #关闭listen
    self.server.close()
    sys.stdout.flush()

  #主函数，用来启动服务器
  def run(self):

    #需要监听的可读对象
    inputs = [self.server]

    runing = True
    #添加监听主循环
    while runing:
      try:
         #此处会被select模块阻塞，只有当监听的三个参数发生变化时，select才会返回
        readable, writeable, exceptional = select.select(inputs, self.outputs, [])
      except select.error,e:
        break
      #当返回的readable中含有本地socket的信息时，表示有客户端正在请求连接
      if self.server in readable:
        #接受客户端连接请求
        client,addr=self.server.accept()
        sys.stdout.write("New Connection from %s\n"%str(addr))
        sys.stdout.flush()
        #更新服务器上客户端连接情况
        #1，数量加1
        #2，self.outputs增加一列
        #3，self.clientmap增加一对
        #4, 给input添加可读监控
        self.clients += 1
        #添加写入对象
        self.outputs.append(client)
        self.clientmap[client] = addr
        inputs.append(client)

      #readable中含有已经添加的客户端socket，并且可读
      #说明 1,客户端有数据发送过来或者 2,客户端请求关闭
      elif len(readable) != 0:
        #1, 取出这个列表中的socket
        csock = readable[0]
        #2, 根据这个socket，在事先存放的clientmap中，去除客户端的地址，端口的详细信息
        host,port = self.clientmap[csock]
        #3,取数据, 或接受关闭请求，并处理
        #注意，这个操作是阻塞的，但是由于数据是在本地缓存之后，所以速度会非常快
        try:
          data = csock.recv(1024).strip()
          for cs in self.outputs:
            if cs != csock:
              cs.send("%s\n"%data)
        except socket.error,e:
          self.clients -= 1
          inputs.remove(csock)
          self.outputs.remove(csock)
          del self.clientmap[csock]
      #print self.outputs
    self.server.close()

if __name__ == "__main__":
  chat=ChatServer("",8008)
  chat.run()

```

可以看出这里`select.select()`所选择的对象均是通道, 此时多个客户端可以同时进行通话，而不需要等待其他客户端。这里实验客户端可以使用`telnet`来进行操作

#### Python的异步IO

Python 3.4标准库有一个新模块`asyncio`，用来支持异步I/O。

`asyncio`的编程模型就是一个消息循环。我们从`asyncio`模块中直接获取一个`EventLoop`的引用，然后把需要执行的协程扔到`EventLoop`中执行，就实现了异步IO。

实例：  

```
import threading
import asyncio

@asyncio.coroutine
def hello():
    print('Hello world! (%s)' % threading.currentThread())
    yield from asyncio.sleep(1)
    print('Hello again! (%s)' % threading.currentThread())

loop = asyncio.get_event_loop()
tasks = [hello(), hello()]
loop.run_until_complete(asyncio.wait(tasks))
loop.close()

#Result:
Hello world! (<_MainThread(MainThread, started 140735195337472)>)
Hello world! (<_MainThread(MainThread, started 140735195337472)>)
(暂停约1秒)
Hello again! (<_MainThread(MainThread, started 140735195337472)>)
Hello again! (<_MainThread(MainThread, started 140735195337472)>)
```

`@asyncio.coroutine`把一个`generator`标记为`coroutine`类型，然后，我们就把这个`coroutine`扔到`EventLoop`中执行。

`yield from`语法可以让我们方便地调用另一个`generator`。由于`asyncio.sleep()`也是一个`coroutine`，所以线程不会等待`asyncio.sleep()`，而是直接中断并执行下一个消息循环。当`asyncio.sleep()`返回时，线程就可以从`yield from`拿到返回值（此处是`None`），然后接着执行下一行语句。

把`asyncio.sleep(1)`看成是一个耗时1秒的IO操作，在此期间，主线程并未等待，而是去执行`EventLoop`中其他可以执行的`coroutine`了，因此可以实现并发执行。

由打印的当前线程名称可以看出，两个`coroutine`是由同一个线程并发执行的。

如果把`asyncio.sleep()`换成真正的IO操作，则多个`coroutine`就可以由一个线程并发执行。

### 尝试回答

1. 同步与异步的区别在于请求是否会立即返回，所谓同步非阻塞即是指，若请求无结果，进程不阻塞住，但是依然会隔断时间来观察 I/O 操作是否完成，所以本质上，I/O 操作依然是同步的。而异步则是调用之后直接返回，由操作系统告知进程 I/O 操作是否结束，如信号驱动。
2. 阻塞于非阻塞关注点在于，程序是否会等待结果返回。
3. 其实所谓 IO 多路复用，就是事件驱动，操作系统通过 select/poll/epoll/kqueue 等系统调用可检测 socket 描述符是否准备就绪，多个描述符的 I/O 操作都能在一个线程内并发交替地顺序完成，这就叫I/O多路复用，这里的“复用”指的是复用同一个线程。


### 参考文档

[python模块介绍- select 等待I/0完成](http://my.oschina.net/u/1433482/blog/191211)  
[asyncio](http://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/001432090954004980bd351f2cd4cc18c9e6c06d855c498000)
