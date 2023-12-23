---
title: Redis与消息队列
date: 2016-04-21 16:19:11
tags:
    - 消息队列
    - redis
categories: 工具
---


## 背景知识
### Pub/Sub

Redis中使用[SUBSCRIBE](http://redis.io/commands/subscribe), [UNSUBSCRIBE](http://redis.io/commands/unsubscribe) 和 [PUBLISH](http://redis.io/commands/publish) 方法将可以实现生产者/消费者模式，所谓的生产者其实就是使用`PUBLISH`将message(这里需要说明，message只能是String，不过我们可以序列化一个对象为Json再进行推送)推送到指定的队列中，再由`SUBSCRIBE`监听到消息并取出。`UNSUBSCRIBE`是取消监听。

<!--more-->

具体来说我们可以打开两个redis-cli来进行交互

``` Python
127.0.0.1:6379> SUBSCRIBE first_channel second_channel
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "first_channel"
3) (integer) 1
1) "subscribe"
2) "second_channel"
3) (integer) 2
1) "message"
2) "first_channel"
3) "first_message"
1) "message"
2) "second_channel"
3) "second_message"
```

``` Python
127.0.0.1:6379[1]> PUBLISH first_channel first_message
(integer) 1
127.0.0.1:6379[1]> PUBLISH second_channel second_message
(integer) 1
127.0.0.1:6379[1]>
```

这里需要说明的是`SUBSCRIBE`可以同时监听多个队列，这里对我们实现消息队列中的`Topic`是至关重要的。
## 功能实现

这里我们使用`Python`实现所有的代码
### 订阅者

``` python

#!/usr/bin/env python
# coding=utf-8

import redis


class SubChannel(object):
    def __init__(self, topic):
        self.topic = topic
        self.redis_conn = redis.Redis(host="localhost", db=1)
        self.pubsub = self.redis_conn.pubsub()
        self.pubsub.subscribe("%s:channel" % topic)

    def run(self):
        for i in self.pubsub.listen():
            if i['type'] == 'message':
                print "Task get", i['data']

if __name__ == "__main__":
    SubChannel("Test").run()

```
### 发布者

``` Python
#!/usr/bin/env python
# coding=utf-8

import redis


class PubChannel(object):
    def __init__(self, topic):
        self.topic = topic
        self.redis_conn = redis.Redis(host="localhost", db=1)
        self.pubsub = self.redis_conn.pubsub()

    def push(self, message):
        self.redis_conn.publish("%s:channel" % self.topic, message)

```

然后我们打开一个Python交互器，我这里使用的是`bpython`

``` python
> bpython                   ~/code/Python/daily/redis-channel@BingLaudeMacBook-Pro.local
bpython version 0.14.2 on top of Python 2.7.10 /usr/bin/python
>>> from pub import PubChannel
>>> p = PubChannel("Test")
>>> p.push("test")
>>>
> bpython                   ~/code/Python/daily/redis-channel@BingLaudeMacBook-Pro.local
bpython version 0.14.2 on top of Python 2.7.10 /usr/bin/python
>>> from pub import PubChannel
>>> p = PubChannel("Test")
>>> p.push("test")
>>> p = PubChannel("Test1")
>>> p.push("test")

# 结果
Task get test
```

这里我们可以设置不同的Topic来实现推送不同类型的消息给不同的订阅者
### 持久化

由于Redis这种队列不能进行持久化，所以我们需要借助其他的存储来实现持久化，如我们可以使用`Mysql`来记录推送的消息，然后在队列消息消耗完成之后将其值置为已完成的状态。

对于Mysql里面的数据，我们可以设置一个消耗时间，如果在某固定时间间隔内消息没有被消耗，我们可以采取相应的操作，如报警，重发等等。
## 最后

Redis实现消息队列还可以使用List来实现， 这种方法主要是依赖`BLPOP`(删除和获取列表中的第一个元素，或阻塞直到有可用) `BRPOP`(删除和获取列表中的最后一个元素，或阻塞直到有可用)这两个命令来实现。下面这篇文章很好的讲解了如何用这两个命令来实现消息队列，并附有优先级的实现方法。

[用redis实现支持优先级的消息队列](http://www.cnblogs.com/laozhbook/p/redis_queue.html)
