---
title: Kafka生产者分析——KafkaProducer
date: 2017-12-18 22:15:26
tags:
    - Kafka
    - 分布式与中间件
categories: 源码分析
---

>  文章参考自《Apache  Kafka 源码剖析》
>
>  github 地址：https://github.com/BingLau7/kafka，里面有代码的相关注释
>
>  具体部署方式不进行指导了，网上资料比较多

## KafkaProducer Demo

```java
public class ProducerDemo {
    public static void main(String[] args) {
        boolean isAsync = args.length == 0 ||
                /* 消息的发送方式：异步发送还是同步发送 */
                !args[0].trim().equalsIgnoreCase("sync");

        Properties props = new Properties();
        props.put("bootstrap.servers", "localhost:9092");
        /* 客户端的 ID */
        props.put("client.id", "DemoProducer");
        /*
        * 消息的 key 和 value 都是字节数组，为了将 Java 对象转化为字节数组，可以配置
        * "key.serializer" 和 "value.serializer" 两个序列化器，完成转化
        */
        props.put("key.serializer", "org.apache.kafka.common.serialization.IntegerSerializer");

        /* StringSerializer 用来将 String 对象序列化成字节数组 */
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

        /* 生产者的核心类 */
        KafkaProducer producer = new KafkaProducer<>(props);

        /* 向指定的 test 这个 topic 发送消息 */
        String topic = "test";

        /* 消息的 key */
        int messageNo = 1;

        while (true) {
            String messageStr = "Message_" + messageNo;
            long startTime = System.currentTimeMillis();

            if (isAsync) { /* 异步发送消息 */
                /*
                *  第一个参数是 ProducerRecord 类型的对象，封装了目标 Topic，消息的 kv
                *  第二个参数是一个 CallBack 对象，当生产者接收到 Kafka 发来的 ACK 确认消息的时候，
                *  会调用此 CallBack 对象的 onCompletion() 方法，实现回调功能
                */
                producer.send(new ProducerRecord<>(topic, messageNo, messageStr),
                        new DemoCallBack(startTime, messageNo, messageStr));
            } else { /* 同步发送消息 */
                try {
                    /*
                    * KafkaProducer.send() 方法的返回值类型是 Future<RecordMetadata>
                    * 这里通过 Future.get 方法，阻塞当前线程，等待 Kafka 服务端的 ACK 响应
                    */
                    producer.send(new ProducerRecord<>(topic, messageNo, messageStr)).get();
                    System.out.printf("Send message: (%d, %s)\n", messageNo, messageStr);
                } catch (InterruptedException | ExecutionException e) {
                    e.printStackTrace();
                }
            }
            /* 递增消息的 key */
            ++messageNo;
        }
    }
}

class DemoCallBack implements Callback {
    /* 开始发送消息的时间戳 */
    private final long startTime;
    private final int key;
    private final String message;

    public DemoCallBack(long startTime, int key, String message) {
        this.startTime = startTime;
        this.key = key;
        this.message = message;
    }

    /**
     * 生产者成功发送消息，收到 Kafka 服务端发来的 ACK 确认消息后，会调用此回调函数
     * @param metadata 生产者发送的消息的元数据，如果发送过程中出现异常，此参数为 null
     * @param exception 发送过程中出现的异常，如果发送成功为 null
     */
    @Override
    public void onCompletion(RecordMetadata metadata, Exception exception) {
        long elapsedTime = System.currentTimeMillis() - startTime;
        if (metadata != null) {
            System.out.printf("message: (%d, %s) send to partition %d, offset: %d, in %d\n",
                    key, message, metadata.partition(), metadata.offset(), elapsedTime);
        } else {
            exception.printStackTrace();
        }
    }
}
```

<!--more-->

## KafkaProducer 分析

### 整体流程

