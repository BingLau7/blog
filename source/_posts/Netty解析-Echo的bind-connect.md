---
title: Netty解析-Echo的bind&connect
date: 2019-01-13 23:07:41
tags:
    - NIO
    - Netty源码分析系列
categories: 源码分析
---


借助 Netty 官方 Echo 实例: [Echo Demo](https://github.com/netty/netty/tree/4.1/example/src/main/java/io/netty/example/echo)

## 问题

1. 如何与底层交互
2. Bootstrap 与 ServerBootstreap 有什么异同
3. NioSocketChannel 与 NioServerSocketChannel 有什么异同
4. 为什么说 netty 是事件驱动，其事件是如何传播的
5. 为什么 ServerBootstrap 需要两个 EventLoopGroup，分别有什么用

<!-- more -->

## 尝试回答

### 如何与底层交互

#### 从 bind 的角度来看

Netty 中与底层网络交互的实际是 io.netty.channel.Channel.Unsafe 类，在各个主要 IO 模式（包括但不限于 NIO、Epoll、KQueue）下均有对其的实现。其中包含了 Server bind 的接口。

```java
/**
 * Bind the {@link SocketAddress} to the {@link Channel} of the {@link ChannelPromise} and notify
 * it once its done.
 */
void bind(SocketAddress localAddress, ChannelPromise promise);
```

怎么调用到的？
在 Echo Demo 中我们选用了 NioServerSocketChannel，在其创建事会一直调用其父类继承链构造器，直到 AbstractChannel，而在其中会创建 Unsafe（NioMessageUnsafe） 与 DefaultChannelPipeline。
在 DefaultChannelPipeline 中存在两个特殊的 ChannelHandlerContext 分别是 head 与 tail，而在 Boostrap 中调用 bind 将会通过 bossGroup 将调用传递到 ChannelPipeline.bind 方法中，这样在经过 head.bind （因为 bind 是一个 outBound 事件，所以会是最后触发 head 的）时候会调用 `unsafe.bind`。这会产生什么效果呢？
在调用 unsafe.bind 实际是在调用 AbstractChannel.AbstractUnsafe.bind(NioMessageUnsafe 父类) 方法，最终又回传到了 NioServerSocketChannel 中的 doBind 方法中，在这里使用了 ServerSocketChannel 的 bind 逻辑，则是 JDK 的方法了。

这么设计有什么优势呢？从 Unsafe 这个名称猜测，是其作为 Channel 辅助类专门管理 Socket 相关接口调用并防止 Netty 上层应用所使用。

#### 从 connect 的角度来看

其实同 bind 一样，当我们追踪 connect 方法时，最终会走到 channel.connect，而在 Client 中我们所选用的是 NioSocketChannel，在其中会创建 Unsafe (NioByteUnsafe)，当链路同样通过 DefaultChannelPipeline 传递到 head.connect 的时候回去调用 NioByteUnsafe.connect 的方法，此时同 bind 一样，最终起到效果的依然是 NioSocketChannel.doConnect 方法，其主要逻辑在于

```java
// ...
boolean connected = SocketUtils.connect(javaChannel(), remoteAddress);
// ...
```

最后依然调用了 SocketChannel.connect 的方法来建立连接。

### Bootstrap 与 ServerBootstrap 有什么异同

ServerBootstrap 比较明显的注意点在，它需要传入两个 EventLoopGroup，我们通常会将第一个称为 bossGroup，第二个称为 workGroup，其中，workGroup 其流程其实与 Bootstrap 的 EventLoop 所起到的作用是一致的，都是充当了 IO 操作的处理现成。这里会涉及到问题五

#### ServerBootstrap 需要两个 EventLoopGroup，分别有什么用

在 Demo 中第一个 EventLoopGroup 也是 bossGroup 用于处理客户端连接请求，而第二个 EventLoopGroup 也是 workerGroup 用于处理与各个客户端连接的 IO 操作。

##### bossGroup 处理 OP_ACCEPT

1. 在我们创建 NioServerSocketChannel 时候会在最后的构造器中传入一个参数，SelectionKey.OP_ACCEPT

```java
/**
 * Create a new instance using the given {@link ServerSocketChannel}.
 */
public NioServerSocketChannel(ServerSocketChannel channel) {
    super(null, channel, SelectionKey.OP_ACCEPT);
    config = new NioServerSocketChannelConfig(this, javaChannel().socket());
}

protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    super(parent);
    this.ch = ch;
    this.readInterestOp = readInterestOp;
    try {
        ch.configureBlocking(false);
    } catch (IOException e) {
        ... // 异常处理
    }
}
```
2. 在 initAndRegister 时候，通过 SingleThreadEventLoop.register 中对于 channel().unsafe() 的 register 调用将 bossGroup 设置到 Channel 的 EventLoop 中.
3. 同样是在 register 逻辑中，当设置完 EventLoop 之后首先在 EventLoop 中注册事件监听到 NioEventLoop 的 selector 中，即 NioServerSocketChannel 中的 `SelectorProvider.openSelector` 中
4. 通知 ChannelPipeline 相关事件
5. 准备 read 事件（AbstractChannel.beginRead 开始）在其中，可以看到若当前 selector 监听的事件无对应 Channel.readInterestOp(即第一步的 OP_ACCEPT) 则对其进行注册
6. 在上述 initAndRegister 结束之后会有调用 bossGroup.execute(AbstractBootstrap.doBind0) 方法执行之前说到与底层交互的 Bind 逻辑，同时还会**在 bossGroup 中开启无限循环(NioEventLoop.this.run())** 处理 IO 事件（processSelectedKeys）这里可以看到，我们会对 OP_ACCEPT 调用 unsafe.read() 方法。
7. 调用了 read 事件自然是需要传递写消息进去的，此时的 unsafe 是 NioMessageUnsafe，有个调用 `doReadMessages` 方法最终被 NioServerSocketChannel 重载

```java
    protected int doReadMessages(List<Object> buf) throws Exception {
        SocketChannel ch = SocketUtils.accept(javaChannel()); // 接收网络连接

        try {
            if (ch != null) {
                buf.add(new NioSocketChannel(this, ch)); // 包装网络连接为 NioSocketChannel
                return 1;
            }
        } catch (Throwable t) {
            ... // 异常处理
        }

        return 0;
    }
```
最终将包装的 NioSocketChannel 传入到 ChannelPipeline 事件流中

接下来就是说明如何处理 OP_ACCEPT 事件到 workerGroup 中了。

##### workerGroup 处理其他网络 IO

1. 回到 initAndRegister 中对于 channel 的 init 方法，该出被 ServerBootstrap 进行了重载
```java
void init(Channel channel) throws Exception {
    ... // 对于 attr 与 option 的设置
    ChannelPipeline p = channel.pipeline();

    final EventLoopGroup currentChildGroup = childGroup;
    final ChannelHandler currentChildHandler = childHandler;
    final Entry<ChannelOption<?>, Object>[] currentChildOptions;
    final Entry<AttributeKey<?>, Object>[] currentChildAttrs;
    synchronized (childOptions) {
        currentChildOptions = childOptions.entrySet().toArray(newOptionArray(0));
    }
    synchronized (childAttrs) {
        currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(0));
    }

    p.addLast(new ChannelInitializer<Channel>() {
        @Override
        public void initChannel(final Channel ch) throws Exception {
            final ChannelPipeline pipeline = ch.pipeline();
            ChannelHandler handler = config.handler();
            if (handler != null) {
                pipeline.addLast(handler);
            }

            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {
                    pipeline.addLast(new ServerBootstrapAcceptor(
                            ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                }
            });
        }
    });
}
```
在这里我们可以看到在 ChannelPipeline 中添加了一个 ServerBootstrapAcceptor 的 ChannelInboundHandlerAdapter，在上述监听到 OP_ACCEPT 事件时会通过 ChannelPipeline 调用其 channelRead 方法
2. 在 channelRead 方法中会将 ServerBootstrap 设置的 channelHandler 给加入 ChannelPipeline 中，然后通过 childGroup(即 workerGroup)来注册到接收的 NioSocketChannel 中等待网络 IO 事件并处理(让注册的 ChannelHandler 处理)。
```java
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    final Channel child = (Channel) msg;

    child.pipeline().addLast(childHandler);

    setChannelOptions(child, childOptions, logger);

    for (Entry<AttributeKey<?>, Object> e: childAttrs) {
        child.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
    }

    try {
        childGroup.register(child).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) {
                    forceClose(child, future.cause());
                }
            }
        });
    } catch (Throwable t) {
        forceClose(child, t);
    }
}
```

---

Bootstrap 其实就是其上面的 worker 的处理逻辑（都是 NioSocketChannel 处理）。

### NioSocketChannel 与 NioServerSocketChannel 有什么异同

它们之前最大的差异体现在其握有的 SocketChannel / ServerSocketChannel。这二者在 Java Nio 中一个是连接到 TCP 网络套接字的通道，另一个是可以监听新进来的 TCP 连接的通道。Netty 中 NioSocketChannel 与 NioServerSocketChannel 其实都是对其二者操作的包装及兼容到 Netty 本身对于协议的操作抽象中来。

### 为什么说 netty 是事件驱动，其事件是如何传播的

首先说说事件驱动，netty 的事件主要是指 inBound 事件与 outBound 事件，inBound 事件通常是 Socket 接收到网络事件然后进行逐层 ChannelHandler 传播，而 outBound 事件则是由用户发起如 bind/connect/read/write 等事件然后在 netty 的 ChannelHandler 中逐层传播。

然后我们可以先看一下 ChannelPipeline 中的一张图

```java
 *                                                 I/O Request
 *                                            via {@link Channel} or
 *                                        {@link ChannelHandlerContext}
 *                                                      |
 *  +---------------------------------------------------+---------------+
 *  |                           ChannelPipeline         |               |
 *  |                                                  \|/              |
 *  |    +---------------------+            +-----------+----------+    |
 *  |    | Inbound Handler  N  |            | Outbound Handler  1  |    |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |              /|\                                  |               |
 *  |               |                                  \|/              |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |    | Inbound Handler N-1 |            | Outbound Handler  2  |    |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |              /|\                                  .               |
 *  |               .                                   .               |
 *  | ChannelHandlerContext.fireIN_EVT() ChannelHandlerContext.OUT_EVT()|
 *  |        [ method call]                       [method call]         |
 *  |               .                                   .               |
 *  |               .                                  \|/              |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |    | Inbound Handler  2  |            | Outbound Handler M-1 |    |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |              /|\                                  |               |
 *  |               |                                  \|/              |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |    | Inbound Handler  1  |            | Outbound Handler  M  |    |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |              /|\                                  |               |
 *  +---------------+-----------------------------------+---------------+
 *                  |                                  \|/
 *  +---------------+-----------------------------------+---------------+
 *  |               |                                   |               |
 *  |       [ Socket.read() ]                    [ Socket.write() ]     |
 *  |                                                                   |
 *  |  Netty Internal I/O Threads (Transport Implementation)            |
 *  +-------------------------------------------------------------------+
```

在 DefaultChannelPipeline 中，有两个特殊的 ChannelHandler，其中一个是 HeadContext 主要用于一些网络操作的处理，同时作为 ChannelOutboundHandler/ChannelInboundHandler，而另一个则是 TailContext 主要用途在于将最后传入到此处若为得到释放的消息给释放掉，作为一个 ChannelInboundHandler。

若是 Inbound 事件则是由 head 开始触发，直到最后触发到 tail 中，期间经历过各种 inbound 类型的 context，执行该 context 的 handler 的对应处理方法，具体体现就是各种在 ChannelInboundInvoker 中的 fireChannel* 方法。而若发生 Outbound 事件则是由 tail 开始触发最终触发到 head 中，期间经历过各种outbound类型的context，执行该context的handler的对应处理方法，具体体现就在各种在 ChannelOutboundInvoker 中的方法。

## 参考文档

[官方文档](https://netty.io/4.1/api/io/netty/channel/Channel.html)
[Netty 源码分析之 一 揭开 Bootstrap 神秘的红盖头 (服务器端)](https://segmentfault.com/a/1190000007283053)
[Netty源码之Channel初始化和注册过程](https://my.oschina.net/7001/blog/1421893)
[深入浅出Netty：NioEventLoop](http://blog.jobbole.com/105564/)
《Netty 权威指南》
《Netty 实战》