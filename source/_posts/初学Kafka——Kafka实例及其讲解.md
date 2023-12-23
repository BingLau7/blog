---
title: 初学Kafka——Kafka实例及其讲解
date: 2016-03-22 16:21:45
tags:
    - Kafka
    - 分布式与中间件
categories: 工具
---

## 基本概念

**Topic：**Kafka将消息种子(Feed)分门别类， 每一类的消息称之为话题(Topic).
**Producer：**发布消息的对象称之为话题生产者(Kafka topic producer)
**Consumer：**订阅消息并处理发布的消息的种子的对象称之为话题消费者(consumers)
**Broker：**已发布的消息保存在一组服务器中，称之为Kafka集群。集群中的每一个服务器都是一个代理(Broker). 消费者可以订阅一个或多个话题，并从Broker拉数据，从而消费这些已发布的消息。
**Partition：**每个Topic下会有一个或多个Partition，创建Topic时可指定Parition数量。每个Partition对应于一个文件夹，该文件夹下存储该{Partition的数据和索引文件。

<!-- more -->

Kafka就是讲生产者所生产的消息进行分类，而分类的条目就是**Topic**，最后由消费者接受消息的一个消息中间件。而消费者不是直接从生产者那里拿数据，而是从**Broker**从拉取数据，所谓**Broker**其实是一个服务器实例，负责接受从生产者那里来的数据，负责消息维护作用。

对于每个Topic，Kafka都会这一个分区的log，如图所示：
![kafka partition](/uploads/kafka_partition.jpg)
每一个分区都是一个顺序的、不可变的消息队列， 并且可以持续的添加。分区中的消息都被分配了一个序列号，称之为**偏移量(offset)**,在每个分区中此偏移量都是唯一的（可以理解为Mysql中一张表的主键id）。Kafka集群保持所有的消息，直到它们过期，无论消息是否被消费了。实际上消费者所持有的仅有的元数据就是这个偏移量，也就是消费者在这个log中的位置。 这个偏移量由消费者控制：正常情况当消费者消费消息的时候，偏移量也线性的的增加。但是实际偏移量由消费者控制，消费者可以将偏移量重置为更老的一个偏移量，重新读取消息。可以看到这种设计对消费者来说操作自如， 一个消费者的操作不会影响其它消费者对此log的处理。

再说说分区。Kafka中采用分区的设计有几个目的：
1. 是可以处理更多的消息，不受单台服务器的限制。Topic拥有多个分区意味着它可以不受限的处理更多的数据。
2. 分区可以作为并行处理的单元。
## 命令行执行

我们可以先使用命令行才测试Kafka的简单操作。

先手动启动一个生产者者

```
>kafka-console-producer.sh --broker-list 192.168.33.10:9092 --topic test
This is message
```

在手动启动一个消费者

```
>kafka-console-consumer.sh --zookeeper 192.168.33.10:2181 --topic test --from-beginning
This is message //这就是收到的消息
```

这里要说明一下，我是使用Vagrant来做Kafka的服务器，连接时候会出现连接超时的情况，这时候需要调整kafka的配置中的`advertised.host.name`，将其设置为我们为Vagrant的`private_network`

首先我这个kafka是在本机启动的，然后Vagrant是通过端口查询到的，然后kafka和zookeeper的端口是通过Vagrant映射来的,kafka是通过zookeeper来维护集群信息的，通过zookeeper找到Broker，然后尝试连接Broker。这个问题就是由于kafka发送的ip错误造成的，而`advertised.host.name`设置是强行设置kafka的反馈ip。
## Producer讲解
#### 进行配置，主要有序列化的配置

``` Java
       public ProducerConfig config(){
        Properties props = new Properties();
        props.put("metadata.broker.list", "192.168.33.10:9092,192.168.33.10:9091"); //Broker集群位置，
            props.put("serializer.class", "kafka.serializer.StringEncoder");//指定采用何种序列化的方式将消息传给Broker
            props.put("partitioner.class", "io.github.binglau.BingPartitioner");//指定发送分区的对应方式，如果不指定则随机发送到某个分区，这里可以对分区进行一个负载均衡的操作
            props.put("request.required.acks", "1");//指定消息是否确认发送成功

            ProducerConfig config = new ProducerConfig(props);
            return config;
        } 
```
#### 定义Producer对象

`Producer<String, String> producer = new Producer<String, String>(config);`
#### 发送消息

``` Java
public void singleModeProduce(ProducerConfig config) throws InterruptedException{
        Producer<String, String> producer = new Producer<String, String>(config);
        for(int i = 0; i < 10000; i++){
            producer.send(getMsg(i));
            System.out.println(i);
            Thread.sleep(1000);
        }
}

public void batchModeProduce(ProducerConfig config){
        List<KeyedMessage<String, String>> messages = new ArrayList<KeyedMessage<String, String>>(100);
        Producer<String, String> producer = new Producer<String, String>(config);
        for(int i = 0; i < 1000; i++){
                messages.add(getMsg(i));
                if (i % 100 == 0){
                        producer.send(messages);
                        messages.clear();
                }
        }
        producer.send(messages);
}

```
## HighLevelConsumer讲解

有时，我们消费Kafka的消息，并不关心偏移量，我们仅仅关心数据能被消费就行。High Level Consumer(高级消费者)提供了消费信息的方法而屏蔽了大量的底层细节。
### 设计一个高级消费者

了解使用高层次消费者的第一件事是，它可以（而且应该！）是一个多线程的应用。线程围绕在你的主题分区的数量，有一些非常具体的规则： 
- 如果你提供比在topic分区多的线程数量，一些线程将永远不会看到消息。 
- 如果你提供的分区比你拥有的线程多，线程将从多个分区接收数据。 
- 如果你每个线程上有多个分区，对于你以何种顺序收到消息是没有保证的。举个例子，你可能从分区10上获取5条消息和分区11上的6条消息，然后你可能一直从10上获取消息，即使11上也拥有数据。 
- 添加更多的进程线程将使kafka重新平衡，可能改变一个分区到线程的分配。
#### 配置

``` Java
private ConsumerConfig config(){
    Properties props = new Properties();
    props.put("group.id", groupId);
    props.put("consumer.id", consumerId);
    props.put("zookeeper.connect", Config.ZK_CONNECT);
    props.put("zookeeper.session.timeout.ms", "60000");
    props.put("zookeeper.sync.time.ms", "2000");
    return new ConsumerConfig(props);
}
```
#### 获取消息流

`Map<String, List<KafkaStream<byte[], byte[]>>> streams = connector.createMessageStreams(topicCountMap);
`
#### 创建线程处理消息

```
    private class BingStreamThread implements Runnable{
        private KafkaStream<byte[], byte[]> stream;

        public BingStreamThread(KafkaStream<byte[], byte[]> stream){
           this.stream = stream;
        }
        public void run() {
            ConsumerIterator<byte[], byte[]> streamIterator = stream.iterator();

            // 逐条处理消息
            while (streamIterator.hasNext()) {
                MessageAndMetadata<byte[], byte[]> message = streamIterator.next();
                String topic = message.topic();
                int partition = message.partition();
                long offset = message.offset();
                String key = new String(message.key());
                String msg = new String(message.message());
                // 在这里处理消息,这里仅简单的输出
                // 如果消息消费失败，可以将已上信息打印到日志中，活着发送到报警短信和邮件中，以便后续处理
                System.out.println("consumerId:" + consumerId
                        + ", thread : " + Thread.currentThread().getName()
                        + ", topic : " + topic
                        + ", partition : " + partition
                        + ", offset : " + offset
                        + " , key : " + key
                        + " , mess : " + msg);
            }
        }
    }

```
#### 消费消息

```

        // 为每个stream启动一个线程消费消息
        for (KafkaStream<byte[], byte[]> stream : streams.get(Config.TOPIC)) {
            BingStreamThread bingStreamThread = new BingStreamThread(stream);
            executor.submit(bingStreamThread);
        }

```
## SimpleConsumer讲解
### 为什么使用SimpleConsumer?

使用“SimpleConsumer”的主要原因是你想比使用“消费者分组”更好的控制分区消费。 
比如你想: 
    1.  多次读取消息 
    2.  在一个处理过程中只消费Partition其中的一部分消息 
    3.  添加事务管理机制以保证消息被处理且仅被处理一次 
### 使用SimpleConsumer有哪些弊端呢

这个SimpleConsumer确实需要很大的工作量： 
    1.  必须在程序中跟踪offset值. 
    2.  必须找出指定Topic(主题) Partition(分区)中的lead broker. 
    3.  必须处理broker的变动. 
### 使用SimpleConsumer的步骤
#### 找出Leader Broker

```
// 从所有活跃的broker中找出哪个是指定Topic（主题） Partition（分区）中的leader broker

        BrokerEndPoint leaderBroker = findLeader(Config.KAFKA_ADDRESS, Config.TOPIC, partition);


    /**
     * 找到指定分区的leader broker
     * @return 指定分区的leader broker
     */
    public BrokerEndPoint findLeader(String brokerHosts, String topic, int partition) {
        BrokerEndPoint leader = findPartitionMetadata(brokerHosts, topic, partition).leader();
        System.out.println(String.format("Leader tor topic %s, partition %d is %s:%d",
                topic, partition, leader.host(), leader.port()));
        return leader;
    }

    /**
     * 找到指定分区的元数据
     * @return 指定分区元数据
     */
    private PartitionMetadata findPartitionMetadata(String brokerHosts, String topic, int partition) {
        PartitionMetadata returnMetadata = null;
        for (String brokerHost : brokerHosts.split(",")) {
            SimpleConsumer consumer = null;
            try {
                String[] splits = brokerHost.split(":");
                consumer = new SimpleConsumer(
                        splits[0], Integer.valueOf(splits[1]), 100000, 64 * 1024, "leaderLookup"
                );
                List<String> topics = Collections.singletonList(topic);
                TopicMetadataRequest request = new TopicMetadataRequest(topics);
                TopicMetadataResponse response = consumer.send(request);
                List<TopicMetadata> topicMetadatas = response.topicsMetadata();
                for (TopicMetadata topicMetadata : topicMetadatas) {
                    for (PartitionMetadata partitionMetadata : topicMetadata.partitionsMetadata()) {
                        if (partitionMetadata.partitionId() == partition) {
                            returnMetadata = partitionMetadata;
                            break;
                        }
                    }
                }
            } catch (Exception e) {
                System.out.println("Error communicating with Broker [" + brokerHost + "] to find Leader for [" + topic
                        + ", " + partition + "] Reason: " + e);
            } finally {
                if (consumer != null) {
                    consumer.close();
                }
            }
        }
        return returnMetadata;
    }
```
#### 构造消费者

```
        // 构造消费
        SimpleConsumer simpleConsumer = new SimpleConsumer(
                leaderBroker.host(), leaderBroker.port(), 20000, 10000, "bingSimpleConsumer");
        long startOffset = 1;
        int fetchSize = 1000;

```
#### 构造请求与处理请求

```
        while (true) {
            long offset = startOffset;
            // 构造请求
            FetchRequest request = new FetchRequestBuilder()
                    .addFetch(Config.TOPIC, 0, startOffset, fetchSize).build();

            // 发送请求获取数据
            FetchResponse response = simpleConsumer.fetch(request);

            ByteBufferMessageSet messageSet = response.messageSet(Config.TOPIC, partition);
            for (MessageAndOffset messageAndOffset : messageSet) {
                Message msg = messageAndOffset.message();
                ByteBuffer payload = msg.payload();
                byte[] bytes = new byte[payload.limit()];
                payload.get(bytes);
                String message = new String(bytes);
                offset = messageAndOffset.offset();
                System.out.println("partition : " + 3 + ", offset : " + offset + "  msg : " + message);
            }
            // 继续消费下一批
            startOffset = offset + 1;
        }
```
## 参考资料

[Kafka教程](http://orchome.com/)
## 代码示例

[代码示例](https://github.com/BingLau7/kafkademo)