![KafkaProducer流程](https://github.com/BingLau7/blog/blob/master/images/blog_42/Kafka.png?raw=true)

上图是一个  KafkaProducer 在发送消息的整个流程，我们就上面来进行一个宏观的了解：

1. ProducerInterceptors 对消息进行拦截
2. Serializer 对消息的 key 和 value 进行序列化
3. Partitioner 为消息选择合适的 Partition
4. RecordAccumulator 收集消息，实现批量发送
5. Sender 从 RecordAccumulator 获取消息
6. 构造 ClientRequest
7. 将 ClientRequest 交给 NetworkClient，准备发送
8. NetworkClient 将请求放入 KafkaChannel 的缓存
9. 执行网络 I/O，发送请求
10. 收到响应，调用 ClientRequest 的回调函数
11. 调用  RecordBatch 的回调函数，最终调用每个消息上注册的回调函数

消息发送的过程中，涉及两个线程协同工作。主线程首先将业务数据封装或 ProducerRecord 对象，之后调用 `send()` 方法将消息放入 RecordAccummulator (消息收集器，也可以理解为主线程与 Sender 线程之间的缓冲区)中暂存。Sender 线程负责将消息信息构成请求，并最终执行网络 I/O 的线程，它从 RecordAccumulator 中取出消息并批量发送出去。需要注意的是，KafkaProducer 是线程安全的，多个线程间可以共享使用同一个 KafkaProucer 对象。

KafkaProducer 实现了 Producer 接口，在 Producer 接口中定义 KafkaProducer 对外提供的 API，分为四类方法：

-  `send()` 方法：发送消息，实际是将消息放入 RecordAccumulator 暂存，等待发送
-  `flush()` 方法：刷新操作，等待 RecordAccumulator 中所有消息发送完成，在刷新之前就会阻塞调用的线程
-  `partitionsFor()` 方法：在 KafkaProducer 中维护了一个 Metadata 对象用于存储 Kafka 集群的元素局，Metadata 中的元素局会定时更新。`partitionsFor()` 方法负责从 Metadata 中获取指定 Topic 中的分区信息。
-  `close()` 方法：关闭此 Producer 对象，主要操作是设置 close 壁纸，等待 `RecordAccumulator` 中的消息清空，关闭 Sender 线程。

### KafkaProducer 重要字段

```java
    /* clientId 的生成器，如果没有明确指定 client 的 id，则使用字段生成一个 ID */
    private static final AtomicInteger PRODUCER_CLIENT_ID_SEQUENCE = new AtomicInteger(1);
    /* 此生产者的唯一标识 */
    private String clientId;
    /* 分区选择器，根据一定的策略，将消息路由到合适的分区 */
    private final Partitioner partitioner;
    /* 消息的最大长度，这个长度包含了消息头、序列化后的 key 和序列化后的 value 的长度 */
    private final int maxRequestSize;
    /* 发送单个消息的缓冲区大小 */
    private final long totalMemorySize;
    /* 存储 Kafka 集群的元数据 */
    private final Metadata metadata;
    /* RecordAccumulator，用于手机并缓存消息，等待 Sender 线程发送 */
    private final RecordAccumulator accumulator;
    /* 发送消息的 Sender 任务，实现了 Runnable 接口，在 ioThread 线程中执行 */
    private final Sender sender;
    private final Metrics metrics;
    /* 执行 Sender 任务发送消息的线程，称为 『Sender 线程』 */
    private final Thread ioThread;
    /* 压缩算法，可选项有 none、gzip、snappy、lz4.
    * 这是针对 RecordAccumulator 中多条消息进行的压缩，所以消息越多，压缩效果越好 */
    private final CompressionType compressionType;
    /* key 的序列化器 */
    private final Serializer<K> keySerializer;
    /* value 的序列化器 */
    private final Serializer<V> valueSerializer;
    /* 配置对象，使用反射初始化 KafkaProducer 配置的相对对象 */
    private final ProducerConfig producerConfig;
    /* 等待更新 Kafka 集群元数据的最大时长 */
    private final long maxBlockTimeMs;
    /* 消息的超时时间，也就是从消息发送到收到 ACK 响应的最长时长 */
    private final int requestTimeoutMs;
    /* ProducerInterceptor 集合，ProducerInterceptor 可以在消息发送之前对其进行拦截或修改；
    * 也可以先于用户的 Callback，对 ACK 响应进行预处理 */
    private final ProducerInterceptors<K, V> interceptors;
```

在 KafkaProducer 的构造函数中，会初始化上面介绍的字段，其中有几个需要注意：

```java
    private KafkaProducer(ProducerConfig config, Serializer<K> keySerializer, Serializer<V> valueSerializer) {
        try {
			....
            /* 通过反色机制实例化配置的 partitioner 类，keySerializer 类，valueSerializer 类 */
            this.partitioner = config.getConfiguredInstance(ProducerConfig.PARTITIONER_CLASS_CONFIG, Partitioner.class);
            long retryBackoffMs = config.getLong(ProducerConfig.RETRY_BACKOFF_MS_CONFIG);
            if (keySerializer == null) {
                this.keySerializer = config.getConfiguredInstance(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
                        Serializer.class);
                this.keySerializer.configure(config.originals(), true);
            } else {
                config.ignore(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG);
                this.keySerializer = keySerializer;
            }
            if (valueSerializer == null) {
                this.valueSerializer = config.getConfiguredInstance(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
                        Serializer.class);
                this.valueSerializer.configure(config.originals(), false);
            } else {
                config.ignore(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG);
                this.valueSerializer = valueSerializer;
            }
            ...
            // 创建并更新 Kafka 集群的元数据
            this.metadata = new Metadata(retryBackoffMs, config.getLong(ProducerConfig.METADATA_MAX_AGE_CONFIG), true, clusterResourceListeners);
			...

            /* 创建 RecordAccumulator */
            this.accumulator = new RecordAccumulator(config.getInt(ProducerConfig.BATCH_SIZE_CONFIG),
                    this.totalMemorySize,
                    this.compressionType,
                    config.getLong(ProducerConfig.LINGER_MS_CONFIG),
                    retryBackoffMs,
                    metrics,
                    time);

			...
            /* 创建 NetworkClient，这个是 KafkaProducer 网络 I/O 的核心，在后面会详细介绍 */
            NetworkClient client = new NetworkClient(
                    new Selector(config.getLong(ProducerConfig.CONNECTIONS_MAX_IDLE_MS_CONFIG), this.metrics, time, "producer", channelBuilder),
                    this.metadata,
                    clientId,
                    config.getInt(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION),
                    config.getLong(ProducerConfig.RECONNECT_BACKOFF_MS_CONFIG),
                    config.getInt(ProducerConfig.SEND_BUFFER_CONFIG),
                    config.getInt(ProducerConfig.RECEIVE_BUFFER_CONFIG),
                    this.requestTimeoutMs, time);
		   ...
            String ioThreadName = "kafka-producer-network-thread" + (clientId.length() > 0 ? " | " + clientId : "");
            /* 启动 Sender 对应的线程 */
            this.ioThread = new KafkaThread(ioThreadName, this.sender, true);
            this.ioThread.start();
            ...
   }
```

KafkaProducer 构造完成之后，我们来关注 KafkaProducer 的 `send()` 方法

![send方法流程图](https://github.com/BingLau7/blog/blob/master/images/blog_42/KafkaProducer%23send().png?raw=true)

图中关键步骤:

-  调用 `ProducerInterceptors.onSend()` 方法，通过 `ProducerInterceptor` 对 消息进行拦截或修改
-  调用 `watiOnMetadata()` 方法获取 Kafka 集群的信息，底层会唤醒 Send 线程更新 Metadata 中保存的 Kafka 集群元数据
-  调用 `Serializer.serialize()` 方法序列号消息的 key 和 value
-  调用 `partition()` 为消息选择合适的分区
-  调用 `RecordAccumulator.append()` 方法，将消息追加到 `RecordAccumulator` 中
-  唤醒 `Sender` 线程将 `RecordAccumulator` 中缓存的消息发送出去

### `ProducerInterceptors&ProducerInterceptor`

`ProducerInterceptors` 是一个 `ProducerInterceptor` 的集合，其 `onSend` 方法、`onAcknowledgement` 方法、`onSendError` 方法，实际上是循环调用其封装的 `ProducerInterceptor` 集合的对应方法。

`ProducerIntercepto` 对象可以在消息发送之前对其进行拦截或修改，也可以先于用户的 Callback，对 ACK 响应进行预处理。如果要使用自定义 `ProducerInterceptor` 类，只要实现 `ProducerInterceptor` 接口，创建其对象并添加到 `ProducerInterceptors` 中即可。

### Kafka 集群元数据

Kafka 中每个 Topic 中有个多个分区，这些分区的 Leader 副本可以分配在集群中不同的 Broker 上。在运行过程中，Leader 副本随时都可能出现故障而导致 Leader 副本重新选举，新的 Leader 副本会在其他 Broker 上继续提供对外服务，所以由于种种原因分区的数量以及 Leader 副本的分布是动态变化的。当需要提高某 Topic 的并发处理消息能力时，我们可以通过增加其分区的数量来实现。

在 KafkaProducer 中，使用 Node、TopicPartition、PartitionInfo 这三个类封装了 Kafka 集群的相关元数据，其主要字段：

![Node&TopicPartition&PartitionInfo](https://github.com/BingLau7/blog/blob/master/images/blog_42/Node&TopicPartition&PartitionInfo.jpg?raw=true)

-  Node 表示集群中的一个节点，Node 记录这个节点的 host、ip、port等新兴
-  TopicPartition 表示某个 Topic 的一个分区，其中的 topic 字段是 Topic 的名称，partition 则是该分区编号(ID)
-  PartitionInfo 表示一个分区的详细信息

通过这三个类的组合，我们可以完整表示出 KafkaProducer 需要的集群元数据。这些元数据保存在 `Cluster` 这个类中，并按照不同的映射方式进行存放，方便查询。

![Cluster](https://github.com/BingLau7/blog/blob/master/images/blog_42/Cluster.jpg?raw=true)

-  nodes: Kafka 集群中节点信息列表
-  nodesById：BrokerId 与 Node 节点之间对应关系，方便按照 BrokerId 进行索引
-  partitionsByTopicPartition：记录了 TopicPartition 与 PartitionInfo 之间的映射关系
-  partitionsByTopic：记录了 Topic 名称和 PartitionInfo 的映射关系，可以按照 Topic 名称查询其中全部分区的详细信息。
-  avaliablePartitionByTopic：Topic 与 PartitionInfo 的映射关系，这里的  `List<PartitionInfo>` 中存放的分区必须是有 Leader 副本的 Partition，而 partitionByTopic 中记录的分区则不一定有 Leader 副本，因为某些中间状态。
-  partitionsByNode: 记录了 Node 与 PartitionInfo 的映射关系，可以按照节点 Id 查询其上分布的全部分区的详细信息。

Metadata 中封装了 Cluster 对象，并保持 Cluster 数据的最后更新时间、版本号（version）、是否需要更新等待信息

```java
public final class Metadata {

    private static final Logger log = LoggerFactory.getLogger(Metadata.class);

    public static final long TOPIC_EXPIRY_MS = 5 * 60 * 1000;
    private static final long TOPIC_EXPIRY_NEEDS_UPDATE = -1L;

    /* 两次发出更新 Cluster 保存的元数据信息的最小时间差，默认为 100ms。防止更新操作过于频繁而造成
     * 网络阻塞和增加服务端压力。在 Kafka 中与重试操作有关的操作中，都有这种『退避(backoff)时间』
     * 设计的身影 */
    private final long refreshBackoffMs;
    /* 每隔多久更新一次 */
    private final long metadataExpireMs;
    /* 表示 Kafka 集群元数据的版本号。Kafka 集群元数据每更新成功一次，version++ */
    private int version;
    /* 记录上次更新元数据的时间戳（也包含更新失败的情况） */
    private long lastRefreshMs;
    /* 上一次成功更新的时间戳 */
    private long lastSuccessfulRefreshMs;
    /* 记录 Kafka 集群的元数据 */
    private Cluster cluster;
    /* 标识是否强制更新 Cluster，这是触发 Sender 线程更新集群元数据的条件之一 */
    private boolean needUpdate;
    /* Topics with expiry time */
    /* 记录了当前已知的所有 topic，在 cluster 字段中记录了 Topic 最新的元数据，并记录了其数据对应的过期时间 */
    private final Map<String, Long> topics;
    /* 监听 Metadata 更新的监听器集合。自定义 Metadata 监听实现 MetadataListener.onMetadataUpdate()
    * 方法即可，在更新 Metadata 中的 cluster 字段之前，会通知 listener 集合中全部 Listener 对象*/
    private final List<Listener> listeners;
    private final ClusterResourceListeners clusterResourceListeners;
    /* 是否需要更新全部 Topic 的元数据，一般情况下 KafkaProducer 只维护它用到的 Topic 的元数据，
     * 是集群中全部 Topic 的子集 */
    private boolean needMetadataForAllTopics;
    private final boolean topicExpiryEnabled;
  
  ...
}
```

Metadata 的方法比较简单，主要是操纵上面的几个字段，这里着重介绍主线程中使用到的 `requestUpdate()` 和 `awaitUpdate()` 方法。

-  `requestUpdate()` 方法将 `needUpdate` 字段修改为 `true` ，这样当 Sender 线程运行时更新 Metadata 记录的集群元数据，然后返回 version 字段的值。

   ```java
       public synchronized int requestUpdate() {
           this.needUpdate = true;
           return this.version;
       }
   ```

   ​

-  `awaitUpdate()` 方法主要是通过 version 版本号来判断元数据是否更新完成，更新为完成则阻塞等待。

   ```java
       public synchronized void awaitUpdate(final int lastVersion, final long maxWaitMs) throws InterruptedException {
           if (maxWaitMs < 0) {
               throw new IllegalArgumentException("Max time to wait for metadata updates should not be < 0 milli seconds");
           }
           long begin = System.currentTimeMillis();
           long remainingWaitMs = maxWaitMs;
           /* 比较版本号，通过版本号比较集群元数据是否更新完成 */
           while (this.version <= lastVersion) {
               /* 主线程与 Sender 通过 wait/notify 同步，更新元数据的操作则交给 Sender 线程去完成 */
               if (remainingWaitMs != 0)
                   wait(remainingWaitMs);
               long elapsed = System.currentTimeMillis() - begin;
               if (elapsed >= maxWaitMs)
                   throw new TimeoutException("Failed to update metadata after " + maxWaitMs + " ms.");
               remainingWaitMs = maxWaitMs - elapsed;
           }
       }
   ```

   需要注意的是，Metadata 中的字段可以由主线程读，Sedner 线程更新，因此它必须是线程安全的，这也是上面为什么所有方法都使用 synchronized 同步的原因。

   下面介绍 `KafkaProducer.waitOnMetadta()` 方法(`KafkaProducer#doSend调用`)，它负责触发 Kafka 集群元数据的更新，并阻塞主线程等等更新完毕。它的主要步骤是：

   1. 直接添加 topic 进入 metadata 中，如果已经存在则更新其过期时间
   2. 尝试获取 Topic 中分区的详细信息，失败后会调用 `requestUpdate()` 方法设置 `Metadata.needUpdate` 字段，并得到当前元数据版本号
   3. 唤醒 Sender 线程，由 Sender 线程更新 Metadata 中保存的 Kafka 集群元数据。
   4. 主线程调用 `awaitUpdate()` 方法，等待 Sender 线程完成更新
   5. 从 Metadata 中获取指定 Topic 分区的详细信息（即 PartitionInfo 集合）。若失败，则回到步骤2继续尝试，若等待时间超时，则抛出异常。

   其具体实现如下：

   ```java
       private ClusterAndWaitTime waitOnMetadata(String topic, Integer partition, long maxWaitMs) throws InterruptedException {
           // add topic to metadata topic list if it is not there already and reset expiry
           metadata.add(topic);
           /* 获取分区信息 */
           Cluster cluster = metadata.fetch();
           Integer partitionsCount = cluster.partitionCountForTopic(topic);
           // Return cached metadata if we have it, and if the record's partition is either undefined
           // or within the known partition range
           if (partitionsCount != null && (partition == null || partition < partitionsCount))
               return new ClusterAndWaitTime(cluster, 0);

           long begin = time.milliseconds();
           long remainingWaitMs = maxWaitMs;
           long elapsed;
           // Issue metadata requests until we have metadata for the topic or maxWaitTimeMs is exceeded.
           // In case we already have cached metadata for the topic, but the requested partition is greater
           // than expected, issue an update request only once. This is necessary in case the metadata
           // is stale and the number of partitions for this topic has increased in the meantime.
           do {
               log.trace("Requesting metadata update for topic {}.", topic);
               /* 获取失败之后调用 requestUpdate() 方法，并获取当前元数据版本号 */
               int version = metadata.requestUpdate();
               /* 唤醒 Sender 线程 */
               sender.wakeup();
               try {
                   /* 阻塞等待元数据更新完毕 */
                   metadata.awaitUpdate(version, remainingWaitMs);
               } catch (TimeoutException ex) {
                   // Rethrow with original maxWaitMs to prevent logging exception with remainingWaitMs
                   throw new TimeoutException("Failed to update metadata after " + maxWaitMs + " ms.");
               }
               cluster = metadata.fetch();
               elapsed = time.milliseconds() - begin;
               /* 检测超时时间 */
               if (elapsed >= maxWaitMs)
                   throw new TimeoutException("Failed to update metadata after " + maxWaitMs + " ms.");
               /* 检测权限 */
               if (cluster.unauthorizedTopics().contains(topic))
                   throw new TopicAuthorizationException(topic);
               remainingWaitMs = maxWaitMs - elapsed;
               partitionsCount = cluster.partitionCountForTopic(topic);
           } while (partitionsCount == null);

           if (partition != null && partition >= partitionsCount) {
               throw new KafkaException(
                       String.format("Invalid partition given with record: %d is not in the range [0...%d).", partition, partitionsCount));
           }

           return new ClusterAndWaitTime(cluster, elapsed);
       }
   ```

   ### `Serializer&Deserializer`

   客户端发送的消息的 key 和 value 都是 byte 数组， `Serializer` 和 `Deserializer` 接口提供了将 Java 对象序列号（反序列化）为 byte 数组的功能。在 KafkaProducer 中，根据配置文件，使用合适的 `Serializer` 。

   ![ImplementSerializer](https://github.com/BingLau7/blog/blob/master/images/blog_42/implementSerializer.jpg?raw=true) 

![ImplementDeserializer](https://github.com/BingLau7/blog/blob/master/images/blog_42/implementDeserializer.jpg?raw=true)

Kafka 已经为我们提供了 Java 基本类型的 Serializer 实现和 Deserializer 实现，我们也可以对 Java 复杂类型的自定义 Serializer 和 Deserializer 实现。

在 Serializer 接口中， `configure()` 方法是在执行序列化操作之前的配置，例如，在 `StringSerializer.configure()` 方法中会选择合适的编码（encoding），默认是 UTF-8；`serializer()` 方法是真正进行序列化的地方，将传入的 Java 对象序列化为 byte[]。`close()` 方法是在其后的关闭方法，多为空实现。

### Partitioner

`KafkaProducer.send()` 方法的下一步操作是选择消息的分区。在有的应用场景中，由业务逻辑控制每个消息追加到合适的分区中，而有时候业务逻辑并不关心分区的选择。在 `KafkaProducer.partition()` 方法中，优先根据 `ProducerRecord` 中 `partition` 字段指定的序号选择分区，如果 `ProducerRecord.partition` 字段没有明确指定分区编号，则通过 `Partitioner.partition()` 方法选择 partition。

Kafka 提供了 Partitioner 接口的一个默认实现 `DefaultPartitioner`

![DefaultPartitioner](https://github.com/BingLau7/blog/blob/master/images/blog_42/DefaultPartitioner.jpg?raw=true)

可以看到，之前介绍的 `ProducerInterceptor` 接口也继承了 `Configurable` 接口。

在创建 KafkaProducer 时传入的 key/value 配置项会保存到 `AbstractConfig` 的 `originals` 字段中，`AbstractConfig` 的核心方法是 `getConfiguredInstance()` 方法，其主要功能是通过反射机制实例化 `originals` 字段中指定的类。

设计 `Configurable` 接口的**目的是统一反射后的初始化过程**，对外提供同意的初始化接口。在 `AbstractConfig.getConfiguredInstance` 方法中通过反射构造出来的对象，都是通过无参构造函数构造成功的，需要初始化的字段个数和类型各式各样， `Configurable` 接口的 `configure()` 方法封装了对象初始化过程且只有一个参数 （`originals`）字段，这样对外的接口就变得统一了。 

`DefaultPartitioner.partition()` 方法负责在 `ProducerRecord` 中没有明确指定分区编号的时候，为其选择合适的分区， `count` 不断递增，确保消息不会都发到同一个 `Partition` 里；如果消息有 key 的话，则对 key 进行 hash（murmur2），然后与分区数量取模，来确定 key 所在分区到达负载均衡。

```java
    /* counter 初始化为一个随机数，注意，这里是一个 AtomicInteger */
    /* KafkaProducer 必须是一个线程安全类，多个业务发送数据时候也必须保证 Partitioner 线程安全 */
    private final AtomicInteger counter = new AtomicInteger(new Random().nextInt());

    /**
     * Compute the partition for the given record.
     *
     * @param topic The topic name
     * @param key The key to partition on (or null if no key)
     * @param keyBytes serialized key to partition on (or null if no key)
     * @param value The value to partition on or null
     * @param valueBytes serialized value to partition on or null
     * @param cluster The current cluster metadata
     */
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        /* 从 Cluster 中获取对应的 Topic 的分区信息 */
        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
        /* 分区数量 */
        int numPartitions = partitions.size();
        /* 没有 key 的情况 */
        if (keyBytes == null) {
            /* 递增 counter */
            int nextValue = counter.getAndIncrement();
            /* 选择 avaliablePartitions */
            List<PartitionInfo> availablePartitions = cluster.availablePartitionsForTopic(topic);
            if (availablePartitions.size() > 0) {
                int part = Utils.toPositive(nextValue) % availablePartitions.size();
                return availablePartitions.get(part).partition();
            } else {
                // no partitions are available, give a non-available partition
                return Utils.toPositive(nextValue) % numPartitions;
            }
        } else { /* 消息有可以的情况 */
            // hash the keyBytes to choose a partition
            return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
        }
    }
```

