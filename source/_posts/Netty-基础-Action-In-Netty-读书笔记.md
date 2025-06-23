---
title: Netty-基础(Action In Netty 读书笔记)
date: 2025-06-23 21:29:29
tags:
    - Netty
---

## Netty 的主要构成组件
### Channel
Channel 是 Java NIO 的一个基本构造。 

它代表**一个到实体**(如一个硬件设备、一个文件、一个网络套接字或者一个能够执行一个或者多个不同的I/O操作的程序组件)**的开放连接，如读操作和写操作**。 

目前，**可以把 Channel 看作是传入(入站)或者传出(出站)数据的载体**。因此，它可以被打开或者被关闭，连接或者断开连接。

<!-- more -->

#### 生命周期
![](/images/netty-basic/img-1.png)

1. ChannelUnregistered：Channel 已经被创建，但还未注册到 EventLoop
2. ChannelRegistered：Channel 已经被注册到了 EventLoop
3. ChannelActive：Channel 处于活动状态(已经连接到它的远程节点)。它现在可以接收和发送数据了
4. ChannelInactive：Channel没有连接到远程节点

#### EmbeddedChannel
Netty 专 门为改进针对 ChannelHandler 的单元测试而提供的。将入站数据或者出站数据写入到 EmbeddedChannel 中，然后检 查是否有任何东西到达了 ChannelPipeline 的尾端。以这种方式，你便可以确定消息是否已 经被编码或者被解码过了，以及是否触发了任何的 ChannelHandler 动作。

### Future
Netty 提供了它自己的 Future 实现——ChannelFuture，**用于在执行异步操作的时候使用。**

ChannelFuture提供了几种额外的方法，这些方法使得我们**能够注册一个或者多个 ChannelFutureListener实例**。监听器的回调方法**operationComplete()**，将会在对应的操作完成时被调用。然后监听器可以判断该操作是成功地完成了还是出错了。如果是后者，我们可以检索产生的 Throwable。简而言之 ，**由ChannelFutureListener提供的通知机制消除了手动检查对应的操作是否完成的必要**。

每个 Netty 的出站 I/O 操作都将返回一个 ChannelFuture；也就是说，**它们都不会阻塞**。 正如我们前面所提到过的一样，Netty 完全是异步和事件驱动的。

### 事件和 ChannelHandler
Netty 在内部**使用了回调来处理事件**；当一个回调被触发时，相关的事件可以被一个 interface- **ChannelHandler** 的实现处理。

Netty 使用不同的事件来通知我们状态的改变或者是操作的状态。这使得我们能够基于已经发生的事件来触发适当的动作。这些动作可能是:

+ 记录日志;
+ 数据转换;
+ 流控制;
+ 应用程序逻辑。

Netty 是一个网络编程框架，所以事件是按照它们与入站或出站数据流的相关性进行分类的。可能由入站数据或者相关的状态更改而触发的事件包括:

+ 连接已被激活或者连接失活;
+ 数据读取;
+ 用户事件;
+ 错误事件

出站事件是未来将会触发的某个动作的操作结果，这些动作包括:

+ 打开或者关闭到远程节点的连接; 
+ 将数据写到或者冲刷到套接字。

**每个 ChannelHandler 的实例都类似于一种为了响应特定事件而被执行的回调。**

#### 生命周期
这些 方法中的每一个都接受一个 ChannelHandlerContext 参数。

1. handlerAdded：当把 ChannelHandler 添加到 ChannelPipeline 中时被调用
2. handlerRemoved：当从 ChannelPipeline 中移除 ChannelHandler 时被调用
3. exceptionCaught：当处理过程中在 ChannelPipeline 中有错误产生时被调用

#### ChannelInboundHandler
`ChannelInboundHandler`属于 `ChannelHandler`的子接口，专门处理入站事件，例如数据读取（`channelRead`）、通道激活（`channelActive`）等方法。

**ChannelInboundHandlerAdaptor** 提供了对 ChannelInboundHandler 的空实现，仅实现自己感兴趣的事件即可，避免冗余代码。

以下是`ChannelInboundHandler` 提供的主要事件方法及作用：

