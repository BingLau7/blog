---
title: Flink-Source-1 Connector 的一些概念理解
date: 2024-08-18 17:29:27
tags: 
    - Flink
---

本篇文章主要目的是在工作和学习过中对于 Flink Source 的一些使用和学习的理解记录。

<!-- more -->

## 主要的概念介绍
### 关系介绍
![](/images/flink/flink-source-1.svg)

### API

| 类名 | 用途 | 示例 |
| --- | --- | --- |
| `Source<T, Split, EnumChk>` | Flink 的 Source 工具类 <br> 1. 负载创建 SplitEnumerator 与 SourceReader 这两个 Source 关键类; <br>  2. 提供了创建 Split 与 EnumChk 的序列化和反序列化器的接口; <br>  3. 提供是不是有界的标识 | KafkaSource 的实现<br>  1. 创建 KafkaSourceEnumerator 与 KafkaSourceReader <br>  2. 提供 KafkaPartitionSplitSerializer 与 KafkaSourceEnumStateSerializer <br> 3. 在创建 KafkaSource 确定是否是有界的(配置项) |
| `SplitEnumerator<Split, EnumChk>` | Flink 为 Source 区分了切片（Split）和根据切片读取数据的两个主要组件。SplitEnumerator 就是生产切片的生产者。<br>  1. 负责生产切片，以及分配切片到 Reader 中 <br> 2. 可以理解成分布式系统中的控制节点，在 Flink 中的 JobManager 中运行或者在单个 TaskManager 中运行，不会在多台机器中同时运行。 | KafkaSource 的实现 KafkaSourceEnumerator <br>  负责动态地发现和分配 Kafka 分区（TopicPartitions）给 Source 任务（Reader）。 |
| `SourceReader<T, SplitT>` |  Flink 用于读取切片，生产数据的组件 <br> 1. 负载接受 Split 然后生产出 T(消息泛型) <br>  2. 可以在多个 TaskManager 中运行 | KafkaSourceReader 基于 Base 实现，读取 partition 的数据 |
| `SimpleVersionedSerializer` | Flink 中对于 Split 与 EnumChk 需要提供序列化和反序列化的能力 <br>  - Split 的序列化主要是用于 SplitEnumerator 与 SourceReader。同时用于 SourceReader 的 Snapshot 序列化 <br> - EnumChk 用于 Checkpoint 的持久化 | Kafka 的 KafkaPartitionSplitSerializer 与 KafkaSourceEnumStateSerializer |

### API 中的泛型
| 泛型名称 | 泛型用途 | 使用方 | 示例 |
| --- | --- | --- | --- |
| T | 表示 Source 生成的消息 | SourceReader | Kafka 的是外围传递的泛型 |
| SplitT | 表示 SplitEnumerator 生成的切片类型 | SplitEnumerator 与 SourceReader | Kafka 是 KafkaPartitionSplit，(topicPartition, startOffset, endOffset) |
| EnumChk | 表示 Checkpoint 类型 | SplitEnumerator | Kafka 是 KafkaSourceEnumState `Set<TopicPartition> assignedPartitions` |


