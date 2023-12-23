---
title: 初学Kafka——Kafka配置
date: 2016-03-24 16:22:24
tags:
    - Kafka
    - 分布式与中间件
categories: 工具
---

### 系统环境

我使用的是vagrant配置的Debian8环境

<!--more-->

```
$  ~ uname -a
Linux debian-jessie 3.16.0-4-amd64 #1 SMP Debian 3.16.7-ckt20-1+deb8u4 (2016-02-29) x86_64 GNU/Linux
$  ~  free -m
             total       used       free     shared    buffers     cached
Mem:           494        155        338          1         10         88
-/+ buffers/cache:         57        437
Swap:          461          4        457
$  ~ df -h
文件系统        容量  已用  可用 已用% 挂载点
/dev/sda1       9.2G  2.7G  6.1G   30% /
udev             10M     0   10M    0% /dev
tmpfs            99M  4.4M   95M    5% /run
tmpfs           248M     0  248M    0% /dev/shm
tmpfs           5.0M     0  5.0M    0% /run/lock
tmpfs           248M     0  248M    0% /sys/fs/cgroup
tmpfs            50M     0   50M    0% /run/user/1000
```
### JDK以及Kafka环境配置

```
 91 # Java环境配置
 92 export JAVA_HOME="$HOME/.utils/jdk"
 93 export PATH="$JAVA_HOME/bin:$PATH"
 94 export CLASSPATH="$JAVA_HOME/lib:$PATH"
 95
 96 # Kafka环境配置
 97 export KAFKA_HOME="$HOME/.utils/kafka"
 98 export PATH="$KAFKA_HOME/bin:$PATH"
 99 export KAFKA_HEAP_OPTS="-Xmx256M -Xms128M" #与下面错误有关
```

这里可以看出，我将Java和kafka放入`/home/.utils`中

这里我们需要先看一下`kafka`的目录

```
$  kafka ls
bin  config  libs  LICENSE  logs  NOTICE  site-docs
```

进入bin

```
$  bin ls
connect-distributed.sh     kafka-consumer-groups.sh             kafka-reassign-partitions.sh   kafka-simple-consumer-shell.sh   zookeeper-server-start.sh
connect-standalone.sh      kafka-consumer-offset-checker.sh     kafka-replay-log-producer.sh   kafka-topics.sh                  zookeeper-server-stop.sh
kafka-acls.sh              kafka-consumer-perf-test.sh          kafka-replica-verification.sh  kafka-verifiable-consumer.sh     zookeeper-shell.sh
kafka-configs.sh           kafka-mirror-maker.sh                kafka-run-class.sh             kafka-verifiable-producer.sh
kafka-console-consumer.sh  kafka-preferred-replica-election.sh  kafka-server-start.sh          windows
kafka-console-producer.sh  kafka-producer-perf-test.sh          kafka-server-stop.sh           zookeeper-security-migration.sh
```

进入config

```
$  kafka cd config
$  config ls
connect-console-sink.properties    connect-file-sink.properties    connect-standalone.properties  producer.properties    tools-log4j.properties
connect-console-source.properties  connect-file-source.properties  consumer.properties            server.properties      zookeeper.properties
connect-distributed.properties     connect-log4j.properties        log4j.properties               test-log4j.properties
```
### 启动
#### 启动`zookeeper`

这里我们使用`kafka`自带的`zookeeper`
从上面的`bin`中我们可以找到`zookeeper-server-start.sh`用来启动`zookeeper`，配置使用`config`中的`zookeeper.properties`
进入`zookeeper.properties`可以看到

```
15 # the directory where the snapshot is stored.
16 dataDir=/data/kafka/zookeeper
17 # the port at which the clients will connect
18 clientPort=2181
19 # disable the per-ip limit on the number of connections since this is a non-production config
20 maxClientCnxns=0
```

我们将数据放入`/data/kafka/zookeeper`

启动命令
`zookeeper-server-start.sh ~/.utils/kafka/config/zookeeper.properties`
#### 启动kafka

`kafka`的配置文件我们同样选择`config`中的`server.properties`
其中我们只需要关注这几个配置

```
broker.id = 0
port = 9092
log.dirs = /data/kafka-logs # 非log，数据存储处
zookeeper.connect = localhost:2181
```

如果是不同副本前三个需要不一致

启动命令
`kafka-server-start.sh ~/.utils/kafka/config/server.properties`
这里我遇到一个错误:`There is insufficient memory for the Java Runtime Environment to continue`
其具体描述与解决方法如下:
[解决办法](http://stackoverflow.com/questions/21448907/kafka-8-and-memory-there-is-insufficient-memory-for-the-java-runtime-environme)

可以通过`jps -l`查看运行在jdk上的应用程序

```
$  ~ jps -l
6049 sun.tools.jps.Jps
5714 kafka.Kafka
5561 org.apache.zookeeper.server.quorum.QuorumPeerMain
```

可看到`zookeeper`与`kafka`成功运行
#### 创建Topic

`kafka-topics.sh --create --zookeeper localhost:2181 --topic topic_name --partition 3 --replication-factor 1`
`--partition` 指定分区数
`--replication-factor` 指定副本数

通过下面命令来查看创建的`topic`

```
$  ~ kafka-topics.sh --list --zookeeper localhost:2181
test1
```

这样`kafka`就算搭建起来了