1. `channelRegistered(ChannelHandlerContext ctx)`  
当通道被注册到事件循环组（EventLoopGroup）时触发。可用于**初始化**通道相关逻辑 。
2. `channelUnregistered(ChannelHandlerContext ctx)`  
当通道从事件循环组中注销时触发，通常发生在**通道关闭后** 。
3. `channelActive(ChannelHandlerContext ctx)`  
当通道变为活跃状态（即**连接建立成功**）时触发，常用于处理连接建立后的逻辑 。
4. `channelInactive(ChannelHandlerContext ctx)`  
当通道变为非活跃状态（如**连接断开**）时触发，可用于清理资源或重连逻辑 。
5. `channelRead(ChannelHandlerContext ctx, Object msg)`  
**接收入站数据时触发**，是**核心**的数据处理方法。用户需在此方法中实现业务逻辑，并显式释放资源（如 `ByteBuf`） 。
6. `channelReadComplete(ChannelHandlerContext ctx)`  
**当一次完整的入站数据读取完成后触发**，通常**用于刷新缓冲区或执行后续操作** 。
7. `userEventTriggered(ChannelHandlerContext ctx, Object evt)`  
**自定义用户事件**触发时调用，**例如空闲超时检测或心跳包处理 **。
8. `channelWritabilityChanged(ChannelHandlerContext ctx)`  
当通道的**可写状态发生变化时触发**，可**用于控制流量或调整写入策略** 。
9. `handlerAdded(ChannelHandlerContext ctx)`  
当当前处理器被添加到 `ChannelPipeline` 时触发，**常用于初始化资源** 。
10. `handlerRemoved(ChannelHandlerContext ctx)`  
当当前处理器从 `ChannelPipeline` 移除时触发，**可用于资源清理** 。
11. `exceptionCaught(ChannelHandlerContext ctx, Throwable cause)`  
当入站事件传播过程中发生异常时触发，**用于统一处理错误 **。

#### ChannelOutboundHandler
`ChannelOutboundHandler` 是 Netty 中用于处理**出站事件**的核心接口，定义了与出站操作相关的方法。以下是其提供的主要事件方法及作用：

1. `bind(ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise)`  
**当通道绑定到本地地址时触发**（例如服务器启动时绑定端口）。可用于拦截绑定操作或修改绑定逻辑 。
2. `connect(ChannelHandlerContext ctx, SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise)`  
当通道**发起连接**到远程地址时触发（例如客户端连接服务器）。可在此方法中处理连接前的预操作或日志记录 。
3. `disconnect(ChannelHandlerContext ctx, ChannelPromise promise)`  
当通道**断开连接**时触发。适用于清理资源或记录断开事件 。
4. `close(ChannelHandlerContext ctx, ChannelPromise promise)`  
当通道**关闭**时触发。通常用于释放资源或执行关闭后的回调逻辑 。
5. `write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise)`  
当数据**被写入通道时触发。这是核心的出站事件方法**，常用于修改发送的数据、添加日志或处理异常 。
6. `flush(ChannelHandlerContext ctx)`  
当通道缓冲区中的数据**被刷新到远程节点时触发**。可用于优化网络传输效率或执行批量刷新操作 。
7. `read(ChannelHandlerContext ctx)`  
当请求**从通道读取数据时触发**（例如手动触发读操作）。通常由入站事件间接引发，但属于出站处理器的处理范围 。
8. `release(ChannelHandlerContext ctx, Object msg)`  
当**释放出站消息（如 **`**ByteBuf**`**）的资源时触发**。用于显式管理内存释放逻辑，避免内存泄漏 。
9. `exceptionCaught(ChannelHandlerContext ctx, Throwable cause)`  
当**出站事件传播过程中发生异常时触发**。可用于统一处理出站操作中的错误 。

##### 适配器简化实现
直接实现 `ChannelOutboundHandler` 需要覆盖所有方法，因此 Netty 提供了 `ChannelOutboundHandlerAdapter` 作为适配器类，为所有方法提供默认空实现。用户只需重写感兴趣的方法即可，例如：  

```java
public class MyOutboundHandler extends ChannelOutboundHandlerAdapter {
    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
        // 修改或记录出站数据
        ctx.write(msg.retain(), promise); // 示例：保留消息引用计数
    }
}
```

通过 `ctx.write()` 或 `ctx.flush()` 可显式将事件传递给下一个处理器 。

#### 资源管理
每当通过调用 `ChannelInboundHandler.channelRead()`或者 `ChannelOutboundHandler.write()`方法来处理数据时，你都需要确保没有任何的资源泄漏。Netty 使用引用计数来处理池化的 ByteBuf。所以在完全使用完某个 ByteBuf 后，调整其引用计数是很重要的。

为了帮助你诊断潜在的(资源泄漏)问题，Netty提供了class **ResourceLeakDetector**， 它将对你应用程序的缓冲区分配做**大约 1%的采样来检测内存泄露**。相关的开销是非常小的。

如果检测到了内存泄露，将会产生类似于下面的日志消息:

```java
LEAK: ByteBuf.release() was not called before it's garbage-collected. 
Enable advanced leak reporting to find out where the leak occurred. 
To enable advanced leak reporting, specify the JVM option 
'-Dio.netty.leakDetectionLevel=ADVANCED' or call ResourceLeakDetector.setLevel().
```

泄漏检测级别

java -Dio.netty.leakDetectionLevel=SIMPLE