### SourceReaderBase 实现
| 类名 | 用途 | 示例 |
| --- | --- | --- |
| `SourceReaderBase<E, T, SplitT, SplitStatus>` | 因为 SourceReader 的高灵活性，简化了 SourceReader 的实现，定义了如何读取外部数据源（通过 SplitReader），同时通过构造方法确定了工作线程模型（`SplitFetcherManager`）。 | |
| `SingleThreadMultiplexSourceReaderBase<E, T, SplitT, SplitStatus>` | SingleThreadMultiplexSourceReaderBase 是根据 SourceReaderBase 实现的一个特定线程模型，专为实现**单线程多路复用的数据读取场景设计**。它允许在一个单独的线程中同时管理多个数据源或数据分片（splits）的读取，这对于那些每个数据源或分片的读取并不需要独立线程，或者资源有限希望减少线程开销的场景非常有用。 | KafkaSourceReader |
| `RecordEmitter<E, T, SplitStatus>` | 通常用于将从数据源读取到的数据转换为Flink的内部数据结构（泛型 E），它是数据读取逻辑的一部分，帮助**将原始数据格式转换为Flink可以处理的形式** |  |
| `SplitFetcherManager<E, SplitT>` | 负责管理所有的 SplitFetcher 实例。在 Flink 中，数据源可能会被切分成多个 Split，以支持并行读取。SplitFetcherManager 确保每个 Split 都有相应的 SplitFetcher 负责从该分片读取数据，并且管理这些 fetcher 的生命周期。 | SingleThreadFetcherManager |
| `FutureCompletingBlockingQueue<RecordsWithSplits>` | 可以理解为一个异步阻塞队列，之所以单独拿出来，是它承担了SplitReader 读出来数据，将其传递到 SourceReaderBase 的职责 |  |
| `SplitContext` | 针对每个 Split 生成一个 SplitContext，是 SourceReaderBase 的一个 private inner class. 用于生成 SourceOutput |  |
| `SourceOutput` | 与 Flink 框架层交互的入口 | SourceOutputWithWatermarks 根据 WatermarkGenerator 来通过 Record 生成 Watermark，然后将数据发送给下游 |
| `WatermarkOutput<T>` | 在 SourceOutput 发送 Record 过程中，通过 Record 生成 Watermark 并发送到框架中 |  |
| `SplitFetcher<E, Split>` | 负责从特定的split中获取数据，并将数据放入到 FutureCompletingBlockingQueue 中。它是数据读取逻辑的核心部分，实现了如何与外部数据源交互的具体逻辑 |  |
| `SplitFetcherTask<E, Split>` | `SplitFetcherManager` 管理的实体接口，定义了 `run` 与 `wakeup`两个方法 |  |
| `FetchTask<E, Split>` | 实现 SplitFetcherTask，驱动 SplitReader 的 fetch 方法 |  |
| `DummySplitFetcherTask<E, Split>` | `SplitFetcher`中的一个标记类，判断此时是不是处于 `wakeup`状态 |  |
| `AddSplitsTask<E, Split>` | 实现 SplitFetcherTask，驱动 SplitReader 的 handleSplitChanges 方法 |  |
| `SplitReader<E, Split>` | 读取数据和处理切片逻辑 | KafkaPartitionSplitReader 根据 partition 读取 (startOffset, endOffset) 之间的数据 |
| `RecordsWithSplits<E>` | 提供了一个类 Iterator 来读取 Record 的接口，不同的是，需要同时提供 Split 的生命周期管理能力，需要在读取完成一批消息之后能获知下一个 Split 是什么以方便初始化 SplitStatus，在读取完所有的 Split 之后需要能知道哪些 Split 已经完成了。 | KafkaPartitionSplitRecords |

## 为什么要这么设计的一些自己的理解
### 为什么要区分 Split 和 Reader？
将数据源读取的逻辑解耦成两部分：

1. SplitEnumerator 负责划分数据源读取逻辑，并承担分布式系统中的中心分配任务角色，将数据源需要的顺序性在这里实现。
2. SourceReader 仅专注于负责读数据，具备可扩展性，可以通过增加 Task Slot 实现水平扩展。并且容灾也依赖于 SourceReader，当一个 SourceReader 任务节点故障之后，SplitEnumerator 可以通过向其他健康的 SourceReader 发送 Split 即可恢复整体系统的数据读取。
### SplitStatus 是什么？
SplitStatus 可以理解是细化的 Split，`SourceReaderBase`中通过 `initializedState`/`toSplitType`两个方法来达到 Split 与 SplitStatus 之间的相互转换，然后通过 `snapshotState`方法将 SplitStatus 转换成根据 CheckpointId 存储的 snapshot。
SplitStatus 通常是依赖 `RecordEmitter`进行推进，比如 Kafka 中 `KafkaRecordEmitter`接收到一个 Record 就将 `KafkaPartitionSplitState`的 currentOffset 推进一位，这样确保退出或容灾情况下的 snapshot 是最近读到的 offset 开始。
### Flink-Source Connector 的 Base 是期望引导我们向哪个方向实现 Source？

