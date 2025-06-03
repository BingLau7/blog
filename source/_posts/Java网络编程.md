---
title: Java网络编程-基础
date: 2025-06-03 11:23:25
tags:
    - Java
---

## 阻塞式编程
这里给出一个 TCP 简单的使用阻塞式网络编程的案例，为后面的 NIO 做个介绍的基础。

<!--more-->

```java
public class TCPServer {
    public static void main(String[] args) {
        int port = 12345; // 服务端监听的端口号

        try (ServerSocket serverSocket = new ServerSocket(port)) {
            System.out.println("服务端正在监听端口: " + port);

            while (true) { // 循环等待客户端连接
                try {
                    Socket clientSocket = serverSocket.accept();
                    System.out.println("客户端已连接: " + clientSocket.getInetAddress());

                    // 启动线程处理客户端通信
                    new ClientHandler(clientSocket).start();

                } catch (IOException e) {
                    System.err.println("客户端连接处理异常");
                    e.printStackTrace();
                }
            }

        } catch (IOException e) {
            System.err.println("服务端启动失败");
            e.printStackTrace();
        }
    }

    // 内部类：用于处理每个客户端连接
    private static class ClientHandler extends Thread {
        private final Socket clientSocket;

        public ClientHandler(Socket socket) {
            this.clientSocket = socket;
        }

        @Override
        public void run() {
            try (
                BufferedReader in = new BufferedReader(new InputStreamReader(clientSocket.getInputStream()));
            ) {
                String inputLine;
                while ((inputLine = in.readLine()) != null) {
                    System.out.println("收到客户端消息: " + inputLine);
                }
            } catch (IOException e) {
                System.out.println("客户端断开连接: " + clientSocket.getInetAddress());
            } finally {
                try {
                    clientSocket.close();
                } catch (IOException e) {
                    // Ignore
                }
            }
        }
    }
}

public class TCPClient {
    public static void main(String[] args) {
        String hostName = "localhost"; // 服务端 IP 地址
        int port = 12345; // 服务端监听的端口号

        try (Socket socket = new Socket(hostName, port)) {
            // 获取输出流
            PrintWriter out = new PrintWriter(socket.getOutputStream(), true);

            // 向服务端发送消息
            out.println("你好，服务端！");

            // 关闭资源
            out.close();
        } catch (UnknownHostException e) {
            System.err.println("未知主机: " + hostName);
            e.printStackTrace();
        } catch (IOException e) {
            System.err.println("I/O 异常");
            e.printStackTrace();
        }
    }
}
```

在传统的 Socket 编程中，实际 Java 依赖的是两个主要的类

+ Server 端依赖 `ServerSocket` 来监听端口，通过 accpet 来阻塞并接收请求，通常的编程模式是将每个请求作为一个单独的 Thread 来进行处理。
+ Client 端依赖 `Socket`来进行连接。

如果仅是测试场景或者非高并发场景，传统的 Socket 编程或许已经足够了，但是基于以下一些理由，在网络编程中 SocketAPI 通常不会成为首选：

1. **阻塞式 I/O：**每个连接都需要一个线程处理，导致资源浪费，难以支撑高并发。
2. **性能瓶颈明显：**线程数受限于系统资源，连接数增加时容易出现性能问题甚至崩溃。

更优的选择一般有两个：

+ Java 提供的 NIO 编程（比如 Kafka 的选择）
+ Netty （绝大多数的选择）

## NIO 编程
Java NIO类库包含以下三个核心组件：

+ Channel（通道）
+ Buffer（缓冲区）
+ Selector（选择器）

### NIO 的编程模式
#### Server
1. 初始化阶段
    1. 创建 `Selector` 实例，用于监听事件。
    2. 创建 `ServerSocketChannel` 并绑定端口，设置为非阻塞模式。
    3. 将 `ServerSocketChannel` 注册到 `Selector` 上，关注 `OP_ACCEPT` 事件（客户端连接请求）。