1. DISABLED：禁用泄漏检测。只有在详尽的测试之后才应设置为这个值
2. SIMPLE：使用 1%的默认采样率检测并报告任何发现的泄露。这是默认级别，适合绝大部分的情况
3. ADVANCED：使用默认的采样率，报告所发现的任何的泄露以及对应的消息被访问的位置
4. PARANOID：类似于ADVANCED，但是其将会对每次(对消息的)访问都进行采样。这对性能将会有很大的影响，应该只在调试阶段使用



当你通过 Netty 发送或者接收一个消息的时候，就将会发生一次数据转换。**入站消息会被解码**; 也就是说从字节转换为另一种格式，通常是一个 Java 对象。如果是**出站消息，则会发生相反方向的转换**: 它将从它的当前格式被编码为字节。这两种方向的转换的原因很简单:网络数据总是一系列的字节。

#### 解码器
##### ByteToMessageDecoder
将字节解码为消息(或者另一个字节序列)是一项如此常见的任务，以至于 Netty 为它提供了一个 抽象的基类:`**ByteToMessageDecoder**`。由于你不可能知道远程节点是否会一次性地发送一个完整 的消息，所以这个类会对入站数据进行缓冲，直到它准备好处理。下面是它的两个重要方法：

+ `**decode( ChannelHandlerContextctx, ByteBuf in, List<Object> out)**`

必须实现的唯一抽象方法。`decode()`方法被调用时将会传入一个包含了传入数据的 `ByteBuf`，以及一个用来添加解码消息的 List。对这个方法的调用将会重复进行，直到确定没有新的元素被添加到该 List，或者该 ByteBuf 中没有更多可读取的字节时为止。然后，**如果该 List 不为空，那么它的内容将会被传递给 ChannelPipeline 中的下一个 ChannelInboundHandler。**

+ `**decodeLast( ChannelHandlerContextctx, ByteBuf in,List<Object> out)**`

Netty提供的这个默认实现只是简单地调用了decode()方法。 当Channel的状态变为非活动时，这个方法将会被调用一次。 可以重写该方法以提供特殊的处理。

```java
public class ToIntegerDecoder extends ByteToMessageDecoder { 
    @Override
    public void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
        if (in.readableBytes() >= 4) {
            out.add(in.readInt());
        }
    }
}
```

##### ReplayingDecoder
ReplayingDecoder扩展了ByteToMessageDecoder类，使得我们不必调用`readableBytes()`方法。它通过使用一个自定义的ByteBuf实现， ReplayingDecoderByteBuf，包装传入的ByteBuf实现了这一点，其将在内部执行该调用。

`public abstract class ReplayingDecoder<S> extends ByteToMessageDecoder`

其中 `S`指定了用于状态管理的类型，其中 Void 代表不需要状态管理。

```java
public class ToIntegerDecoder2 extends ReplayingDecoder<Void> { 
    @Override
    // 如果没有足够的字节可用，这 个readInt()方法的实现将会抛出一个Error，
    // 其将在基类中被捕获并处理。当有更多的数据可供读取时，该decode()方法将会被再次调用
    public void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
        out.add(in.readInt());
    }
}
```

##### MessageToMessageDecoder
`public abstract class MessageToMessageDecoder<I> extends ChannelInboundHandlerAdapter`

类型参数 I 指定了 `decode()` 方法的输入参数 msg 的类型，它是你必须实现的唯一方法。

+ `decode(ChannelHandlerContext ctx, I msg, List<Object> out)`

对于每个需要被解码为另一种格式的入站消息来说，该方法都将会被调用。解码消息随后会被传递给 ChannelPipeline 中的下一个 ChannelInboundHandler

```java
public class IntegerToStringDecoder extends MessageToMessageDecoder<Integer> {
    @Override
    public void decode(ChannelHandlerContext ctx, Integer msg
        List<Object> out) throws Exception {
        out.add(String.valueOf(msg));
    } 
}
```

#### 编码器
##### MessageToByteEncoder
将 Message(泛型) 转换成 Bytes

+ `encode(ChannelHandlerContext ctx, I msg, ByteBuf out)`

`encode()`方法是你需要实现的唯一抽象方法。它被调用时将会传入要被该类编码为 ByteBuf 的(类型为 I 的)出站 消息。该 ByteBuf 随后将会被转发给 ChannelPipeline 中的下一个 ChannelOutboundHandler。

这个类只有一个方法，而解码器有两个。原因是**解码器通常需要在 Channel 关闭之后产生最后一个消息(因此也就有了 decodeLast()方法)。**这显然不适用于编码器的场景——在连接被关闭之后仍然产生一个消息是毫无意义的。

```java
public class ShortToByteEncoder extends MessageToByteEncoder<Short> { 
    @Override
    public void encode(ChannelHandlerContext ctx, Short msg, ByteBuf out) throws Exception {
        out.writeShort(msg);
    }
}
```

##### MessageToMessageEncoder
出站数据将如何从一种消息编码为另一种。