1. 职责分离，将有顺序要求的部分放入 SplitEnumerator 来实现，SourceReader 专注实现读取数据的能力。
2. 状态管理分离，期望 SplitEnumerator 与 SourceReader 分别管理自身的状态，SplitEnumerator 负责产生 EnumChk 用于自身的断点及容灾，而对于已经发送的 Split 期望是 SourceReader 自身进行管理，分布式的方式，不需要统一汇集到某处管理整个任务的状态。
3. 线程模型的多样化，SourceReaderBase 默认提供了单线程读 Split 的 SingleThreadMultiplexSourceReaderBase，这种在 SourceReader 读消息比较耗时的情况下实际使用资源会较少，但同时也允许直接继承 SourceReaderBase 实现，然后自己实现 SplitFetcherManager 来实现 SourceReader 的线程管理模型。
### Checkpoint 是什么？
Checkpoint 是 Flink 用于容灾的状态数据，通过 Checkpoint Flink 可以对作业的状态和计算位置进行恢复。

1. **开启：**可以通过 `StreamExecutionEnvironment#enableCheckpointing(timeInterval)`来开启 Checkpoint。周期性（timeInterval）触发。
2. **发送：**Checkpoint 是由框架的 JobManager 发起，会向参与任务的 TaskManager 发出一个 checkpoint 开始的 barrier（一种特殊的消息）。这个 barrier 会随着数据流一起流动，并且会确保在 barrier 之前的所有数据都已经被处理完成。
3. **状态快照：**每个 TaskManager 收到 barrier 后，会对其管理的任务状态进行快照。这包括了算子的状态、时间窗口信息等。状态快照是异步进行的，以减少对数据处理的影响。
4. **持久化：**状态快照完成后，这些快照会被持久化存储到外部存储系统中（如 HDFS、S3 等），以保证即使整个集群失败也能恢复数据。
5. **完成确认：**一旦所有任务的状态快照都成功持久化，JobManager 会收到确认并标记此检查点为已完成。此时，之前的旧检查点可以被清理以节省存储空间。

对于 Source 端来说，Checkpoint 主要影响其以下方面：

1. **记录读取位置：**Source 端在接收到 barrier 时，会记录当前的数据读取位置（例如，在 Kafka 中就是当前消费到的 offset）。这个位置信息会作为 checkpoint 的一部分被持久化，这样在恢复时可以从这个位置重新开始读取数据，避免数据重复处理或丢失。`SplitEnumerator#snapshotState`与`SourceReader#snapshot`方法
2. **暂停数据读取：**为了确保一致性，在 barrier 通过时，Source 可能需要暂时停止从源头读取新数据，直到当前的 checkpoint 完成。这样做是为了保证 barrier 和其之前的数据包能够作为一个整体被处理和快照。
3. **恢复逻辑：**当 Flink 应用因故障重启后，它会从最近成功的 checkpoint 恢复执行。这时候，Source 会根据之前记录的读取位置信息重新初始化自己，**从正确的偏移量开始读取数据，从而保证数据处理的精确一次语义**。
### Watermark 是什么？
Watermark 在数据结构上是一个 `long timestamp`的数据封装的 final class。代表数据流中的进度标记，表示到目前为止所有小于或等于该时间戳的事件都已到达系统。简单来说，Watermark 是一种逻辑时钟，帮助Flink理解数据流中时间的进展，从而在处理乱序事件时能够做出基于时间的决策，尤其是处理时间窗口聚合时。
Watermark 的生成是通过 `WatermarkGenerator`来配置确定的，使用 Flink API 时需要设置一个同时包含 `TimestampAssigner` 和 `WatermarkGenerator` 的 `WatermarkStrategy`。`WatermarkStrategy` 工具类中也提供了许多常用的 watermark 策略，并且用户也可以在某些必要场景下构建自己的 watermark 策略。
从 Source 接口的角度来看，当 Record 通过 `RecordEmitter#emitRecord`发送到框架的时候，`SourceOutput`会透出 `void collect(T record, long timestamp);`接口，而这个 timestamp 会根据 `TimestampAssigner`与 `WatermarkGenerator`来生成 `Watermark`。

## 后续

- 介绍 SourceReaderBase 拉取数据的流程以及其中的线程模型

## 参考

- [FLIP-27: Refactor Source Interface](https://cwiki.apache.org/confluence/display/FLINK/FLIP-27%3A+Refactor+Source+Interface)
- [Apache Flink 文档](https://nightlies.apache.org/flink/flink-docs-release-1.20/zh/)

