---
title: Kafka-Network-阅读
date: 2021-10-23 15:45:04
tags:
    - Kafka
---

## JavaNio 及 EventLoop
![单个线程监听多个网络事件](/images/kafka_network/image_1.png "单个线程监听多个网络事件")


<!-- more -->

在 Java NIO 中最为重要的三个概念

- Selector：用于管理多个 Channel（通过 `register` 方法来注册 `event`来实现）。主要方法
   - `open`：static 构建函数，构建一个 `Selector`，用于屏蔽操作系统实现细节。
   - `SelectableChannel#register(Channel, ops)`：将 `Channel` 注册到 `Selector`中，并返回 `SelectionKey`作为标识
   - `selectedKeys()`：给到所有注册过的 `Channel` 的 `SelectionKey`
   - `select(timeout) 与 selectNow()`：block 请求，返回已经 ready 的事件数量
   - `wakeup()`：唤醒被 `select` 阻塞的线程
- Channel：对于 IO 的抽象，包括 `FileChannel`、`ServerSocketChannel`、`SocketChannel`
   - `read(Buffer)`：使用 `Buffer`读取`Channel`的数据
   - `write(Buffer)`：将 `Buffer` 的数据写入到 `Channel`中
- Buffer：用于与 `Channel`读写交互数据使用

![read 与 write 的 pos / limit / cap 关系](/images/kafka_network/image_2.png "read 与 write 的 pos / limit / cap 关系")
### SimpleExample
```java
public static void main(String[] args) throws Exception {
    String closeDemo = "close demo";
    Selector selector = Selector.open();
    ServerSocketChannel serverSocket = ServerSocketChannel.open();
    serverSocket.bind(new InetSocketAddress("localhost", 5454));
    serverSocket.configureBlocking(false);
    serverSocket.register(selector, SelectionKey.OP_ACCEPT);
    ByteBuffer buffer = ByteBuffer.allocate(256);
    while (true) {
        selector.select();
        Set<SelectionKey> selectedKeys = selector.selectedKeys();
        Iterator<SelectionKey> iter = selectedKeys.iterator();
        while (iter.hasNext()) {
            SelectionKey key = iter.next();
            int interestOps = key.interestOps();
            System.out.println(interestOps);
            if (key.isAcceptable()) {
                SocketChannel client = serverSocket.accept();
                client.configureBlocking(false);
                client.register(selector, SelectionKey.OP_READ);
            }
            if (key.isReadable()) {
                SocketChannel client = (SocketChannel) key.channel();
                client.read(buffer);
                if (new String(buffer.array()).trim().equals(closeDemo)) {
                    client.close();
                    System.out.println("Not accepting client messages anymore");
                }
                buffer.flip();
                client.write(buffer);
                buffer.clear();
            }
            iter.remove();
        }
    }
}
```
## Kafka 中 NetworkClient 对于 JavaNio 的封装
其中也有对应的 `Selector`、`KafkaChannel` 但是还多出了 `TransportLayer` 接口。

1. `Selector` 中的 `connect`方法会将每个 `node` 都关联一个 `KafkaChannel`到一个 `SocketChannel` 然后包装成 `TrnsportLayer` 并注册到 `nioSelector` 中
2. `KafkaChannel` 用来持有 `TransportLayer`即 `SocketChannel` 
3. `TransportLayer` 是在传输协议上的封装，主要是为了鉴权使用。
## Kafka 中 NetworkClient 是如何发送数据的？
在 `NetworkClient`中对于 `Request` 的发送主要使用了三个方法

- `ready`用于建立与 node 的连接，创建一个 `KafkaChannel`
- `send`用于将 data 数据缓存到 `KafkaChannel` 中，并将 `SocketChannel`的 ops 增加为 `WRITE`
- `poll`的方法就比较复杂
   - 先读：读 `KafkaChannel`中等待读的数据，将其缓存到 `Selector`中的 `completedReceives `数据结构中
   - 再写：将 `KafkaChannel`中缓存的数据写入到网络中，然后将写入成功的数据放入 `completedSends ` 数据结构中
   - 处理等待的请求，如果已经成功收到 broker 的返回结果对请求发起回调（如果存在）
### Ready



![image.png](/images/kafka_network/image_3.png)

1. 当 `Sender`调用 `ready`的时候如果对应 node 没有与 client 建立连接，则会通过 `NetworkClient#initiateConnect `方法建连.
2. 其中建连最为重要的则是 `Selector#connect` 方法会新建一个 `OPEN` 的 `SocketChannel(JDK)` 并通过该 `SocketChannel`与 `node`的 `address` 建连。
3. 在建连之后会将该 `SocketChannel`注册到 `Selector`对应的 `NioSelector(JDK)`中并得到一个 `SelectionKey`，这时候会创建一个 `KafkaChannel`（`Selector#registerChannel`） ，让 `SelectionKey` attach 该 `KafkaChannel`，这样方便在存在 IO 通知的时候获取对应节点的 `KafkaChannel`，每个 `node`都将拥有一个 `KafkaChannel`用于管理网络事件。
4. `TransportLayer` 主要是处理 auth 之间的差异，同时提供了方法用来包装 `SocketChannel` 以提供给 `KafkaChannel`使用
### Send
需要注意，在 `RequestBuilder` 组成 `ClientRequest`之后会生成 `Send`接口，这个 `Send` 可以简单理解为对于要发送 `bytes` 的一个反向依赖。
同时在 `NetworkClient#send` 方法中会将 `ClientRequest`加入到 `inFlightRequests`中
![image.png](/images/kafka_network/image_4.png)
在 `NetworkClient`中将要发送的请求 `Send`set 到 `KafkaChannel`的成员变量中，然后通过 `TransportLayer`将对应 node 的 `SocketChannel` 设置为 `OPEN_WRITE`
### Poll


![image.png](/images/kafka_network/image_5.png)

1. 在调用 `poll`之后会直接调用 `Selector#poll` 然后通过 `nioSelector#select` 获取所有已经准备就绪的 `KafkaChannel`
2. 针对每个 `KafkaChannel`会走一段鉴权逻辑，如果鉴权已经通过 `KafkaChannel#ready()` 方法会返回 `true`
3. 此时会尝试先读，如果 `Channel` 的状态是 `isReadable `则调用 `attemptRead` 方法将得到的数据 通过 `NetworkReceived` 存入 `Channel` 中
4. 然后尝试写，如果 `Channel`的状态是 `isWritable`则会将数据通过 `TransportLayer `写入到 `SocketChannel`中
5. 最后在 `NetworkClient`中调用 `handlerXXX` 方法处理各种从网络中得到的请求，对于请求完成的（读到 response）调用回调。 