+ `encode(ChannelHandlerContext ctx, I msg, List<Object> out)`

需要实现的唯一方法。每个通过 write() 方法写入的消息都将会被传递给 encode() 方法，以编码为一个或者多个出站消息。随后，这些出站消息将会被转发给ChannelPipeline 中的下一个ChannelOutboundHandler

```java
public class IntegerToStringEncoder extends MessageToMessageEncoder<Integer> {
    @Override
    public void encode(ChannelHandlerContext ctx, Integer msg
        List<Object> out) throws Exception {
        out.add(String.valueOf(msg)); 
    }
}
```

#### 编解码器
同一个类中管理 入站和出站数据和消息的转换是很有用的。Netty 的抽象编解码器类正好用于这个目的，因为它们每 个都将捆绑一个解码器/编码器对，以处理我们一直在学习的这两种类型的操作。

##### ByteToMessageCodec
##### MessageToMessageCodec
##### CombinedChannelDuplexHandler
个类充当了 ChannelInboundHandler 和 ChannelOutboundHandler(该类的类型 参数 I 和 O)的容器。通过提供分别继承了解码器类和编码器类的类型，我们可以实现一个编解码器，而又不必直接扩展抽象的编解码器类。

#### SimpleChannelInboundHandler<T>
Netty 中主要用于处理业务逻辑的可扩展基类。

其中 T 是你要处理的消息的 Java 类型 。在这个 ChannelHandler 中， 你将需要重写基类的一个或者多个方法，并且获取一个到 ChannelHandlerContext 的引用， 这个引用将作为输入参数传递给 ChannelHandler 的所有方法。

在这种类型的 ChannelHandler 中，最重要的方法是 channelRead0(Channel- HandlerContext,T)。除了要求不要阻塞当前的 I/O 线程之外，其具体实现完全取决于你。

#### 预制的 ChannelHandler 与编解码器
+ SslHandler：支持 SSL/TLS，使用 Java 提供的 javax.net.ssl 包，它的 SSLContext 和 SSLEngine 类使得实现解密和加密相当简单直接。
+ HttpRequestEncoder：将 HttpRequest、HttpContent 和 LastHttpContent 消息编码为字节
+ HttpResponseEncoder：将 HttpResponse、HttpContent 和 LastHttpContent 消息编码为字节
+ HttpRequestDecoder：将字节解码为 HttpRequest、HttpContent 和 LastHttpContent 消息
+ HttpResponseDecoder：将字节解码为 HttpResponse、HttpContent 和 LastHttpContent 消息

##### 连接管理
+ IdleStateHandler：当连接空闲时间太长时，将会触发一个 IdleStateEvent 事件。然后， 你可以通过在你的 ChannelInboundHandler 中重写 userEventTriggered()方法来处理该 IdleStateEvent 事件
+ ReadTimeoutHandler：如果在指定的时间间隔内没有收到任何的入站数据，则抛出一个 ReadTimeoutException 并关闭对应的 Channel。可以通过重写你的 ChannelHandler 中的 exceptionCaught()方法来检测该 ReadTimeoutException
+ WriteTimeoutHandler：如果在指定的时间间隔内没有任何出站数据写入，则抛出一个 WriteTimeoutException 并关闭对应的 Channel。可以通过重写你的 ChannelHandler 的 exceptionCaught()方法检测该 WriteTimeoutException

##### 基于分隔符的协议和基于长度的协议
+ DelimiterBasedFrameDecoder：使用任何由用户提供的分隔符来提取帧的通用解码器
+ LineBasedFrameDecoder：提取由行尾符(\n 或者\r\n)分隔的帧的解码器。这个解码器比 DelimiterBasedFrameDecoder 更快
+ FixedLengthFrameDecoder：提取在调用构造函数时指定的定长帧
+ LengthFieldBasedFrameDecoder：根据编码进帧头部中的长度值提取帧;该字段的偏移量以及 长度在构造函数中指定

#### 写大型数据
在写大型数据时，需要准备好处理到远程节点的连接是慢速连接的情况，这种情况会导致内存释放的延迟。

利用 NIO 的零拷贝特性，使用一个 FileRegion 接口的实现，其在 Netty 的 API 文档中的定义是: “通过支持零拷贝的文件传输的 Channel 来发送的文件区域。”

```java

FileInputStream in = new FileInputStream(file); 
FileRegion region = new DefaultFileRegion(in.getChannel(), 0, file.length()); 
channel.writeAndFlush(region).addListener(
    new ChannelFutureListener() {
        @Override
        public void operationComplete(ChannelFuture future)
             throws Exception {
             if (!future.isSuccess()) {
                Throwable cause = future.cause();
                // Do something 处理失败 
             }
        }
});
```

只适用于文件内容的直接传输，不包括应用程序对数据的任何处理。

