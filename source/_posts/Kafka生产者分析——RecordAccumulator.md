---
title: Kafka生产者分析——RecordAccumulator
date: 2017-12-24 19:38:31
tags:
    - Kafka
    - 分布式与中间件
categories: 源码分析
---

>  紧接 [Kafka生产者分析——KafkaProducer](https://binglau7.github.io/2017/12/18/Kafka%E7%94%9F%E4%BA%A7%E8%80%85%E5%88%86%E6%9E%90%E2%80%94%E2%80%94KafkaProducer/)

前文介绍过，KafkaProducer 可以有同步和异步两种方式发送消息，其实两者的底层实现相同，都是通过异步方式实现的。主线程调用 `KafkaProducer#send()` 方法发送消息的时候，先将消息放到 `RecordAccumulator` 中暂存，然后主线程就可以从 `send()` 方法中返回了，此时消息并没有真正地发送给 Kafka，而是缓存在了 `RecordAccumulator` 中。之后，业务线程**通过 `KafkaProducer#send()` 方法不断向 `RecordAccumulator` 追加消息，当达到一定的条件，会唤醒 `Sender` 线程发送 `RecordAccumulator` 中的消息。**

下面我们就来介绍 `RecordAccumulator` 的结构。首先需要注意的是，**`RecordAccumulator` 至少有一个业务线程和一个 `Sender` 线程并发操作，所以必须是线程安全的**。

<!-- more -->

`RecordAccumulator` 中有一个以 `TopicPartition` 为 key 的 `ConcurrentMap` ，每个 value 是 `ArrayDeque<RecordBatch>` （`ArrayDeque` 并不是线程安全的集合，后面会详细介绍其加锁处理过程），其中缓存了发往对应 `TopicPartition` 的消息。每个 `RecordBatch` 拥有一个 `MemoryRecords` 对象的引用。`MemoryRecords` 才是消息最终存放的地方。

![RecordAccumulator&RecordBatch&MemoryRecords](https://github.com/BingLau7/blog/blob/master/images/blog_43/RecordAccumulator&RecordBatch&MemoryRecords.jpg?raw=true)


## `MemoryRecords`

我们从最底层的 `MemoryRecords` 开始分析。`MemoryRecords` 表示的是多个消息的集合，其中封装了 Java NIO ByteBuffer 用来保存消息数据，`Compressor` 用于对 `ByteBuffer` 中的消息进行压缩，以及其他控制字段。其中有四个字段比较重要：

-  buffer：用于保存消息数据的 Java NIO ByteBuffer
-  writeLimit：记录 buffer 字段最多可以写入多少个字节的数据
-  compressor：压缩器，对消息数据进行压缩，将压缩后的数据输出到 buffer。
-  writable：此 `MemoryRecords` 对象是只读的模式，还是可写模式。在 `MemoryRecords` 发送前，会将其设置成只读模式。

在 Compressor 比较重要的字段和方法，有两个输出流类型的字段：

-  `bufferStream`

   在 buffer 上建立的 `ByteBufferOutputStream` (Kafka 自己提供的实现)对象，`ByteBufferOutputStream` 继承了 `java.io.OutputStream` ，封装了 `ByteBuffer`，当写入数据超过 `ByteBuffer` 容量时，`ByteBufferOutputStream` 会进行自动扩容。

-  `appendStream`

   `DataOutputStream` 类型，是对 `bufferStream` 进行了一层装饰，为其添加了压缩的功能。

`MemoryRecords` 中的 `Compressor` 的压缩类型是由 `compression.type` 配置参数指定的，即 `KafkaProducer.compressor.type` 字段的指。

下面来分析一下创建压缩流的方式，目前 `KafkaProducer` 支持 `GZIP、SNAPPY、LZ4` 三种压缩方式

```java
    public Compressor(ByteBuffer buffer, CompressionType type) {
        /* 从 KafkaProducer 传递过来的压缩类型 */
        this.type = type;
		...
        // create the stream
        bufferStream = new ByteBufferOutputStream(buffer);
        /* 下面根据压缩类型创建合适的压缩流 */
        appendStream = wrapForOutput(bufferStream, type, COMPRESSION_DEFAULT_BUFFER_SIZE);
    }

    public static DataOutputStream wrapForOutput(ByteBufferOutputStream buffer, CompressionType type, int bufferSize) {
        try {
            switch (type) {  /* 根据不同的类型选择创建不同压缩流 */
                case NONE: /* 不压缩的方式 */
                    return new DataOutputStream(buffer);
                case GZIP:
                    return new DataOutputStream(new GZIPOutputStream(buffer, bufferSize) /* 使用 JDK 自带的包 */);
                case SNAPPY:
                    try {
                        /* 使用额外引入的依赖包，为了在不使用 Snappy 压缩方式时，减少依赖包，使用反射的方式动态创建(如果不使用反射则会造成为多个压缩方式的第三方依赖做初始化工作的时间损耗) */
                        OutputStream stream = (OutputStream) snappyOutputStreamSupplier.get().newInstance(buffer, bufferSize);
                        return new DataOutputStream(stream);
                    } catch (Exception e) {
                        throw new KafkaException(e);
                    }
                case LZ4:
                    try {
                        OutputStream stream = (OutputStream) lz4OutputStreamSupplier.get().newInstance(buffer);
                        return new DataOutputStream(stream);
                    } catch (Exception e) {
                        throw new KafkaException(e);
                    }
                default:
                    throw new IllegalArgumentException("Unknown compression type: " + type);
            }
        } catch (IOException e) {
            throw new KafkaException(e);
        }
    }
```

`Compressor` 提供了一系列 `put*()` 方法，向 `appendStream` 流写入数据，一个典型的装饰器模式的运用，通过 `bufferStream`  装饰，添加自动扩容的功能；通过 `appendStream` 装饰后，添加压缩功能。

![Compressor装饰器模式](https://github.com/BingLau7/blog/blob/master/images/blog_43/Compressor-putXXX.png?raw=true)

`Compressor.estimateBytesWritten()` 方法的功能是根据指定压缩方式的压缩率，写入的未压缩数据的字节数（`writtenUncompressed` 字段记录）、估算因子（COMPRESSION_RATE_ESTIMATION_FACTOR 字段），估计已写入的（压缩后的）字节数，次方法主要用来判断 `MemoryRecords` 是否写满的逻辑中使用。

下面分析过程暂且认为是使用的非压缩方式（None）。

了解了 `Compressor` 的实现逻辑后，我们回到 `MemoryRecords` 继续分析。`MemoryRecords` 的构造方法是私有的，只能通过 `emptyRecords()` 方法得到其对象。`MemoryRecords` 中有四个比较重要的方法。

-  `append()` 方法：先判断 `MomoryRecords` 是否为可写模式，然后调用 `Compressor.put*()` 方法，将消息数据写入 `ByteBuffer` 中。
-  `hasRoomFor()` 方法：根据 `Compressor` 估计的已写字节数，估计 `MemoryRecords` 剩余空间是否足够写入指定的数据。注意，这里仅仅是估计，所以不一定准确，通过 `hasRoomFor()` 方法判断之后写入数据，也可能就会导致底层 `ByteBuffer` 出现扩容的情况。
-  `close()` 方法：出现 `ByteBuffer` 扩容的情况时，`MemoryRecords.buffer` 字段与 `ByteBufferOutputStream.buffer` 字段所指向的不再是同一个 `ByteBuffer` 对象。在 `close()` 方法中，会将 `MemoryRecords.buffer` 字段指向扩容后的 `ByteBuffer` 对象。同时，将 `writbale` 设置为 `false` （即只读模式）
-  `sizeInBytes()` 方法:对于可写的 `MemoryRecords` ，返回的是 `ByteBufferOutputStream.buffer` 字段的大小；对于只读 `MemoryRecords` ，返回的是 `MemoryRecords.buffer` 的大小。

`MemoryRecords` 还提供了迭代器，主要是用于在 `Consumer` 端读取其中的消息。

## RecordBatch

在了解了 `MemoryRecords` 的具体实现之后，来分析 `RecordBatch` 类的实现。我们在前面所知，每个 `RecordBatch` 对象中封装了一个 `MemoryRecords` 对象，除此之外，还封装了很多控制信息和统计信息，接下来简单介绍一下。

```java
    /* 记录了保存的 Record 的个数 */
    public int recordCount = 0;
    /* 最大 Record 的字节数 */
    public int maxRecordSize = 0;
    /* 尝试发送当前 RecordBatch 的次数 */
    public volatile int attempts = 0;
    public final long createdMs;
    public long drainedMs;
    /* 最后一次尝试发送的时间戳 */
    public long lastAttemptMs;
    /* 指向用来存储数据的 MemoryRecords 对象 */
    public final MemoryRecords records;
    /* 当前 RecordBatch 中缓存的消息都会发送给此 TopicPartition */
    public final TopicPartition topicPartition;
    /* 标识 RecordBatch 状态的 Future 对象 */
    public final ProduceRequestResult produceFuture;
    /* 最后一次向 RecordBatch 追加消息的时间戳 */
    public long lastAppendTime;
    /* Thunk 对象的集合 */
    private final List<Thunk> thunks;
    /* 用来记录某消息在 RecordBatch 中的偏移量 */
    private long offsetCounter = 0L;
    /* 是否正在重试。如果 RecordBatch 中的数据发送失败，则会重新尝试发送 */
    private boolean retry;
```

在下图中，以 `RecordBatch` 为中心，刻画了其相关类间的对应关系。

>  这里推荐一下 idea 的 [Plantuml](http://plantuml.com/sitemap) 这个插件

![RecordBatch](https://github.com/BingLau7/blog/blob/master/images/blog_43/RecordBatch.png?raw=true)

下面我们来分析一下 `ProduceRequestResult` 这个类的功能。`ProduceRequestResult` 并未实现 `java.util.concurrent.Future` 接口，但是其通过包含一个 `count` 值为 1 的 `CountDownLatch` 对象，实现了类似于 `Future` 的功能。

当 `RecordBatch` 中全部的消息被正常响应、或超时、或关闭生产者时，会调用 `ProduceRequestResult.done()` 方法，将 `produceFuture` 标记为完成并通过 `ProduceRequestResult.error` 字段区分『异常完成』还是『正常完成』，之后调用 `CountDownLatch` 对象的 `countDown()` 方法。此时，会唤醒阻塞在 `CountDownLatch` 对象的 `await()` 方法的线程（这些线程通过 `ProduceRequestResult` 的 `await` 方法等待上述三个事件的发送）。

Kafka 的分区会为其中记录的消息分配一个 `offset` 并通过此 `offset` 维护消息顺序。在 `ProduceRequestResult` 中海油一个需要注意的字段 `baseOffset`，表示的是服务的为此 `RecordBatch` 中第一条消息分配的 `offset`，这样每个消息可以根据此 `offset` 以及自身在此 `RecordBatch` 中的相对偏移量，计算出其在服务端分区中的偏移量了。

在介绍 `Thunk` 类之前，先回顾一下 `KafkaProducer.send()` 方法的第二个参数，是一个 `Callback` 对象，它是针对单个消息的回调函数（每个消息对会有一个对应的 `Callback` 对象作为回调）。`RecordBatch.thunks` 字段可以理解为消息的回调对象队列，`Thunk` 中的 `callback` 字段就指向对应消息的 `Callback` 对象，其另一个字段 `future` 是 `FutureRecordMetadata` 类型。`FutureRecordMetadata` 类有两个关键字段。

-  `result`：`ProduceRequestResult` 类型，指向对应消息所在 `RecordBatch` 的 `produceFuture` 字段。
-  `relativeOffset`：`long`类型，记录了对应消息在 `RecordBatch` 中的偏移量。

`FutureRecordMetadata` 实现了 `java.util.concurrent.Future` 接口，但其实现基本都是委托给了 `ProduceRequestResult` 对应的方法，由此可以看出，消息应该是按照 `RecordBatch` 进行发送和确认的。

当生产者已经收到某消息的响应时，`FutureRecordMetadata.get()` 方法就会返回 `RecordMetadata` 对象，其中包含消息在 `Partition` 中的 `offset` 等其他元数据，可供用户自定义 `Callback` 使用。

分析完 `RecordBatch` 依赖的组件，现在回来看看 `RecordBatch` 类的核心方法。`tryAppend()` 方法是最核心的方法，其功能是尝试将消息添加到当前的 `RecordBatch` 中缓存，代码如下所示

```java
    public FutureRecordMetadata tryAppend(long timestamp, byte[] key, byte[] value, Callback callback, long now) {
        /* 估算剩余空间不足，前面说过，这不是一个准确值 */
        if (!this.records.hasRoomFor(key, value)) {
            return null;
        } else {
            /* 向 MemoryRecords 中添加数据。注意，offsetCounter 是在 RecordBatch 中的偏移量 */
            long checksum = this.records.append(offsetCounter++, timestamp, key, value);
            /*  更新统计信息 */
            this.maxRecordSize = Math.max(this.maxRecordSize, Record.recordSize(key, value));
            this.lastAppendTime = now;
            /* 创建 FutureRecordMetadata 对象 */
            FutureRecordMetadata future = new FutureRecordMetadata(this.produceFuture, this.recordCount,
                                                                   timestamp, checksum,
                                                                   key == null ? -1 : key.length,
                                                                   value == null ? -1 : value.length);
            /* 将用户自定义 CallBack 和 FutureRecordMetadata 封装成 Thunk，保存到 thunks 集合中 */
            if (callback != null)
                thunks.add(new Thunk(callback, future));
            this.recordCount++;
            return future;
        }
    }
```

当 `RecordBatch` 成功收到正常响应、或超市、或关闭生成者时，都会调用 `RecordBatch` 的 `done()` 方法。在 `done()` 方法中，会回调 `RecordBatch` 中全部消息的  `Callback` 回调，并调用其 `produceFuture` 字段的 `done()` 方法。`RecordBatch.done()` 方法的调用关系如下：

![RecordBatch.done()调用链](https://github.com/BingLau7/blog/blob/master/images/blog_43/RecordBatch-done.jpg?raw=true)

其代码如下

```java
    public void done(long baseOffset, long timestamp, RuntimeException exception) {
        log.trace("Produced messages to topic-partition {} with base offset offset {} and error: {}.",
                  topicPartition,
                  baseOffset,
                  exception);
        // execute callbacks
        for (int i = 0; i < this.thunks.size(); i++) {
            try {
                Thunk thunk = this.thunks.get(i);
                if (exception == null) { /* 正常处理完成 */
                    // If the timestamp returned by server is NoTimestamp, that means CreateTime is used. Otherwise LogAppendTime is used.
                    /* 将服务端返回的信息（offset 和 timestamp）和消息的其他信息封装成 RecordMetadata */
                    RecordMetadata metadata = new RecordMetadata(this.topicPartition,  baseOffset, thunk.future.relativeOffset(),
                                                                 timestamp == Record.NO_TIMESTAMP ? thunk.future.timestamp() : timestamp,
                                                                 thunk.future.checksum(),
                                                                 thunk.future.serializedKeySize(),
                                                                 thunk.future.serializedValueSize());
                    /* 调用详细对应的自定义 Callback */
                    thunk.callback.onCompletion(metadata, null);
                } else { /* 处理过程中出现异常，注意，第一个参数为 null，与上面情况刚好相反 */
                    thunk.callback.onCompletion(null, exception);
                }
            } catch (Exception e) {
                log.error("Error executing user-provided callback on message for topic-partition {}:", topicPartition, e);
            }
        }
        /* 标识整个 RecordBatch 都已经处理完成 */
        this.produceFuture.done(topicPartition, baseOffset, exception);
    }
```

## BufferPool

`ByteBuffer` 的创建和释放是比较消耗资源的，为了实现内存的高效利用，基本上每个成熟的框架或工具都有一套内存管理机制。Kafka 客户端使用 `BufferPool` 来实现 `ByteBuffer` 的复用。

首先需要了解的是，每个 `BufferPool` 对象只针对特定大小（由`poolableSize` 字段指定）的 `ByteBuffer` 进行管理，对于其他大小的 `ByteBuffer` 并不会缓存进 `BufferPool` 。一般情况，我们会调整 `MemoryRecords` 的大小（`RecordAccumulator.batchSize` 字段指定），使得每个 `MemoryRecords` 可以缓存多条消息。但也有例外情况，当一条消息的字节数大于 `MemoryRecords` 时，就不会复用`BufferPool` 中缓存的 `ByteBuffer` ，而是例外分配 `ByteBuffer`，在它被使用完后也不会放入 `BufferPool` 进行管理，而是直接丢弃由 GC 回收。

下面介绍一下 `BufferPoll` 的关键字段

```java
    /* 记录了整个 Pool 的大小 */
    private final long totalMemory;
    /* 因为有多线程并发分配和回收 ByteBuffer，所以使用锁控制并发，保证线程安全 */
    private final ReentrantLock lock;
    /* 是一个 ArrayDeque<ByteBuffer> 队列，其中缓存了指定大小的 ByteBuffer 对象 */
    private final Deque<ByteBuffer> free;
    /* 记录因申请不到足够空间而阻塞的线程，此队列中实际记录的是阻塞线程对应的 Condition 对象 */
    private final Deque<Condition> waiters;
    /* 记录了可用的空间大小，这个空间是 totalMemory - free 列表的 ByteBuffer 大小 */
    private long availableMemory;
```

`BufferPool.allocate()` 方法负责从缓冲池中申请 `ByteBuffer` ，**当缓冲池中空间不足时，就会阻塞调用线程**。

```java
    public ByteBuffer allocate(int size, long maxTimeToBlockMs) throws InterruptedException {
        if (size > this.totalMemory)
            throw new IllegalArgumentException("Attempt to allocate " + size
                                               + " bytes, but there is a hard limit of "
                                               + this.totalMemory
                                               + " on memory allocations.");

        this.lock.lock(); /* 同步加锁 */
        try {
            // check if we have a free buffer of the right size pooled
            /* 请求的是 poolableSize 指定大小的 ByteBuffer，且 free 中有空闲的 ByteBuffer */
            if (size == poolableSize && !this.free.isEmpty())
                return this.free.pollFirst(); /* 返回合适的 ByteBuffer */

            /* 当申请的空间大小不是 poolableSize，则执行下面的操作 */

            // now check if the request is immediately satisfiable with the
            // memory on hand or if we need to block
            /* free 队列中都是 poolableSize 大小的 ByteBuffer，可以直接计算整个 free 队列的空间 */
            int freeListSize = this.free.size() * this.poolableSize;
            if (this.availableMemory + freeListSize >= size) {
                // we have enough unallocated or pooled memory to immediately
                // satisfy the request
                /* 为了让 availableMemory > size，freeUp() 方法会从 free 队列中不断释放
                 * ByteBuffer，直到 availableMemory 满足这次申请 */
                freeUp(size);
                this.availableMemory -= size; /* 减少 availableMemory */
                lock.unlock(); /* 解锁 */
                /* 这里没有用 free 队列中的 buffer，而是直接分配 size 大小的 HeapByteBuffer */
                return ByteBuffer.allocate(size);
            } else { /* 没有足够空间，只能阻塞了 */
                // we are out of memory and will have to block
                int accumulated = 0;
                ByteBuffer buffer = null;
                Condition moreMemory = this.lock.newCondition();
                long remainingTimeToBlockNs = TimeUnit.MILLISECONDS.toNanos(maxTimeToBlockMs);
                /* 将 Condition 添加到 waiters 中 */
                this.waiters.addLast(moreMemory);
                // loop over and over until we have a buffer or have reserved
                // enough memory to allocate one
                while (accumulated < size) { /* 循环等待 */
                    long startWaitNs = time.nanoseconds();
                    long timeNs;
                    boolean waitingTimeElapsed;
                    try {
                        /* 阻塞 */
                        waitingTimeElapsed = !moreMemory.await(remainingTimeToBlockNs, TimeUnit.NANOSECONDS);
                    } catch (InterruptedException e) {
                        /* 异常，移除此线程对应的 Condition */
                        this.waiters.remove(moreMemory);
                        throw e;
                    } finally {
                        /* 统计阻塞时间 */
                        long endWaitNs = time.nanoseconds();
                        timeNs = Math.max(0L, endWaitNs - startWaitNs);
                        this.waitTime.record(timeNs, time.milliseconds());
                    }

                    if (waitingTimeElapsed) { /* 超时，报错 */
                        this.waiters.remove(moreMemory);
                        throw new TimeoutException("Failed to allocate memory within the configured max blocking time " + maxTimeToBlockMs + " ms.");
                    }

                    remainingTimeToBlockNs -= timeNs;
                    // check if we can satisfy this request from the free list,
                    // otherwise allocate memory
                    /* 请求的是 poolableSize 大小的 ByteBuffer，且 free 中有空间的 ByteBuffer */
                    if (accumulated == 0 && size == this.poolableSize && !this.free.isEmpty()) {
                        // just grab a buffer from the free list
                        buffer = this.free.pollFirst();
                        accumulated = size;
                    } else { /* 先分配一部分空间，并继续等待空闲空间 */
                        // we'll need to allocate memory, but we may only get
                        // part of what we need on this iteration
                        freeUp(size - accumulated);
                        int got = (int) Math.min(size - accumulated, this.availableMemory);
                        this.availableMemory -= got;
                        accumulated += got;
                    }
                }

                // remove the condition for this thread to let the next thread
                // in line start getting memory
                /* 已经成功分配空间，移除 Condition */
                Condition removed = this.waiters.removeFirst();
                if (removed != moreMemory)
                    throw new IllegalStateException("Wrong condition: this shouldn't happen.");

                // signal any additional waiters if there is more memory left
                // over for them
                /* 还是要用空闲空间，就唤醒下一个线程 */
                if (this.availableMemory > 0 || !this.free.isEmpty()) {
                    if (!this.waiters.isEmpty())
                        this.waiters.peekFirst().signal();
                }

                // unlock and return the buffer
                lock.unlock(); /* 解锁 */
                if (buffer == null)
                    return ByteBuffer.allocate(size);
                else
                    return buffer;
            }
        } finally { /* 解锁 */
            if (lock.isHeldByCurrentThread())
                lock.unlock();
        }
    }
```

了解了 `allocate()` 方法的实现后，继续分享 `deallocate()` 方法的实现

```java
    public void deallocate(ByteBuffer buffer, int size) {
        lock.lock(); /* 加锁 */
        try {
            /* 释放的 ByteBuffer 的大小是 poolableSize，放入 free 队列进行管理 */
            if (size == this.poolableSize && size == buffer.capacity()) {
                buffer.clear();
                this.free.add(buffer);
            } else {
                /* 释放的 ByteBuffer 大小不是 poolableSize，不会复用 ByteBuffer，仅修改 availableMemory 的值 */
                this.availableMemory += size;
            }
            /* 唤醒一个因空间不足而阻塞的线程 */
            Condition moreMem = this.waiters.peekFirst();
            if (moreMem != null)
                moreMem.signal();
        } finally {
            lock.unlock(); /* 解锁 */
        }
    }
```

## RecordAccumulator

介绍完了 `MemoryRecord`、`RecordBatch` 以及 `BufferPool` 的工作机制，再来看 `RecordAccumulator` 的实现就比较简单了。先来看看 `RecordAccumulator` 中的关键字段

```java
    /* 指定每个 RecordBatch 底层 ByteBuffer 的大小 */
    private final int batchSize;
    /* 压缩类型 */
    private final CompressionType compression;
    /* BufferPool 对象 */
    private final BufferPool free;
    /* TopicPartition 与 RecordBatch 集合的映射关系，类型是 CopyOnWriteMap，是线程安全的结合，
     * 但其中的 Deque 是 ArrayDeque 类型，是非线程安全的结合。追加新消息或发送 RecordBatch
     * 的时候，需要同步加锁。每个 Deque 都保持了发往对应 TopicPartition 的 RecordBatch 集合*/
    private final ConcurrentMap<TopicPartition, Deque<RecordBatch>> batches;
    /* 未发送完成 RecordBatch 集合，底层通过 Set<RecordBatch> 实现 */
    private final IncompleteRecordBatches incomplete;
    /* 使用 drain 方法批量导出 RecordBatch 时，为了防止饥饿，使用 drainIndex 记录上次发送停止时的位置，
     * 下次继续从此位置开始发送 */
    private int drainIndex;
```

`KafkaProducer.send()` 方法最终会调用 `RecordsAccumulator.append()` 方法将消息追加到 `RecordAccumulator` 中

```java
    public RecordAppendResult append(TopicPartition tp,
                                     long timestamp,
                                     byte[] key,
                                     byte[] value,
                                     Callback callback,
                                     long maxTimeToBlock) throws InterruptedException {
        // We keep track of the number of appending thread to make sure we do not miss batches in
        // abortIncompleteBatches().
        /* 统计正在向 RecordsAccumulator 中追加数据的线程数 */
        appendsInProgress.incrementAndGet();
        try {
            /* 1. 查找 TopicPartition 对应的 Deque */
            // check if we have an in-progress batch
            Deque<RecordBatch> dq = getOrCreateDeque(tp);
            synchronized (dq) { /* 2. 对 Deque 对象加锁 */
                /* 边界检查 */
                if (closed)
                    throw new IllegalStateException("Cannot send after the producer is closed.");
                /* 3. 向 Deque 中最后一个 RecordBatch 追加 Record */
                RecordAppendResult appendResult = tryAppend(timestamp, key, value, callback, dq);
                if (appendResult != null)
                    return appendResult; /* 5. 追加成功直接返回 */
            } /* 4. synchronized 块结束，自动解锁 */

            // we don't have an in-progress record batch try to allocate a new batch
            /* 6. 追加失败，从 BufferPool 中申请新空间 */
            int size = Math.max(this.batchSize, Records.LOG_OVERHEAD + Record.recordSize(key, value));
            log.trace("Allocating a new {} byte message buffer for topic {} partition {}", size, tp.topic(), tp.partition());
            ByteBuffer buffer = free.allocate(size, maxTimeToBlock);
            synchronized (dq) {
                // Need to check if producer is closed again after grabbing the dequeue lock.
                /* 边界检查 */
                if (closed)
                    throw new IllegalStateException("Cannot send after the producer is closed.");

                /* 7. 对 Deque 加锁后，再次调用 tryAppend() 方法尝试追加 Record */
                RecordAppendResult appendResult = tryAppend(timestamp, key, value, callback, dq);
                if (appendResult != null) { /* 8. 追加成功，则返回 */
                    // Somebody else found us a batch, return the one we waited for! Hopefully this doesn't happen often...
                    free.deallocate(buffer); /* 释放 7 申请的新空间 */
                    return appendResult;
                }
                MemoryRecords records = MemoryRecords.emptyRecords(buffer, compression, this.batchSize);
                RecordBatch batch = new RecordBatch(tp, records, time.milliseconds());
                /* 9. 在新创建的 RecordBatch 中追加 Record，并将其添加到 batches 集合中 */
                FutureRecordMetadata future = Utils.notNull(batch.tryAppend(timestamp, key, value, callback, time.milliseconds()));

                dq.addLast(batch);
                /* 10. 将新建的 RecordBatch 追加到 incomplete 集合 */
                incomplete.add(batch);
                /* 12. 返回 RecordAppendResult */
                return new RecordAppendResult(future, dq.size() > 1 || batch.records.isFull(), true);
            } /* 11. synchronized 块结束，解锁 */
        } finally {
            appendsInProgress.decrementAndGet();
        }
    }
```

上面之所以分为两个 `synchronized` 是因为向 `BufferPool` 申请新 `ByteBuffer` 的时候，可能会导致阻塞。我们假设在一个 `synchronized` 中完成上面所有的追加操作，有下面的场景：线程 1 发送的消息比较大，需要向 `BufferPool` 申请新空间，而此时 `BufferPool` 空间不足，线程 1 在 `BufferPool` 上等待，此时它依然持有对应 `Deque` 的锁；线程 2 发送的消息较小，`Deque` 最后一个 `RecordBatch` 剩余空间够用，但是由于线程 1 未释放 `Deque` 的锁，所以也需要一起等待。若线程 2 这样的线程比较多，就会造成很多不必要的线程阻塞，降低了吞吐量。

第二次加锁后重试，是为了防止多个线程并发向 `BufferPool` 申请空间后，造成内部碎片。

现在回到 `KafkaProducer.doSend()` 方法，`doSend()` 方法的最后一步就是判断此次向 `RecordAccumulator` 中追加消息后是否满足唤醒 `Sender` 线程条件，这里唤醒 `Sender` 线程的条件是消息所在队列的最后一个 `RecordBatch` 满了或此队列中不止一个 `RecordBatch`。

在客户端将消息发送给服务端之前，会调用 `RecordAccumulator.ready()` 方法获取集群中符合发送消息条件的节点集合。这些条件是站在 `RecordAccumulator` 的家都对集群中的 `Node` 进行筛选，具体条件如下：

1. `Deque` 中有多个 `RecordBatch` 或是第一个 `RecordBatch` 是否满了
2. 是否超时了
3. 是否有其他线程在等待 `BufferPool` 释放空间（即 `BufferPool` 的空间耗尽了）
4. 是否有线程正在等待 `flush` 操作完成
5. `Sender` 线程准备关闭

下面来看一下 `ready` 的代码，它会遍历 `batches` 集合中每个分区，首先查找当前分区 `Leader` 副本所在的 `Node` ，如果满足上述五个条件，则将此 `Node` 信息记录到 `readyNodes` 集合中。遍历完成后返回 `ReadyCheckResult` 对象，其中记录了满足发送条件的 `Node` 集合、在遍历过程中是否找不到 `Leader` 副本的分区（也可以认为是 `Metadata` 中当前的元数据过时了）、下次调用 `ready()` 方法进行检查的时间间隔。

```java
    public ReadyCheckResult ready(Cluster cluster, long nowMs) {
        Set<Node> readyNodes = new HashSet<>(); /* 用来记录可以向那些 Node 节点发送数据 */
        long nextReadyCheckDelayMs = Long.MAX_VALUE; /* 记录下次需要调用 ready() 方法的时间间隔 */
         /* 根据 Metadata 元数据中记录有找不到 Leader 副本的分区 */
        Set<String> unknownLeaderTopics = new HashSet<>();

        /* 是否有现车在阻塞等待 BufferPool 释放空间 */
        boolean exhausted = this.free.queued() > 0;
        /* 下面遍历 batches 集合，对其中每个分区的 Leader 副本所在的 Node 都进行判断 */
        for (Map.Entry<TopicPartition, Deque<RecordBatch>> entry : this.batches.entrySet()) {
            TopicPartition part = entry.getKey();
            Deque<RecordBatch> deque = entry.getValue();

            Node leader = cluster.leaderFor(part); /* 查找分区的 Leader 副本所在的 Node */
            synchronized (deque) { /* 加锁读取 deque 的元素 */
                /* 根据 Cluster 的信息检查 Leader，Leader 找不到，肯定不能发送消息 */
                if (leader == null && !deque.isEmpty()) {
                    // This is a partition for which leader is not known, but messages are available to send.
                    // Note that entries are currently not removed from batches when deque is empty.
                    /* 这里不为空之后会触发 Metadata 的更新 */
                    unknownLeaderTopics.add(part.topic());
                } else if (!readyNodes.contains(leader) && !muted.contains(part)) {
                    /* 只取 Deque 中的第一个 RecordBatch */
                    RecordBatch batch = deque.peekFirst();
                    if (batch != null) {
                        boolean backingOff = batch.attempts > 0 && batch.lastAttemptMs + retryBackoffMs > nowMs;
                        long waitedTimeMs = nowMs - batch.lastAttemptMs;
                        long timeToWaitMs = backingOff ? retryBackoffMs : lingerMs;
                        long timeLeftMs = Math.max(timeToWaitMs - waitedTimeMs, 0);
                        /* Deque 中有多个 RecordBatch 或是第一个 RecordBatch 是否满了 */
                        boolean full = deque.size() > 1 || batch.records.isFull();
                        /* 是否超时了 */
                        boolean expired = waitedTimeMs >= timeToWaitMs;
                        boolean sendable = full || expired
                                || exhausted /* 是否有其他线程在等待 BufferPool 释放空间（即 BufferPool 的空间耗尽了）*/
                                || closed  /* Sender 线程准备关闭 */
                                || flushInProgress(); /* 是否有线程正在等待 flush 操作完成 */
                        if (sendable && !backingOff) {
                            readyNodes.add(leader);
                        } else {
                            // Note that this results in a conservative estimate since an un-sendable partition may have
                            // a leader that will later be found to have sendable data. However, this is good enough
                            // since we'll just wake up and then sleep again for the remaining time.
                            /* 记录下次需要调用 ready() 方法检查的时间间隔 */
                            nextReadyCheckDelayMs = Math.min(timeLeftMs, nextReadyCheckDelayMs);
                        }
                    }
                }
            }
        }

        return new ReadyCheckResult(readyNodes, nextReadyCheckDelayMs, unknownLeaderTopics);
    }
```

调用 `RecordAccumulator.ready()` 方法得到 `readyNodes` 集合后，此集合还要经过 `NetworkClient` 的过滤之后，才能得到最终能够发送消息的 `Node` 集合。

`RecordAccumulator.drain()` 方法会根据上述 `Node` 集合获取要发送的消息，返回 `Map<Integer, List<RecordBatch>>` 集合，`key` 是 `NodeId`，`value` 是待发送的 `RecordBatch` 集合。`drain` 方法也是由 `Sender` 线程调用的。

`drain` 方法的核心逻辑是进行映射的转换：将 `RecordAccumulatro` 记录的 `TopicPartition -> RecordBatch` 集合的映射，转换成了 `NodeId -> RecordBatch` 集合的映射。

为什么需要这次转换？在网络 I/O 层面，生产者是面向 `Node` 节点发送消息数据，它只建立到 `Node` 的连接并发送数据，并不关心这些数据数据哪个 `TopicPartition` ；而是调用 `KafkaProducer` 的上层业务逻辑中，则是按照 `TopicPartition` 的 `Node` 节点上。在之后介绍 `Sender` 线程的时候会发现，它每次向每个 `Node` 节点至多发送一个 `ClientRequest` 请求，其中封装了追加到此 `Node` 节点上多个分区的消息，待请求到达服务端后，由 Kafka 对请求进行解析。

下面来看看 `drain` 方法的代码

```java
    public Map<Integer, List<RecordBatch>> drain(Cluster cluster,
                                                 Set<Node> nodes,
                                                 int maxSize,
                                                 long now) {
        if (nodes.isEmpty())
            return Collections.emptyMap();

        /* 转换之后的结果 */
        Map<Integer, List<RecordBatch>> batches = new HashMap<>();
        for (Node node : nodes) { /* 遍历指定的 ready Node 集合 */
            int size = 0;
            /* 获取当前 Node 上的分区集合 */
            List<PartitionInfo> parts = cluster.partitionsForNode(node.id());
            /* 记录要发送的 RecordBatch */
            List<RecordBatch> ready = new ArrayList<>();
            /* to make starvation  less likely this loop doesn't start at 0 */
            /* drainIndex 是 batches 的下标，记录上次发送停止时的位置，下次继续从此位置开始发送
             * 若一直从索引 0 的队列开始发送，可能会出现一直发送前几个分区的消息的情况，造成其他分区饥饿 */
            int start = drainIndex = drainIndex % parts.size();
            do {
                PartitionInfo part = parts.get(drainIndex); /* 获取分区详细情况 */
                TopicPartition tp = new TopicPartition(part.topic(), part.partition());
                // Only proceed if the partition has no in-flight batches.
                if (!muted.contains(tp)) {
                    /* 获取对应的 RecordBatch 队列 */
                    Deque<RecordBatch> deque = getDeque(new TopicPartition(part.topic(), part.partition()));
                    /* 边界检查 */
                    if (deque != null) {
                        synchronized (deque) {
                            RecordBatch first = deque.peekFirst();
                            if (first != null) {
                                boolean backoff = first.attempts > 0 && first.lastAttemptMs + retryBackoffMs > now;
                                // Only drain the batch if it is not during backoff period.
                                if (!backoff) {
                                    if (size + first.records.sizeInBytes() > maxSize && !ready.isEmpty()) {
                                        // there is a rare case that a single batch size is larger than the request size due
                                        // to compression; in this case we will still eventually send this batch in a single
                                        // request
                                        break; /* 队列已满，结束循环，一般是一个请求的大小 */
                                    } else {
                                        /* 从队列中获取一个 RecordBatch，并将这个 RecordBatch 放到 ready 集合中 */
                                        RecordBatch batch = deque.pollFirst();
                                        /* 关闭 Compressor 及底层输出流，并将 MemoryRecords 设置为只读 */
                                        batch.records.close();
                                        size += batch.records.sizeInBytes();
                                        ready.add(batch);
                                        batch.drainedMs = now;
                                    }
                                }
                            }
                        }
                    }
                }
                /* 更新 drainIndex */
                this.drainIndex = (this.drainIndex + 1) % parts.size();
            } while (start != drainIndex);
            batches.put(node.id(), ready);
        }
        return batches;
    }
```

注意，在上面代码中，只从每个队列中取出一个 `RecordBatch` 放到 `ready` 集合中，这也是为了防止饥饿，提高系统的可用性。

`RecordAccumulator` 的工作原理到这里就介绍完了，整个 `KafkaProducer.send()` 方法过程中用到的所有组件也都分析完了。后面将分析的是 `Sender` 线程是如何发送消息的。