2. 事件循环阶段（主逻辑）
    1. 进入一个无限循环，调用 `selector.select()` 阻塞等待事件发生。
    2. 当有事件触发时，遍历所有触发的 `SelectionKey`：
        1. 如果是 `OP_ACCEPT`：说明有新的客户端连接，接受连接并创建`SocketChannel`，注册到 `Selector`，关注 `OP_READ`。
        2. 如果是 `OP_READ`：说明客户端发送了数据，从 `SocketChannel` 中读取数据，并进行业务处理（如打印或回复消息）。
    3. 每次处理完一个事件后，需要手动将该 `SelectionKey` 从集合中移除，避免重复处理。
3. 关闭资源
    1. 客户端断开连接时，关闭对应的 SocketChannel，并取消其在 Selector 中的注册。
    2. 服务端保持运行，继续监听新连接。

```java
public class NIOServer {
public static void main(String[] args) throws IOException {
    int port = 12345;

    // 1.a 创建实例
    Selector selector = Selector.open();
    ServerSocketChannel serverChannel = ServerSocketChannel.open();
    // 1.b 绑定端口并设置非阻塞
    serverChannel.bind(new InetSocketAddress(port));
    serverChannel.configureBlocking(false);
    // 1.c 监听 OP_ACCEPT 事件
    serverChannel.register(selector, SelectionKey.OP_ACCEPT);

    System.out.println("NIO 服务端启动，监听端口: " + port);

    // 2. 事件循环等待
    while (true) {
        int readyCount = selector.select(); // 阻塞直到有事件发生
        if (readyCount == 0) continue;

        Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
        
        while (iterator.hasNext()) {
            SelectionKey key = iterator.next();
            // 2.b.i OP_ACCEPT 事件，注册 SocketChannel
            if (key.isAcceptable()) {
                ServerSocketChannel serverSocketChannel = (ServerSocketChannel) key.channel();
                SocketChannel clientChannel = serverSocketChannel.accept();
                if (clientChannel != null) {
                    clientChannel.configureBlocking(false);
                    clientChannel.register(selector, SelectionKey.OP_READ);
                    System.out.println("客户端连接: " + clientChannel.getRemoteAddress());
                }
            }

            // 2.b.ii OP_READ 事件，读取 Socket 数据
            if (key.isReadable()) {
                SocketChannel clientChannel = (SocketChannel) key.channel();
                ByteBuffer buffer = ByteBuffer.allocate(1024);
                int readBytes = clientChannel.read(buffer);

                // 3. 关闭客户端链接
                if (readBytes == -1) {
                    System.out.println("客户端断开: " + clientChannel.getRemoteAddress());
                    clientChannel.close();
                    key.cancel();
                    continue;
                }

                buffer.flip();
                byte[] data = new byte[buffer.remaining()];
                buffer.get(data);
                String message = new String(data);
                System.out.println("收到客户端消息: " + message);

                // 回复客户端
                String response = "服务端已收到: " + message;
                clientChannel.write(ByteBuffer.wrap(response.getBytes()));
            }

            iterator.remove();
        }
    }
}
}

```

#### Client
1. 建立连接
    1. 创建 SocketChannel 并设置为非阻塞模式。
    2. 调用 connect() 发起连接请求。
    3. 循环调用 finishConnect() 直到连接完成。
2. 数据发送
    1. 使用 write() 方法将数据写入 SocketChannel，发送给服务端。
3. 接收响应（可选）
    1. 使用 read() 方法从 SocketChannel 读取服务端的响应数据。
    2. 将数据从 ByteBuffer 中提取出来并解析成字符串输出。
4. 关闭连接
    1. 手动调用 close() 关闭 SocketChannel。