在需要将数据从文件系统复制到用户内存中时，可以使用 `ChunkedWriteHandler`，它支持异步写大型数据 流，而又不会导致大量的内存消耗。

关键是 interface ChunkedInput<B>，其中类型参数 B 是 readChunk()方法返回的 类型。Netty 预置了该接口的 4 个实现，下面每个都代表了一个将由 ChunkedWriteHandler 处理的不定长度的数据流。

+ ChunkedFile：从文件中逐块获取数据，当你的平台不支持零拷贝或者你需要转换数据时使用
+ ChunkedNioFile： 和 ChunkedFile 类似，只是它使用了 FileChannel
+ ChunkedStream：从 InputStream 中逐块传输内容
+ ChunkedNioStream：从ReadableByteChannel中逐块传输内容

#### 序列化数据
##### Protocol Buffers
+ ProtobufDecoder：使用 protobuf 对消息进行解码
+ ProtobufEncoder：使用 protobuf 对消息进行编码
+ ProtobufVarint32FrameDecoder：根据消息中的 Google Protocol Buffers 的“Base 128 Varints”a 整型长度字段值动态地分割所接收到的 ByteBuf
+ ProtobufVarint32LengthFieldPrepender：向ByteBuf前追加一个GoogleProtocalBuffers的“Base 128 Varints”整型的长度字段值

### ChannelPipeline
ChannelPipeline 提供了** ChannelHandler 链的容器**，并**定义了用于在该链上传播入站和出站事件流的 API**。当 Channel 被创建时，它会被自动地分配到它专属的 ChannelPipeline。

ChannelHandler 安装到 ChannelPipeline 中的过程如下所示: 

+ 一个ChannelInitializer的实现被注册到了 ServerBootstrap 中;
+ 当 ChannelInitializer.initChannel()方法被调用时，ChannelInitializer 将在 ChannelPipeline 中安装一组自定义的 ChannelHandle
+ ChannelInitializer 将它自己从 ChannelPipeline 中移除。

**通常 ChannelPipeline 中的每一个 ChannelHandler 都是通过它的 EventLoop(I/O 线程)来处理传递给它的事件的。所以至关重要的是不要阻塞这个线程，因为这会对整体的 I/O 处理产生负面的影响。**

##### ChannelPipeline的入站操作
+ fireChannelRegistered：调用 ChannelPipeline 中下一个 ChannelInboundHandler 的 channelRegistered(ChannelHandlerContext)方法
+ fireChannelUnregistered：调用 ChannelPipeline 中下一个 ChannelInboundHandler 的 channelUnregistered(ChannelHandlerContext)方法
+ fireChannelActive：调用 ChannelPipeline 中下一个 ChannelInboundHandler 的 channelActive(ChannelHandlerContext)方法
+ fireChannelInactive：调用 ChannelPipeline 中下一个 ChannelInboundHandler 的 channelInactive(ChannelHandlerContext)方法
+ fireExceptionCaught：调用 ChannelPipeline 中下一个 ChannelInboundHandler 的 exceptionCaught(ChannelHandlerContext, Throwable)方法
+ fireUserEventTriggered：调用 ChannelPipeline 中下一个 ChannelInboundHandler 的 userEventTriggered(ChannelHandlerContext, Object)方法
+ fireChannelRead：调用 ChannelPipeline 中下一个 ChannelInboundHandler 的 channelRead(ChannelHandlerContext, Object msg)方法
+ fireChannelReadComplete：调用 ChannelPipeline 中下一个 ChannelInboundHandler 的 channelReadComplete(ChannelHandlerContext)方法
+ fireChannelWritabilityChanged：调用 ChannelPipeline 中下一个 ChannelInboundHandler 的 channelWritabilityChanged(ChannelHandlerContext)方法

##### ChannelPipeline的出站操作
+ bind：将 Channel 绑定到一个本地地址，这将调用 ChannelPipeline 中的下一个 ChannelOutboundHandler 的 bind(ChannelHandlerContext, Socket- Address, ChannelPromise)方法
+ connect：将 Channel 连接到一个远程地址，这将调用 ChannelPipeline 中的下一个 ChannelOutboundHandler 的 connect(ChannelHandlerContext, Socket- Address, ChannelPromise)方法
+ disconnect：将 Channel 断开连接。这将调用 ChannelPipeline 中的下一个 ChannelOutbound- Handler 的 disconnect(ChannelHandlerContext, Channel Promise)方法
+ close：将 Channel 关闭。这将调用 ChannelPipeline 中的下一个 ChannelOutbound- Handler 的 close(ChannelHandlerContext, ChannelPromise)方法
+ deregister：将 Channel 从它先前所分配的 EventExecutor(即 EventLoop)中注销。这将调 用 ChannelPipeline 中的下一个 ChannelOutboundHandler 的 deregister (ChannelHandlerContext, ChannelPromise)方法
+ flush：冲刷 Channel 所有挂起的写入。这将调用 ChannelPipeline 中的下一个 ChannelOutboundHandler 的 flush(ChannelHandlerContext)方法
+ write：将消息写入 Channel。这将调用 ChannelPipeline 中的下一个 ChannelOutboundHandler 的 `write(ChannelHandlerContext, Object msg, ChannelPromise)`方法。注意:这并不会将消息写入底层的 Socket，而只会将它放入队列中。 要将它写入 Socket，需要调用 flush()或者 writeAndFlush()方法
+ writeAndFlush：这是一个先调用write()方法再接着调用flush()方法的便利方法
+ read：请求从 Channel 中读取更多的数据。这将调用 ChannelPipeline 中的下一个ChannelOutboundHandler 的 read(ChannelHandlerContext)方法

