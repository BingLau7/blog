---
title: Kafka 的攒批机制
date: 2022-02-23 19:50:39
tags:
    - Kafka
---

## 介绍
本文的起因是在工作中到 Kafka 的使用**最广泛、性能要求最高、问题最多的组件**。而其中性能问题的**调优思路往往与 Kafka 的攒批息息相关**。所以这篇文章中会一起学习 Kafka 攒批原理及如何使用 Kafka 的攒批。

<!-- more -->

### 什么是攒批？
发送消息（Record）时，让消息在**内存中**驻留一段时间，等待多个消息同时达到可发送状态时，形成一次 Request 发送到 Server 中。
### 为什么需要攒批？

- 默认网络发送成本大于内存中流转成本。使用攒批可以极大提升吞吐量。
- 对数据压缩有优势。增加吞吐量及减少存储成本。
#### 攒批一定好吗？

- 攒批时间太长会影响时效性。
- 由于消息是在内存中驻留的，会增加内存占用，可能影响 GC 反而减少了吞吐。
## 相关配置及方法原理
### 相关 API 描述
在 [Producer](https://kafka.apache.org/34/javadoc/org/apache/kafka/clients/producer/KafkaProducer.html) 的 API 中明确写到，Producer 是由 buffer space 与 background IO 组成的，而其中 buffer space 保存了尚未传输到 server 的 records，而这些 records 是有 `send`异步方法被调用时候传入到缓冲区中的。
`buffer.memory` 控制生产者可用于缓冲的内存总量。 如果记录的发送速度快于它们可以传输到服务器的速度，则此缓冲区空间将被耗尽。 当缓冲区空间耗尽时，额外的发送调用将被阻塞。 阻塞时间的阈值由 max.block.ms 确定，超过它会抛出 TimeoutException。
### Record 如何到 Server 中的？
![Kafka.png](/images/kafka-batch/image_1.png)
Kafka 在 ProducerClient 中发送消息主要是由 `RecordAccumulate` 作为中转站，由主线程通过 `append` 方法根据 partition 不同追加消息进入 `ProduceBatch` 中，`Sender`线程每次轮询将准备好（`ready`方法）的消息通过 `drain()` 方法获取到，然后根据 Node 不同组成 `ProduceRequest` 发送给 KafkaServer。

1. 应用程序发送的消息通过拦截器和序列化得到消息的各个部分（`Header`、`key`、`value`...）
2. 消息分区之后通过 `RecordAccumulate#append`将消息放入 `ProducerBatch` 中，底层存储使用了 `BufferPool` 分配的 `ByteBuffer`，这里涉及到 Kafka 的内存控制，后续有机会介绍（坑1）
3. `Sender`通过 `ready` 方法询问是否有准备好发送的消息，如果有的话返回其 Node 信息
4. `Sender`通过 `drain` 方法从 `RecordAccumulate`中获取在 `max.request.size`配置下允许的所有 `ProducerBatch`，返回结果以 `Node`-> `List<ProducerBatch>`作为映射。
5. 每个 Node 的 `List<ProducerBatch>`组装成 `ProduceRequest`
6. 通过 `NetworkClient`发送给 `Server`
### 有什么方式能控制攒批？

- `RecordAccumulator`是属于 producer.internal 的类，主要就是控制攒批。

类描述如下
```
/**
 * This class acts as a queue that accumulates records into {@link MemoryRecords}
 * instances to be sent to the server.
 * <p>
 * The accumulator uses a bounded amount of memory and append calls will block when that memory is exhausted, unless
 * this behavior is explicitly disabled.
 */
```
其中主要方法是
#### `append`：追加消息
会被 `Producer.doSend`调用，并返回一个携带有 `FutureRecordMeta`的返回结果；
主要实现：

1. 给每个 `partition` 分配一个 `Deque<ProducerBatch>`用于缓存消息
2. 如果 `Deque`非空则会最终调用 `MemoryRecordsBuilder.tryAppend`方法将 `Record` 累加到 `ProducerBatch`中
3. 如果 `Deque`为空，则开始重新生成 `ProducerBatch`(主要是生成其中的 `MemoryRecordsBuilder`再 `tryAppend`

即是说，Record 被 `ProducerBatch`保存了起来，等待发送。
#### `ready()` 之后 `drain`
`Sender.sendProducerData` 是 `Sender`的核心方法在线程 `run` 的时候被持续调用，而在其方法的开头就调用了 `RecordAccumulator.ReadyCheckResult result = this.accumulator.ready(cluster, now);`方法，核心就是在攒批准备好之后发送。我们先看看 `ready`的方法签名。
```json
/**
 * Get a list of nodes whose partitions are ready to be sent, and the earliest time at which any non-sendable
 * partition will be ready; Also return the flag for whether there are any unknown leaders for the accumulated
 * partition batches.
   获取准备发送分区的节点列表，以及任何不可发送分区准备就绪的最早时间；
   还返回用于累积分区批次是否有任何未知领导者的标志。
 * </p>
 * A destination node is ready to send data if:
   如果一个目的端 node 已经准备好了可以发送数据，则会有	
 * <ol>
 * <li>There is at least one partition that is not backing off its send
   至少有一个分区没有停止发送
 * <li><b>and</b> those partitions are not muted (to prevent reordering if
 *   {@value org.apache.kafka.clients.producer.ProducerConfig#MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION}
 *   is set to one)</li>
   并且这些分区没有被锁（以防止在 MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION 被设置为 1 的时候重排序）
 * <li><b>and <i>any</i></b> of the following are true</li>
   并且一下任何条件达到
 * <ul>
 *     <li>The record set is full</li>
       RecordSet 满了
 *     <li>The record set has sat in the accumulator for at least lingerMs milliseconds</li>
       RecrodSet 已经缓存了超过 lingerMs 时间
 *     <li>The accumulator is out of memory and threads are blocking waiting for data (in this case all partitions
 *     are immediately considered ready).</li>
       Accumulator 内存不或者线程正在阻塞等待数据（这种情况下，所有分区都立即被视为已就绪）
 *     <li>The accumulator has been closed</li>
       Accumulator 已经关闭
 * </ul>
 * </ol>
 */
```
接下来我们深入 `ready` 方法内部探索，由于我们的目的是分析原理，流程只会涉及到主要代码
首先 ready 会针对每个 `partition` 的 `Dequq` 进行一次循环来对每个分区数据进行检测
```java
public ReadyCheckResult ready(Cluster cluster, long nowMs) {
    Set<Node> readyNodes = new HashSet<>();
    long nextReadyCheckDelayMs = Long.MAX_VALUE;
    Set<String> unknownLeaderTopics = new HashSet<>();

    // free 表示是否内存(BufferPool)是否还有剩余，这也是 batch.size 配置生效的地方
    boolean exhausted = this.free.queued() > 0;
    // 循环遍历每个 partition 的数据，this.batches 使用了自实现的 CopyOnWriteMap 用于读多写少的并发场景
    for (Map.Entry<TopicPartition, Deque<ProducerBatch>> entry : this.batches.entrySet()) {
        Deque<ProducerBatch> deque = entry.getValue();
        synchronized (deque) {
            // 这里如果存在大量 paritition 时会成为热点，首先检查 batch 是否存在可以避免后续检查的成本
            ProducerBatch batch = deque.peekFirst();
            if (batch != null) {
                // 获取 partition 的 leader 节点，这里如果节点不存在会其放入结果中，这里不展示
                TopicPartition part = entry.getKey();
                Node leader = cluster.leaderFor(part);
                // readyNodes 是避免重复判断，isMuted 的机制与 max.in.flight.requests.per.connection 保序相关，后续详聊，这里简单理解就是不发送该 partition 数据
            	if (!readyNodes.contains(leader) && !isMuted(part)) {
                    // waitedTimeMs: batch 等待时间
                    // backingOff：是否不需要重试(等待时间小于重试时间间隔)
                    // timeToWaitMs：最终等待发送时间，如果是重试就 retryBackoffMs，否则就 lingerMs
                    long waitedTimeMs = batch.waitedTimeMs(nowMs);
                    boolean backingOff = batch.attempts() > 0 && waitedTimeMs < retryBackoffMs;
                    long timeToWaitMs = backingOff ? retryBackoffMs : lingerMs;
                    // deque.size() > 1 表示最少有一个 ProducerBatcher 是满了
                    // batch.isFull() 的判断逻辑就是 pool 被关了或者是大于之前设置的预设值(batch.size)
                    // 关于 batch 的 size 值设定是在 append 创建 ProducerBatch 的时候通过 Accumulate 中 batchSize 创建得到的
                    // 实际就是 BufferPool 中的 limit
                    boolean full = deque.size() > 1 || batch.isFull();
                    // 是否等待过了
                    boolean expired = waitedTimeMs >= timeToWaitMs;
                    // 事务相关的忽略了...
                    boolean sendable = full   // 超过 BufferPool 的 limit
                        || expired            // 超过 lingerMs 或需要重试
                        || exhausted          // 超过 batch.size
                        || closed             // Accumulate 关闭
                        || flushInProgress(); // 被调用了 flash
                    if (sendable && !backingOff) {
                        readyNodes.add(leader);
                    } else {
                        long timeLeftMs = Math.max(timeToWaitMs - waitedTimeMs, 0);
                        // Note that this results in a conservative estimate since an un-sendable partition may have
                        // a leader that will later be found to have sendable data. However, this is good enough
                        // since we'll just wake up and then sleep again for the remaining time.
                        nextReadyCheckDelayMs = Math.min(timeLeftMs, nextReadyCheckDelayMs);
                    }
                }
            }
        }
    }
    return new ReadyCheckResult(readyNodes, nextReadyCheckDelayMs, unknownLeaderTopics);
}

```
在 `Sender`中通过 `ready` 得到 readyNodes 之后，调用 `drain`返回 `node.id`与 `List<ProducerBatch>` 的 Map。**这样间接说明，Producer 会将发送往一个 Node 的数据 Merge 到一次请求中，这里在 **`**Sender#sendProduceReqest**`**中可以清晰的看到，所以说，测试性能得用多节点，单节点 partition 数量不是真实场景**。
`drain`方法主要是针对每个 ready node 调用 `drainBatchesForOneNode` 方法然后汇聚成 `Map<Integer, List<ProducerBatch>>` 返回给 `Sender`
方法签名:
```java
/**
 * Drain all the data for the given nodes and collate them into a list of batches that will fit within the specified
 * size on a per-node basis. This method attempts to avoid choosing the same topic-node over and over.
 *
	drain 给定节点(nodes) 的所有数据并将其整理成每个节点批次的指定大小（maxSize），返回列表信息，避免选择相同的节点
 * @param cluster The current cluster metadata
 * @param nodes The list of node to drain
 * @param maxSize The maximum number of bytes to drain
 * @param now The current unix time in milliseconds
 * @return A list of {@link ProducerBatch} for each node specified with total size less than the requested maxSize.
 */
```
```java
private List<ProducerBatch> drainBatchesForOneNode(Cluster cluster, Node node, int maxSize, long now) {
    int size = 0;
    List<PartitionInfo> parts = cluster.partitionsForNode(node.id());
    List<ProducerBatch> ready = new ArrayList<>();
    // drainIndex 是从之前轮询中保留下来的，其递增逻辑变更是 (drainIndex + 1) % parts.size()
    // 主要是为了在循环中的 break 条件触发后从 break 的节点开始轮询，减少空轮询
    // 再次 mod parts.size 是为了防止 partition 数量变更
    int start = drainIndex = drainIndex % parts.size();
    do {
        PartitionInfo part = parts.get(drainIndex);
        TopicPartition tp = new TopicPartition(part.topic(), part.partition());
        this.drainIndex = (this.drainIndex + 1) % parts.size();

        // 同步块，这里前面省略了对于 patition 及 deque 不处理情况的判断
        // 忽略 deque 为空及重试的场景代码
        synchronized (deque) {
            // 当前批次不为空且 size 值将要超过 maxSize 时视为结束，可以发送了
            if (size + first.estimatedSizeInBytes() > maxSize && !ready.isEmpty()) {
                break;
            } else {
                // 针对 transaction 消息的设置，正常发送消息不会涉及，这里忽略
                if (shouldStopDrainBatchesForPartition(first, tp))
                    break;

                // 忽略事务相关内容，后续单独说明...（坑2）
                // 本次 batch 不允许再 append，将其放入 ready 中，设置 drain 时间
                batch.close();
                size += batch.records().sizeInBytes();
                ready.add(batch);

                batch.drained(now);
            }
        }
    } while (start != drainIndex);
    return ready;
}
```
## 场景测试
在 Kafka2.x 的版本中体会以下几个版本 batch 上的差异。

1. 使用 Kafka 自带的压测脚本
2. 4K 消息
3. 单节点（测试存在偏差性，条件有限，后面补充多节点测试）
### `batch.size` 影响
JVM 内存 512M
lingerMs 固定 10 ms
max.request.size 固定 16M
buffer.memory  固定 16M
max.in.flight.requests.per.connection = 1
命令
```java
./bin/kafka-producer-perf-test.sh --topic test \
--num-records 1000000 \
--record-size 4000 \
--throughput -1 \
--producer-props bootstrap.servers=11.162.217.15:9093 compression.type=lz4 linger.ms=10 max.request.size=16777216 buffer.memory=16777216 batch.size=100 max.in.flight.requests.per.connection=1
```
| batch.size | 数据 |
| --- | --- |
| 100 | 17.20 MB/sec |
| 16384 (16K) | 281.92 MB/sec |
| 131072 (128K) | 984.49 MB/sec |
| 1048576 (1M) | 1098.23 MB/sec |
| 16777216 (16M) | 841.21 MB/sec |

### `linger.ms`影响
JVM 内存 512M
batch.size 固定 16384 ms
max.request.size 固定 16M
buffer.memory  固定 16M
max.in.flight.requests.per.connection  = 1

测试下来在压测场景下影响不大，因为 `batch.size` 的条件会优先被满足。
## 同类产品的原理对比
### RocketMQ
`RocketMQ`也同样有消息攒批逻辑，参数相差不大：

- `batchSize`：表示消息批的大小，单位是字节。当达到批大小后，RocketMQ 会将消息发送到 Broker。
- `maxDelayTime`：表示最大的等待时间，单位是毫秒。当等待时间超过该时间后，RocketMQ 会将消息发送到 Broker。
### Pulsar
`Pulsar`参数差距同样不大：

- `batchingEnabled`：表示是否启用消息批处理，默认为 false。
- `batchingMaxMessages`：表示消息批的最大数量，默认为 1。
- `batchingMaxPublishDelayMicros`：表示最大的等待时间，单位是微秒。当等待时间超过该时间后，Pulsar 会将消息发送到 Broker。