```java
public static void main(String[] args) throws IOException {
    // 1. 建连
    SocketChannel socketChannel = SocketChannel.open();
    socketChannel.configureBlocking(false);
    socketChannel.connect(new InetSocketAddress("localhost", 12345));

    while (!socketChannel.finishConnect()) {
        System.out.println("正在连接...");
    }

    System.out.println("客户端连接成功");

    // 2. 发送消息
    String message = "你好，NIO 服务端！";
    ByteBuffer buffer = ByteBuffer.wrap(message.getBytes());
    socketChannel.write(buffer);

    // 3. 接收响应
    ByteBuffer responseBuffer = ByteBuffer.allocate(1024);
    int bytesRead = socketChannel.read(responseBuffer);
    if (bytesRead > 0) {
        responseBuffer.flip();
        byte[] data = new byte[responseBuffer.remaining()];
        responseBuffer.get(data);
        System.out.println("收到服务端回复: " + new String(data));
    }

    // 4. 关闭连接
    socketChannel.close();
}
}

```

## Reactor 模式
**Reactor 模式** 是一种广泛应用于高性能网络服务器的 **事件驱动架构设计模式**，特别适用于使用 **非阻塞 I/O（如 Java NIO）** 的场景。它的核心思想是：**将事件分发与业务处理分离**，从而实现高并发、低延迟的网络通信。

#### Reactor 模式的组成
Reactor 模式通常包含以下几个核心组件：

| 组件 | 作用 |
| --- | --- |
| `Reactor` | 主线程，负责监听和分发事件（如连接请求、读写事件）。 |
| `Acceptor` | 当 Reactor 接收到客户端连接事件（OP_ACCEPT），由 Acceptor 负责接受连接并注册读事件。 |
| `Handler` | 处理具体的 I/O 事件（如读取数据、发送响应），可以是单线程或多线程。 |
| `Selector` | 多路复用器，用于监听多个 Channel 上的事件（Java NIO 提供）。 |


---

#### Reactor 模式的工作流程（以 TCP 网络服务为例）
1. **启动 Reactor**
    - 创建 `Selector`，绑定端口，监听客户端连接事件（OP_ACCEPT）。
2. **客户端连接**
    - Reactor 检测到 OP_ACCEPT 事件，交给 Acceptor 处理。
    - Acceptor 接受连接，创建新的 SocketChannel，并注册到 Selector 中，监听读事件（OP_READ）。
3. **客户端发送数据**
    - Reactor 检测到 OP_READ 事件，找到对应的 Handler 并触发读操作。
4. **Handler 处理数据**
    - Handler 从 SocketChannel 中读取数据，进行业务逻辑处理（如解析协议、计算、回复等）。
5. **发送响应**
    - Handler 将处理结果写回客户端（可异步或同步）。

#### Reactor 的不同变体
根据线程模型的不同，Reactor 模式有以下几种常见实现方式：

##### 单 Reactor 单线程（最简单）
+ 所有事件都在一个线程中处理（包括 Accept、Read、Write）。
+ 适合轻量级服务或学习用途。
+ 缺点：在单线程Reactor模式中，Reactor和Handler都在同一条线程中执行。这样，带来了一个问题：**当其中某个Handler阻塞时，会导致其他所有的Handler都得不到执行。**在这种场景下，被阻塞的Handler不仅仅负责输入和输出处理的传输处理器，还包括负责新连接监听的AcceptorHandler处理器，可能导致服务器无响应。