##### ChannelHandlerContext
ChannelHandlerContext **使得 ChannelHandler 能够和它的 ChannelPipeline 以及其他的 ChannelHandler 交 互 。** ChannelHandler 可 以 通 知 其 所 属 的 ChannelPipeline 中 的 下 一 个 ChannelHandler，**甚至可以动态修改它所属的ChannelPipeline**。

ChannelHandlerContext 具有丰富的用于处理事件和执行 I/O 操作的 API。

+ alloc：返回和这个实例相关联的 Channel 所配置的 ByteBufAllocator
+ bind：绑定到给定的 SocketAddress，并返回 ChannelFuture
+ channel：返回绑定到这个实例的 Channel
+ close：关闭 Channel，并返回 ChannelFuture
+ connect：连接给定的 SocketAddress，并返回 ChannelFuture
+ deregister：从之前分配的 EventExecutor 注销，并返回 ChannelFuture
+ disconnect：从远程节点断开，并返回 ChannelFuture
+ executor：返回调度事件的 EventExecutor
+ fireChannelActive：触发对下一个 ChannelInboundHandler 上的 channelActive()方法(已连接)的调用
+ fireChannelInactive：触发对下一个 ChannelInboundHandler 上的 channelInactive()方法(已关闭)的调用
+ fireChannelRead：触发对下一个 ChannelInboundHandler 上的 channelRead()方法(已接收的消息)的调用
+ fireChannelReadComplete：触发对下一个 ChannelInboundHandler 上的 channelReadComplete()方法的调用
+ fireChannelRegistered：触发对下一个 ChannelInboundHandler 上的 fireChannelRegistered()方法的调用
+ fireChannelUnregistered：触发对下一个 ChannelInboundHandler 上的 fireChannelUnregistered()方法的调用
+ fireChannelWritabilityChanged：触发对下一个 ChannelInboundHandler 上的fireExceptionCaught(Throwable)方法的调用
+ fireUserEventTriggered：触发对下一个 ChannelInboundHandler 上的 fireUserEventTriggered(Object evt)方法的调用
+ handler：返回绑定到这个实例的 ChannelHandler
+ isRemoved：如果所关联的 ChannelHandler 已经被从 ChannelPipeline 中移除则返回 true
+ name：返回这个实例的唯一名称
+ pipeline：返回这个实例所关联的 ChannelPipeline
+ read：将数据从Channel读取到第一个入站缓冲区;如果读取成功则触发一个channelRead事件，并(在最后一个消息被读取完成后) 通 知 ChannelInboundHandler 的 channelReadComplete (ChannelHandlerContext)方法
+ write：通过这个实例写入消息并经过 ChannelPipeline
+ writeAndFlush：通过这个实例写入并冲刷消息并经过 ChannelPipeline

**当使用 ChannelHandlerContext 的 API 的时候，请牢记以下两点: **

+ ChannelHandlerContext 和 ChannelHandler 之间的关联(绑定)是永远不会改变的，所以缓存对它的引用是安全的; 
+ 相对于其他类的同名方法，ChannelHandlerContext 的方法将产生更短的事件流，应该尽可能地利用这个特性来获得最大的性能。

重要的 是要注意到，虽然被调用的 Channel 或 ChannelPipeline 上的 write()方法将一直传播事件通 过整个 ChannelPipeline，但是在 ChannelHandler 的级别上，**事件从一个 ChannelHandler 到下一个 ChannelHandler 的移动是由 ChannelHandlerContext 上的调用完成的。**

![](/images/netty-basic/img-2.png)

为什么会想要从 ChannelPipeline 中的某个特定点开始传播事件呢? 

+ 为了减少将事件传经对它不感兴趣的 ChannelHandler 所带来的开销
+ 为了避免将事件传经那些可能会对它感兴趣的 ChannelHandler

##### 异常处理
你应该如何响应异常，可能很大程度上取决于你的应用程序。你可能想要关闭Channel(和连接)，也可 能会尝试进行恢复。如果你不实现任何处理入站异常的逻辑(或者没有消费该异常)， 那么Netty将会记录该异常没有被处理的事实 。

总结一下: 

+ ChannelHandler.exceptionCaught()的默认实现是简单地将当前异常转发给ChannelPipeline 中的下一个 ChannelHandler;
+ 如果异常到达了 ChannelPipeline 的尾端，它将会被记录为未被处理;
+ 要想定义自定义的处理逻辑，你需要重写 exceptionCaught()方法。然后你需要决定是否需要将该异常传播出去。

### EventLoop 与 线程模型
EventLoop 就是之前介绍 Reactor 并发模式中的 Reactor

EventLoop 定义了 Netty 的核心抽象，用于**处理连接的生命周期中所发生的事件**。

![](/images/netty-basic/img-3.png)

+ NioEventLoopGroup：基于 Java NIO 的 EventLoopGroup 实现，适用于多平台环境。通过 Selector 实现非阻塞 I/O 操作 
+ OioEventLoopGroup：用于处理阻塞 I/O（OIO，Old I/O） 的 EventLoopGroup 实现。无法支持高并发。
+ EpollEventLoopGroup：专为 Linux 平台 设计的高性能 EventLoopGroup 实现，基于 Epoll 事件驱动模型 
+ KQueueEventLoopGroup：基于 BSD 系统（如 macOS） 的 kqueue 机制实现的 EventLoopGroup，提供类似 Epoll 的高性能非阻塞 I/O 支持 。
+ LocalEventLoopGroup：Netty 提供的一种线程池实现。更适合用于执行本地任务，如定时任务、异步回调、事件通知等，通常不涉及 I/O 操作。

![](/images/netty-basic/img-4.png)

在这个模型中，一个 EventLoop 将由一个永远都不会改变的 Thread 驱动，同时任务 (Runnable 或者 Callable)可以直接提交给 EventLoop 实现，以立即执行或者调度执行。 根据配置和可用核心的不同，可能会创建多个 EventLoop 实例用以优化资源的使用，并且单个EventLoop 可能会被指派用于服务多个 Channel。 

需要注意的是，Netty的EventLoop在继承了ScheduledExecutorService的同时，只定义了一个方法，parent() 用于返回到当前EventLoop实 现的实例所属的EventLoopGroup的引用。

#### 任务调度
你将需要调度一个任务以便稍后(延迟)执行或者周期性地执行。一个常见的用例是，发送心跳消息到远程 节点，以检查连接是否仍然还活着。如果没有响应，你便知道可以关闭该 Channel 了。

ScheduledExecutorService 的实现具有局限性，例如，事实上作为线程池管理的一部 分，将会有额外的线程创建。如果有大量任务被紧凑地调度，那么这将成为一个瓶颈。Netty 通 过 Channel 的 EventLoop 实现任务调度解决了这一问题。

```java
Channel ch = ...
ScheduledFuture<?> future = ch.eventLoop().schedule(
        new Runnable() {
        @Override
        public void run() {
System.out.println("60 seconds later"); }
}, 60, TimeUnit.SECONDS);
```

经过 60 秒之后，Runnable 实例将由分配给 Channel 的 EventLoop 执行。

### 引导-Boostrap
引导一个应用程序是指对它进行配置，并使它运行起来的过程。

两种类型的引导:

+ 一种用于客户端(简单地称为 Bootstrap)。仅需要绑定一个 EventLoopGroup，用于处理业务逻辑。
+ 另一种 (ServerBootstrap)用于服务器。需要绑定两个 EventLoopGroup，用于接受连接和处理业务逻辑。

![](/images/netty-basic/img-5.png)

无论你的应用程序使用哪种协议或者处理哪种类型的数据，唯一决定它使用哪种引导类的是它是作为一个客户端还是作为一个服务器。

服务器致力于使用一个父 Channel 来接受 来自客户端的连接，并创建子 Channel 以用于它们之间的通信；而客户端将最可能只需要一个单独的、没有父 Channel 的 Channel 来用于所有的网络交互。

### ByteBuf
Netty 的 ByteBuffer 替代品是 ByteBuf。

ByteBuf API 的优点: 

+ 它可以被用户自定义的缓冲区类型扩展
+ 通过内置的复合缓冲区类型实现了透明的零拷贝
+ 容量可以按需增长(类似于 JDK 的 StringBuilder)
+ 在读和写这两种模式之间切换不需要调用 ByteBuffer 的 flip()方法
+ 读和写使用了不同的索引
+ 支持方法的链式调用
+ 支持引用计数
+ 支持池化

#### 堆缓冲区
将数据存储在 JVM 的堆空间中。它能在没有使用池化的情况下提供快速的分配和释放。

#### 直接缓冲区
为了避免在每次调用本地 I/O 操作之前(或者之后)将缓冲区的内容复制到一个中间缓冲区(或者从中间缓冲区把内容复制到缓冲区)。直接缓冲区对于网络数据传输是理想的选择。