```java
public class SingleThreadReactorServer {
    private final Selector selector;
    private final ServerSocketChannel serverSocketChannel;

    public SingleThreadReactorServer(int port) throws IOException {
        selector = Selector.open();
        serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.bind(new InetSocketAddress(port));
        serverSocketChannel.configureBlocking(false);
        // 注册事件到对应 Handler
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT, new Acceptor());
    }

    // 单线程运行
    public void run() throws IOException {
        while (true) {
            // 单个 Selector 监听事件
            selector.select();
            Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
            while (iterator.hasNext()) {
                SelectionKey key = iterator.next();
                iterator.remove();
                // 分发监听事件
                dispatch(key);
            }
        }
    }

    private void dispatch(SelectionKey key) {
        // 监听到时间之后对时间进行处理，但是依然是单线程的
        Runnable handler = (Runnable) key.attachment();
        if (handler != null) handler.run();
    }

    private class Acceptor implements Runnable {
        @Override
        public void run() {
            try {
                SocketChannel clientChannel = serverSocketChannel.accept();
                clientChannel.configureBlocking(false);
                // 注册读数据事件
                clientChannel.register(selector, SelectionKey.OP_READ, new ClientHandler(clientChannel));
                System.out.println("客户端连接");
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    // 读数据 Handler
    private static class ClientHandler implements Runnable {
        private final SocketChannel channel;
        private final ByteBuffer buffer = ByteBuffer.allocate(1024);

        public ClientHandler(SocketChannel channel) {
            this.channel = channel;
        }

        @Override
        public void run() {
            try {
                int read = channel.read(buffer);
                if (read == -1) {
                    channel.close();
                    return;
                }
                buffer.flip();
                byte[] data = new byte[buffer.remaining()];
                buffer.get(data);
                System.out.println("收到消息: " + new String(data));
                buffer.clear();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) throws IOException {
        new SingleThreadReactorServer(12345).run();
    }
}
```

##### 单 Reactor 多线程
+ Reactor 线程负责监听和分发事件。
+ Handler 使用线程池来处理业务逻辑，提升并发能力。
+ 适合中等规模的服务。

```java
public class MultiThreadReactorServer {
    private final Selector selector;
    private final ServerSocketChannel serverSocketChannel;
    private final ExecutorService pool = Executors.newCachedThreadPool();

    public MultiThreadReactorServer(int port) throws IOException {
        selector = Selector.open();
        serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.bind(new InetSocketAddress(port));
        serverSocketChannel.configureBlocking(false);
        // 注册连接事件
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT, new Acceptor());
    }

    public void run() throws IOException {
        while (true) {
            // 依然是单 Selector 处理事件
            selector.select();
            Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
            while (iterator.hasNext()) {
                SelectionKey key = iterator.next();
                iterator.remove();
                dispatch(key);
            }
        }
    }

    private void dispatch(SelectionKey key) {
        // 与单线程不同的是，每个 Key 都是多线程(ExecutorService)处理事件
        Runnable handler = (Runnable) key.attachment();
        if (handler != null) pool.submit(handler);
    }

    private class Acceptor implements Runnable {
        @Override
        public void run() {
            try {
                SocketChannel clientChannel = serverSocketChannel.accept();
                clientChannel.configureBlocking(false);
                clientChannel.register(selector, SelectionKey.OP_READ, new ClientHandler(clientChannel));
                System.out.println("客户端连接");
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    private static class ClientHandler implements Runnable {
        private final SocketChannel channel;
        private final ByteBuffer buffer = ByteBuffer.allocate(1024);

        public ClientHandler(SocketChannel channel) {
            this.channel = channel;
        }

        @Override
        public void run() {
            try {
                int read = channel.read(buffer);
                if (read == -1) {
                    channel.close();
                    return;
                }
                buffer.flip();
                byte[] data = new byte[buffer.remaining()];
                buffer.get(data);
                System.out.println("收到消息: " + new String(data));
                buffer.clear();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) throws IOException {
        new MultiThreadReactorServer(12345).run();
    }
}

```

##### 多 Reactor 多线程（Netty 默认采用）
+ 一个 Main Reactor 负责监听 Accept 事件。
+ 多个 Sub Reactor 分别负责不同的客户端连接（每个 Reactor 一个线程）。
+ 实现真正的高并发、高性能网络服务。
+ 适合大型分布式系统、网关、消息中间件等。