直接缓冲区的主要缺点是，相对于基于堆的缓冲区，它们的分配和释放都较为昂贵。如果你正在处理遗留代码，你也可能会遇到另外一个缺点 :因为数据不是在堆上，所以你不得不进行一次复制。

#### 复合缓冲区
为多个 ByteBuf 提供一个聚合视图。

## 基于 Netty 设计的 Server 应该怎么实现
**所有的 Netty 服务器都需要以下两部分。**

+ **至少一个ChannelHandler**：该组件实现了服务器对从客户端接收的数据的处理，即**它的业务逻辑**。
+ **引导**：这是配置服务器的启动代码。至少，它会将服务器绑定到它要监听连接请求的端口上。

```java
public class NettyServer {
    public static void main(String[] args) throws Exception {
        // 1. 创建两个线程组 bossGroup 和 workerGroup
        // bossGroup 专门负责 接收客户端的连接请求 （即 accept 操作），它监听端口并处理新连接的建立。
        // workerGroup 专门负责 已建立连接的 I/O 读写操作 （即 read/write 操作），包括数据的接收、处理和发送 
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        try {
            // 2. 创建服务器端启动对象
            ServerBootstrap bootstrap = new ServerBootstrap();
            
            // 3. 配置启动参数
            bootstrap.group(bossGroup, workerGroup)
                     .channel(NioServerSocketChannel.class) // 设置通道类型 [[4]]
                     .option(ChannelOption.SO_BACKLOG, 128) // 设置线程队列大小
                     .childOption(ChannelOption.SO_KEEPALIVE, true) // 保持连接
                     .childHandler(new ChannelInitializer<SocketChannel>() {
                         @Override
                         protected void initChannel(SocketChannel ch) {
                             // 4. 添加自定义处理器 [[7]]
                             ch.pipeline().addLast(new NettyServerHandler());
                         }
                     });

            System.out.println("Server is ready...");

            // 5. 绑定端口并启动服务器 [[1]]
            ChannelFuture future = bootstrap.bind("0.0.0.0", 8888).sync();
            
            // 6. 等待服务器关闭 [[6]]
            future.channel().closeFuture().sync();
        } finally {
            // 7. 优雅关闭线程组
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}

// 自定义处理器
class NettyServerHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        // 8. 处理客户端消息 [[9]]
        System.out.println("Server received: " + msg);
        // 回复客户端
        ctx.writeAndFlush("Hello from server");
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        // 9. 异常处理
        cause.printStackTrace();
        ctx.close();
    }
}
```



## 基于 Netty 设计的 Client 应该怎么实现
```java
EventLoopGroup group = new NioEventLoopGroup(); 
// 创建一个 Bootstrap 以创建和连接新的 Client Channel
Bootstrap bootstrap = new Bootstrap(); 
// 设置 EventLoopGroup 提供用于处理 Channel 事件的 EventLoop
bootstrap.group(group)
    .channel(NioSocketChannel.class) // 指定要使用的 Channel 实现
    // 设置用于 Channel 的事件和数据的 ChannelHandler
    .handler(new SimpleChannelInboundHandler<ByteBuf>() {
        @Override
            protected void channeRead0(
                ChannelHandlerContext channelHandlerContext, ByteBuf byteBuf) throws Exception { 
                System.out.println("Received data");
            }
    });
// 连接到远程主机
ChannelFuture future = bootstrap.connect(
    new InetSocketAddress("www.manning.com", 80)); 
future.addListener(new ChannelFutureListener() {
    @Override
    public void operationComplete(ChannelFuture channelFuture)
        throws Exception {
        if (channelFuture.isSuccess()) {
            System.out.println("Connection established"); 
        } else {
            System.err.println("Connection attempt failed"); 
            channelFuture.cause().printStackTrace();
        }
    }
} );
```

## 案例分析-Nifty和Swift
Thrify（RPC 框架） 的 Netty 版本。主要组件:

+ **Thrift 的接口定义语言(IDL)**——用来定义你的服务，并且编排你的服务将要发送和接收的任何自定义类型;
+ **协议**——用来控制将数据元素编码/解码为一个通用的二进制格式(如 Thrift 的二进制协议或者 JSON);
+ **传输**— 提供了一个用于读/写不同媒体(如 TCP 套接字、管道、内存缓冲区)的通用接口;
+ **Thrift 编译器**——解析 Thrift 的 IDL 文件以生成用于服务器和客户端的存根代码，以及在IDL 中定义的自定义类型的序列化/反序列化代码;
+ **服务器实现**— 处理接受连接、从这些连接中读取请求、派发调用到实现了这些接口的对象，以及将响应发回给客户端;
+ **客户端实现**——将方法调用转换为请求，并将它们发送给服务器。

Github 地址（已经废弃）：[https://github.com/facebookarchive/nifty](https://github.com/facebookarchive/nifty)