```java
public class MultiReactorServer {
    private final Selector bossSelector;
    private final ServerSocketChannel serverSocketChannel;
    private final Selector[] workerSelectors = new Selector[4];
    private final SubReactor[] reactors = new SubReactor[4];

    public MultiReactorServer(int port) throws IOException {
        bossSelector = Selector.open();
        serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.bind(new InetSocketAddress(port));
        serverSocketChannel.configureBlocking(false);

        // 多个 Selector，区分 Selector 的类型将处理连接和处理业务逻辑的 Selector 分开
        for (int i = 0; i < workerSelectors.length; i++) {
            workerSelectors[i] = Selector.open();
            reactors[i] = new SubReactor(workerSelectors[i]);
            new Thread(reactors[i]).start();
        }

        // bossSelector 注册到 Acceptor Handler 中
        serverSocketChannel.register(bossSelector, SelectionKey.OP_ACCEPT, new Acceptor());
        // 等价
        // SelectionKey key = serverSocketChannel.register(bossSelector, SelectionKey.OP_ACCEPT);
        // key.attach(new Acceptor());
    }

    public void run() throws IOException {
        // 单线程处理连接时间
        while (true) {
            bossSelector.select();
            Iterator<SelectionKey> iterator = bossSelector.selectedKeys().iterator();
            while (iterator.hasNext()) {
                SelectionKey key = iterator.next();
                iterator.remove();
                dispatch(key);
            }
        }
    }

    private void dispatch(SelectionKey key) {
        // 多线程处理 Key
        Runnable handler = (Runnable) key.attachment();
        if (handler != null) handler.run();
    }

    private class Acceptor implements Runnable {
        private int next = 0;

        @Override
        public void run() {
            try {
                SocketChannel clientChannel = serverSocketChannel.accept();
                clientChannel.configureBlocking(false);
                // 多 Selector 注册 clientChannel，将时间分发到 SubReactor 中
                reactors[next].register(clientChannel);
                next = (next + 1) % reactors.length;
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) throws IOException {
        new MultiReactorServer(12345).run();
    }

    // SubReactor 类
    private static class SubReactor implements Runnable {
        private final Selector selector;

        public SubReactor(Selector selector) {
            this.selector = selector;
        }

        public void register(SocketChannel channel) throws IOException {
            channel.register(selector, SelectionKey.OP_READ, new ClientHandler(channel));
        }

        @Override
        public void run() {
            try {
                while (true) {
                    // 关注 Client 后续事件
                    selector.select();
                    Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
                    while (iterator.hasNext()) {
                        SelectionKey key = iterator.next();
                        iterator.remove();
                        dispatch(key);
                    }
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

        private void dispatch(SelectionKey key) {
            // 在 SubReactor 线程中处理事件
            Runnable handler = (Runnable) key.attachment();
            if (handler != null) handler.run();
        }
    }

    private static class ClientHandler implements Runnable {
        private final SocketChannel channel;
        private final ByteBuffer buffer = ByteBuffer.allocate(1024);

        public ClientHandler(SocketChannel channel) {
            this.channel = channel;
        }

        @Override
        public void run() {
            try {
                int read = channel.read(buffer);
                if (read == -1) {
                    channel.close();
                    return;
                }
                buffer.flip();
                byte[] data = new byte[buffer.remaining()];
                buffer.get(data);
                System.out.println("收到消息: " + new String(data));
                buffer.clear();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}

```

#### Reactor 模式 vs 传统线程模型
| 对比项 | 传统线程模型 | Reactor 模型 |
| --- | --- | --- |
| 线程数 | 每个连接一个线程 | 极少线程管理大量连接 |
| 资源占用 | 高 | 低 |
| 性能表现 | 连接数增加时下降快 | 可支持数十万并发 |
| 开发难度 | 简单 | 相对复杂 |
| 适用场景 | 小规模应用 | 高性能网络服务（如 Netty、Redis、Nginx） |


#### 实际应用场景举例
+ **Netty**：基于 Reactor 模式构建的高性能网络框架。
+ **Redis**：单 Reactor 多线程模型，支撑极高并发访问。
+ **Nginx**：基于事件驱动的多进程/多线程模型，类似 Reactor 思想。
+ **高性能网关、RPC 框架、消息队列**：普遍使用 Reactor 模式提升吞吐量。

