---
title: Akka 基础功能
date: 2017-09-20 16:54:31
tags:
    - akka
    - scala
categories: 分布式
---
### 生命周期

![生命周期](https://github.com/BingLau7/blog/blob/master/images/blog_25/actor_lifecycle.png?raw=true)

<!-- more -->

```scala
package io.binglau.scala.akka.demo

import java.util.concurrent.TimeUnit

import akka.actor.{Actor, ActorSystem, OneForOneStrategy, Props, SupervisorStrategy}
import akka.event.Logging
import akka.pattern.{Backoff, BackoffSupervisor}

import scala.concurrent.duration.Duration

class LifeCycle extends Actor{
  val log = Logging(context.system, this)

  // Actor 对象被创建后启动前该方法会被调用
  // Actor 对象是通过异步方式创建的
  override def preStart(): Unit = {
    log.info("preStart")
  }

  // Actor 对象停止运行时，该方法会被调用
  // 当 ActorSystem 或 ActorContext 对象（Actor 对象中会
  // 含有一个 ActorContext 对象，可通过 context 方法获得）中的
  // stop(ActorRef) 方法被调用是,Actor对象会以一部方式停止运行.
  override def postStop(): Unit =  {
    log.info("postStop")
  }

  // 通过失效监督策略可以重启失效的子 Actor 对象。在
  // 执行重启操作的过程中，可以在重启 Actor 对象前调用该方法。
  // 该方法默认的实现代码会处理 Actor 对象停止运行前使用的资源，
  // 处理 Actor 对象的所有子对象。执行完清理操作后，postStop() 方法会自动被调用。
  // 通常不必重写
  override def preRestart(reason: Throwable, message: Option[Any]): Unit = {
    log.info(s"preRestart e: ${reason.getMessage}, message: ${message.toString}")
    super.preRestart(reason, message)
  }

  // Actor 对象重启后回调用这个方法，使 Actor 对象能够在失效
  // 并重启后被初始化。该方法默认的执行代码会调用 preStart() 方法。
  // 通常不必重写
  override def postRestart(reason: Throwable): Unit = {
    log.info(s"postRestart e: ${reason.getMessage}")
    super.postRestart(reason)
  }

  override def receive: Receive = {
    case "Restart" =>
      log.info("Restart")
      throw new NullPointerException
    case _ => log.info("receive something")
  }
}


object LifeCycleDemo {
  def main(args: Array[String]): Unit = {
    val system = ActorSystem("demo")
    val lifeCycle = Props(classOf[LifeCycle])

    val supervisor = BackoffSupervisor.props(
      Backoff.onStop(
        lifeCycle,
        childName = "myDemo",
        minBackoff = Duration.create(3, TimeUnit.SECONDS),
        maxBackoff = Duration.create(30, TimeUnit.SECONDS),
        randomFactor = 0.2 // adds 20% "noise" to vary the intervals slightly
      ).withSupervisorStrategy(
        OneForOneStrategy() {
          case e: NullPointerException => SupervisorStrategy.Restart
          case _ => SupervisorStrategy.Escalate
        }
      ))


    val actor = system.actorOf(lifeCycle, "lifeCycle")
    system.actorOf(supervisor, "lifeCycleSupervisor")
    actor ! "Test"
    actor ! "Restart"

    Thread.sleep(1000)
    actor ! "Test"

    system.terminate()
  }
}

/**
[INFO] [09/12/2017 00:36:53.376] [demo-akka.actor.default-dispatcher-3] [akka://demo/user/lifeCycle] preStart
[INFO] [09/12/2017 00:36:53.376] [demo-akka.actor.default-dispatcher-3] [akka://demo/user/lifeCycle] receive something
[INFO] [09/12/2017 00:36:53.376] [demo-akka.actor.default-dispatcher-3] [akka://demo/user/lifeCycle] Restart
[INFO] [09/12/2017 00:36:53.377] [demo-akka.actor.default-dispatcher-2] [akka://demo/user/lifeCycleSupervisor/myDemo] preStart
[ERROR] [09/12/2017 00:36:53.398] [demo-akka.actor.default-dispatcher-3] [akka://demo/user/lifeCycle] null
java.lang.NullPointerException
	at io.binglau.scala.akka.demo.LifeCycle$$anonfun$receive$1.applyOrElse(LifeCycle.scala:49)
	at akka.actor.Actor.aroundReceive(Actor.scala:513)
	at akka.actor.Actor.aroundReceive$(Actor.scala:511)
	at io.binglau.scala.akka.demo.LifeCycle.aroundReceive(LifeCycle.scala:11)
	at akka.actor.ActorCell.receiveMessage(ActorCell.scala:527)
	at akka.actor.ActorCell.invoke(ActorCell.scala:496)
	at akka.dispatch.Mailbox.processMailbox(Mailbox.scala:257)
	at akka.dispatch.Mailbox.run(Mailbox.scala:224)
	at akka.dispatch.Mailbox.exec(Mailbox.scala:234)
	at akka.dispatch.forkjoin.ForkJoinTask.doExec(ForkJoinTask.java:260)
	at akka.dispatch.forkjoin.ForkJoinPool$WorkQueue.runTask(ForkJoinPool.java:1339)
	at akka.dispatch.forkjoin.ForkJoinPool.runWorker(ForkJoinPool.java:1979)
	at akka.dispatch.forkjoin.ForkJoinWorkerThread.run(ForkJoinWorkerThread.java:107)

[INFO] [09/12/2017 00:36:53.410] [demo-akka.actor.default-dispatcher-5] [akka://demo/user/lifeCycle] preRestart e: null, message: Some(Restart)
[INFO] [09/12/2017 00:36:53.411] [demo-akka.actor.default-dispatcher-5] [akka://demo/user/lifeCycle] postStop
[INFO] [09/12/2017 00:36:53.413] [demo-akka.actor.default-dispatcher-5] [akka://demo/user/lifeCycle] postRestart e: null
[INFO] [09/12/2017 00:36:53.413] [demo-akka.actor.default-dispatcher-5] [akka://demo/user/lifeCycle] preStart
[INFO] [09/12/2017 00:36:54.380] [demo-akka.actor.default-dispatcher-4] [akka://demo/user/lifeCycle] receive something
[INFO] [09/12/2017 00:36:54.419] [demo-akka.actor.default-dispatcher-5] [akka://demo/user/lifeCycle] postStop
[INFO] [09/12/2017 00:36:54.426] [demo-akka.actor.default-dispatcher-2] [akka://demo/user/lifeCycleSupervisor/myDemo] postStop

Process finished with exit code 0
**/
```

### 更改状态

```scala
package io.binglau.scala.akka.demo

import akka.actor.{Actor, ActorSystem, Props}
import akka.event.Logging

// become(...) 使用该函数可以通过指定偏函数设置 Actor 对象的当前行为。
// 该函数有两个参数，第二个参数为 discardOld, 默认为 true。当该参数为 true 时，
// 在设置当前行为前，Actor 对象的上个行为会被丢弃。当该参数为 false 时，
// Actor 对象的上一个行为仍旧被保存在堆栈中，当前设置的 Actor 对象行为也会被推入
// 堆栈，并位于上一个行为条目的上方

// unbecome 在上一个行为存在的情况下，使用该函数可以使 Actor 对象从当前行为
// 切换到上一个行为

class StatusSwap extends Actor{
  import context._
  val log = Logging(context.system, this)

  def angry: Receive = {
    case "foo" => log.info("I am already angry?")
    case "bar" =>
      become(happy)
      log.info("angry become happy")
    case "return" =>
      unbecome()
      log.info("angry unbecome")
  }

  def happy: Receive = {
    case "bar" => log.info("I am already happy :-)")
    case "foo" =>
      become(angry, false)
//      become(angry)
      log.info("happy become angry")
    case "return" =>
      unbecome()
      log.info("happy unbecome")
  }

  override def receive: Receive = {
    case "foo" =>
      log.info("become angry")
      become(angry)
    case "bar" =>
      log.info("become happy")
      become(happy)
  }
}

// The App trait can be used to quickly turn objects into executable programs.
// Here, object Main inherits the main method of App.
// args returns the current command line arguments as an array.
object StatusDemo extends App {
//  Console.println("Hello World: " + (args mkString ", "))
  val system = ActorSystem("demo")
  val statusSwap = system.actorOf(Props(classOf[StatusSwap]), "statusSwap")

  statusSwap ! "bar" // happy
  statusSwap ! "bar"
  statusSwap ! "foo" // angry
  statusSwap ! "foo"
  statusSwap ! "return" // happy
  statusSwap ! "bar"

  system.terminate()
}

/**
[INFO] [09/13/2017 23:23:03.776] [demo-akka.actor.default-dispatcher-5] [akka://demo/user/statusSwap] become happy
[INFO] [09/13/2017 23:23:03.777] [demo-akka.actor.default-dispatcher-5] [akka://demo/user/statusSwap] I am already happy :-)
[INFO] [09/13/2017 23:23:03.778] [demo-akka.actor.default-dispatcher-5] [akka://demo/user/statusSwap] happy become angry
[INFO] [09/13/2017 23:23:03.778] [demo-akka.actor.default-dispatcher-5] [akka://demo/user/statusSwap] I am already angry?
[INFO] [09/13/2017 23:23:03.778] [demo-akka.actor.default-dispatcher-5] [akka://demo/user/statusSwap] angry unbecome
[INFO] [09/13/2017 23:23:03.778] [demo-akka.actor.default-dispatcher-5] [akka://demo/user/statusSwap] I am already happy :-)
**/
```

### 监督

任何创建了子 Actor 对象的 Actor 对象，都会自动变为其子 Actor 对象的监督者。如果子 Actor 对象崩溃了（例如抛出了异常），那么它的父 Actor 对象就必须在执行下列操作之间做出选择：

-  使子对象继续运行
-  重启子对象
-  停止子对象
-  使失效情况升级（将控制权移交给祖父对象）

```scala
package io.binglau.scala.akka.demo

import akka.actor.SupervisorStrategy.{Escalate, Restart, Resume, Stop}
import akka.actor.{Actor, ActorRef, ActorSystem, OneForOneStrategy, Props}
import akka.event.Logging

import scala.concurrent.duration.DurationInt

class SupervisorChild extends Actor {
  val log = Logging(context.system, this)
  override def receive: Receive = {
    case "null" => throw new NullPointerException
    case "arith" => throw new ArithmeticException
    case "illegal" => throw new IllegalArgumentException
    case "unsupport" => throw new UnsupportedOperationException
    case "exception" => throw new Exception
    case _ => log.info("unknow")
  }
}

class Supervisor extends Actor {
  import context._

  val log = Logging(context.system, this)
  var child: ActorRef = Actor.noSender

  // 实例化一个 SupervisorStrategy 子类 (AllForOneStrategy / OneForOneStrategy)。
  // 通常只是声明 OneForOneStrategy 子类，因为该策略仅会应用于崩溃的子 Actor 对象。
  // 使用 AllForOneStrategy 子类的情况比较少，因为它会对所有子 Actor 对象产生 Decider 偏
  // 函数效果（将失效情况与处理手段对应起来），而不是仅对崩溃的子 Actor 对象生效
  override val supervisorStrategy =
    OneForOneStrategy(
      maxNrOfRetries = 5, // 运行 Actor 对象崩溃的次数, -1 为次数不限
      withinTimeRange = 1 minute // 放弃 Actor 对象前重新启动该对象的时限
    ) { // Decider 偏函数
      case _: NullPointerException => Restart // 重启
      case _: ArithmeticException => Resume // 保持原状态继续运行
      case _: IllegalArgumentException => Stop // 停止
      case _: UnsupportedOperationException => Stop // 停止
      case _: Exception => Escalate // 失效，处理权移交给祖父对象
    }

  def input: Receive = {
    case msg: String => child ! msg
    case _ => log.info("unknow")
  }

  override def receive: Receive = {
    case "A" =>
      child = context.actorOf(Props[SupervisorChild], "childA")
      become(input)
    case _ =>
      throw new NullPointerException
  }
}

object SupervisorDemo extends App {
  val system = ActorSystem("demo")
  val supervisor = system.actorOf(Props(classOf[Supervisor]), "supervisor")

  supervisor ! "A"
  supervisor ! "abc"
  supervisor ! "arith"
  supervisor ! "null"
  Thread.sleep(1000)
  supervisor ! "abc"
  supervisor ! "illegal"
  supervisor ! "abc"
}

/**
[INFO] [09/14/2017 00:20:28.573] [demo-akka.actor.default-dispatcher-3] [akka://demo/user/supervisor/childA] unknow
[WARN] [09/14/2017 00:20:28.581] [demo-akka.actor.default-dispatcher-4] [akka://demo/user/supervisor/childA] null
[ERROR] [09/14/2017 00:20:28.584] [demo-akka.actor.default-dispatcher-3] [akka://demo/user/supervisor/childA] null
java.lang.NullPointerException
	at io.binglau.scala.akka.demo.SupervisorChild$$anonfun$receive$1.applyOrElse(Supervisor.scala:12)
	at akka.actor.Actor.aroundReceive(Actor.scala:513)
	at akka.actor.Actor.aroundReceive$(Actor.scala:511)
	at io.binglau.scala.akka.demo.SupervisorChild.aroundReceive(Supervisor.scala:9)
	at akka.actor.ActorCell.receiveMessage(ActorCell.scala:527)
	at akka.actor.ActorCell.invoke(ActorCell.scala:496)
	at akka.dispatch.Mailbox.processMailbox(Mailbox.scala:257)
	at akka.dispatch.Mailbox.run(Mailbox.scala:224)
	at akka.dispatch.Mailbox.exec(Mailbox.scala:234)
	at akka.dispatch.forkjoin.ForkJoinTask.doExec(ForkJoinTask.java:260)
	at akka.dispatch.forkjoin.ForkJoinPool$WorkQueue.runTask(ForkJoinPool.java:1339)
	at akka.dispatch.forkjoin.ForkJoinPool.runWorker(ForkJoinPool.java:1979)
	at akka.dispatch.forkjoin.ForkJoinWorkerThread.run(ForkJoinWorkerThread.java:107)

[INFO] [09/14/2017 00:20:29.572] [demo-akka.actor.default-dispatcher-4] [akka://demo/user/supervisor/childA] unknow
[ERROR] [09/14/2017 00:20:29.573] [demo-akka.actor.default-dispatcher-3] [akka://demo/user/supervisor/childA] null
java.lang.IllegalArgumentException
	at io.binglau.scala.akka.demo.SupervisorChild$$anonfun$receive$1.applyOrElse(Supervisor.scala:14)
	at akka.actor.Actor.aroundReceive(Actor.scala:513)
	at akka.actor.Actor.aroundReceive$(Actor.scala:511)
	at io.binglau.scala.akka.demo.SupervisorChild.aroundReceive(Supervisor.scala:9)
	at akka.actor.ActorCell.receiveMessage(ActorCell.scala:527)
	at akka.actor.ActorCell.invoke(ActorCell.scala:496)
	at akka.dispatch.Mailbox.processMailbox(Mailbox.scala:257)
	at akka.dispatch.Mailbox.run(Mailbox.scala:224)
	at akka.dispatch.Mailbox.exec(Mailbox.scala:234)
	at akka.dispatch.forkjoin.ForkJoinTask.doExec(ForkJoinTask.java:260)
	at akka.dispatch.forkjoin.ForkJoinPool$WorkQueue.runTask(ForkJoinPool.java:1339)
	at akka.dispatch.forkjoin.ForkJoinPool.runWorker(ForkJoinPool.java:1979)
	at akka.dispatch.forkjoin.ForkJoinWorkerThread.run(ForkJoinWorkerThread.java:107)

[INFO] [09/14/2017 00:20:29.579] [demo-akka.actor.default-dispatcher-3] [akka://demo/user/supervisor/childA] Message [java.lang.String] from Actor[akka://demo/user/supervisor#347975802] to Actor[akka://demo/user/supervisor/childA#794249311] was not delivered. [1] dead letters encountered. This logging can be turned off or adjusted with configuration settings 'akka.log-dead-letters' and 'akka.log-dead-letters-during-shutdown'.
[INFO] [09/14/2017 00:20:53.607] [Thread-0] [CoordinatedShutdown(akka://demo)] Starting coordinated shutdown from JVM shutdown hook
**/
```

### 层级结构

当 Actor 系统被创建时，有几个 Actor 对象会随着它一起被创建。其中包括：

-  root 守护对象
-  user 守护对象（应用程序创建的 Actor 对象上方）
-  system 守护对象

```scala
// will look up this absolute path
context.actorSelection("/user/serviceA/aggregator")
// will look up sibling beneath same supervisor
context.actorSelection("../joe")
// remote
context.actorSelection("akka.tcp://app@otherhost:1234/user/serviceB")
```

### 订阅发布

三个分类器：

-  LookupClassification：通过匹配指定的事件类型，支持简单的查询操作。

   ```scala
     // must define a full order over the subscribers, expressed as expected from
     // `java.lang.Comparable.compare`  
     override protected def compareSubscribers(a: Subscriber, b: Subscriber): Int =
       a.compareTo(b)
   ```

-  SubchannelClassification：通过匹配事件的类型和子类型，支持子通道层次结构

-  ScanningClassification：在某个事件落入两个或多个通道，并且需要将该事件发布给所有合法订阅者，因而需要扫描 EventBus 类的全部分类时，应混入该特征。

   ```scala
     // is needed for determining matching classifiers and storing them in an
     // ordered collection
     override protected def compareClassifiers(a: Classifier, b: Classifier): Int =
       if (a < b) -1 else if (a == b) 0 else 1

     // is needed for storing subscribers in an ordered collection  
     override protected def compareSubscribers(a: Subscriber, b: Subscriber): Int =
       a.compareTo(b)

     // determines whether a given classifier shall match a given event; it is invoked
     // for each subscription for all received events, hence the name of the classifier  
     override protected def matches(classifier: Classifier, event: Event): Boolean =
       event.length <= classifier
   ```

```scala
package io.binglau.scala.akka.demo

import akka.actor.{ActorRef, ActorSystem, Props}
import akka.event.{EventBus, SubchannelClassification}
import akka.util.Subclassification

final case class MsgEnvelope(topic: String, payload: Any)

class StartsWithSubClassification extends Subclassification[String] {
  override def isEqual(x: String, y: String): Boolean = x == y

  override def isSubclass(x: String, y: String): Boolean = x.startsWith(y)
}

class SubChannel extends EventBus with SubchannelClassification {
  // 需要处理的订阅事件
  override type Event = MsgEnvelope
  // 分类器
  override type Classifier = String
  // 发布器
  override type Subscriber = ActorRef

  // 提供两种方法 isEqual 和 isSubClass，将分类器作为一个参数 x 与 classify 返回的值作为另一个参数传入,
  // 两个方法返回任意 true 则可进入
  override protected implicit def subclassification: Subclassification[Classifier] = new StartsWithSubClassification

  // 订阅事件哪部分作为分类信息
  override protected def classify(event: Event): Classifier = event.topic

  // 分类之后的发布操作
  override protected def publish(event: Event, subscriber: Subscriber): Unit = {
    subscriber ! event.payload
  }
}

object SubchannelDemo {
  def main(args: Array[String]): Unit = {
    val system = ActorSystem("Test")
    val print = system.actorOf(Props(classOf[Print]), "print")

    val subchannelBus = new SubChannel
    // 设置发布器和分类器
    subchannelBus.subscribe(print, "abc")
    subchannelBus.publish(MsgEnvelope("xyzabc", "x"))
    subchannelBus.publish(MsgEnvelope("bcdef", "b"))
    subchannelBus.publish(MsgEnvelope("abc", "c"))
    subchannelBus.publish(MsgEnvelope("abcdef", "d"))
  }
}

/**
[INFO] [09/14/2017 23:42:47.268] [Test-akka.actor.default-dispatcher-5] [akka://Test/user/print] receive c
[INFO] [09/14/2017 23:42:47.269] [Test-akka.actor.default-dispatcher-5] [akka://Test/user/print] receive d
**/
```

### Dispatchers

>  添加配置在 /src/main/resources 中添加一份 application.conf

>  An Akka `MessageDispatcher` is what makes Akka Actors “tick”, it is the engine of the machine so to speak. All `MessageDispatcher` implementations are also an `ExecutionContext`, which means that they can be used to execute arbitrary code, for instance [Futures](http://doc.akka.io/docs/akka/current/scala/futures.html).

#### 通过下面代码可以得到一个配置的 Dispatcher

```scala
// for use with Futures, Scheduler, etc.
implicit val executionContext = system.dispatchers.lookup("my-dispatcher")
```

#### 设置 Dispatcher

```properties
my-dispatcher {
  # Dispatcher is the name of the event-based dispatcher
  type = Dispatcher
  # What kind of ExecutionService to use
  executor = "fork-join-executor"
  # Configuration for the fork join pool
  fork-join-executor {
    # Min number of threads to cap factor-based parallelism number to
    parallelism-min = 2
    # Parallelism (threads) ... ceil(available processors * factor)
    parallelism-factor = 2.0
    # Max number of threads to cap factor-based parallelism number to
    parallelism-max = 10
  }
  # Throughput defines the maximum number of messages to be
  # processed per actor before the thread jumps to the next actor.
  # Set to 1 for as fair as possible.
  throughput = 100
}
```

配置Actor使用一个特定的disptacher:

```scala
import akka.actor.Props
val myActor = context.actorOf(Props[MyActor], "myactor")


akka.actor.deployment {
  /myactor {
    dispatcher = my-dispatcher
  }
}
```

或者在代码中设置

```scala
import akka.actor.Props
val myActor =
  context.actorOf(Props[MyActor].withDispatcher("my-dispatcher"), "myactor1")
```

#### type

-  **Dispatcher**

   This is an event-based dispatcher that binds a set of Actors to a thread pool. It is the default dispatcher used if one is not specified.

   -  可共享性: 无限制
   -  邮箱: 任何一种类型，为每一个Actor创建一个
   -  使用场景: 默认派发器，Bulkheading
   -  底层使用: `java.util.concurrent.ExecutorService`
      可以指定“executor”使用“fork-join-executor”, “thread-pool-executor” 或者 the FQCN(类名的全称) of an akka.dispatcher.ExecutorServiceConfigurator

-  **PinnedDispatcher**

   This dispatcher dedicates a unique thread for each actor using it; i.e. each actor will have its own thread pool with only one thread in the pool.

   -  可共享性: 无
   -  邮箱: 任何一种类型，为每个Actor创建一个
   -  使用场景: Bulkheading
   -  底层使用: 任何 akka.dispatch.ThreadPoolExecutorConfigurator
      缺省为一个 “thread-pool-executor”

-  **CallingThreadDispatcher**

   This dispatcher runs invocations on the current thread only. This dispatcher does not create any new threads, but it can be used from different threads concurrently for the same actor. See [CallingThreadDispatcher](http://doc.akka.io/docs/akka/current/scala/testing.html#callingthreaddispatcher) for details and restrictions.

   -  可共享性: 无限制
   -  邮箱: 任何一种类型，每Actor每线程创建一个（需要时）
   -  使用场景: 测试
   -  底层使用: 调用的线程 (duh)

### 确保送达机制

`at-least-once-delivery`

>  To send messages with at-least-once delivery semantics to destinations you can mix-in `AtLeastOnceDelivery` trait to your `PersistentActor` on the sending side. It takes care of re-sending messages when they have not been confirmed within a configurable timeout.
>
>  结合持久化保证一定时间被确认（发送端）
>
>  The state of the sending actor, including which messages have been sent that have not been confirmed by the recipient must be persistent so that it can survive a crash of the sending actor or JVM. The `AtLeastOnceDelivery` trait does not persist anything by itself. It is your responsibility to persist the intent that a message is sent and that a confirmation has been received.
>
>  AtLeastOnceDelivery 不会持久化任何东西，你需要去持久化试图发送的消息并处理确认信息

```scala
package io.binglau.scala.akka.demo

import akka.actor.{Actor, ActorSelection}
import akka.event.Logging
import akka.persistence.{AtLeastOnceDelivery, PersistentActor}

case class Msg(deliveryId: Long, s: String)
case class Confirm(deliveryId: Long)

// sealed
// 其修饰的trait，class只能在当前文件里面被继承
// 用sealed修饰这样做的目的是告诉scala编译器在检查模式匹配的时候，
//   让scala知道这些case的所有情况，scala就能够在编译的时候进行检查，
//   看你写的代码是否有没有漏掉什么没case到，减少编程的错误。
sealed trait Evt
case class MsgSent(s: String) extends Evt
case class MsgConfirmed(deliveryId: Long) extends Evt

class MyPersistentActor(destination: ActorSelection)
  extends PersistentActor with AtLeastOnceDelivery {

  // 在任何情况下，从主题 / 队列持久性 Actor 对象接收消息的持久性视图 Actor 实例，
  // 都会将它的  persistenceId 作为相应主题 / 队列持久性 Actor 对象标识符
  override def persistenceId: String = "persistence-id"

  // 通过 receiveCommand 来接受新消息
  override def receiveCommand: Receive = {
    // To send messages to the destination path, use the **deliver** method after
    // you have persisted the intent to send the message.
    case s: String           => persist(MsgSent(s))(updateState)
    // The destination actor must send back a confirmation message.
    // When the sending actor receives this confirmation message you should
    // persist the fact that the message was delivered successfully
    // and then call the confirmDelivery method.
    case Confirm(deliveryId) => persist(MsgConfirmed(deliveryId))(updateState)
  }

  // 发送确认回执
  override def receiveRecover: Receive = {
    case evt: Evt => updateState(evt)
  }

  def updateState(evt: Evt): Unit = evt match {
    // deliver自动产生一个deliveryId，这个deliveryId是发送方与接收方沟通的标志
    case MsgSent(s) =>
      deliver(destination)(deliveryId => Msg(deliveryId, s))

    case MsgConfirmed(deliveryId) => confirmDelivery(deliveryId)
  }
}

class MyDestination extends Actor {
  val log = Logging(context.system, this)
  def receive = {
    case Msg(deliveryId, s) =>
      log.info(s)
      sender() ! Confirm(deliveryId)
  }
}
```

>  也可参考 Stream 相关内容

### 应用Demo

#### 请求—回复模式

```scala
package io.binglau.scala.akka.demo

import akka.actor.{Actor, ActorRef, ActorSystem, Props}

case class Request(what: String)
case class Reply(what: String)
case class StartWith(server: ActorRef)

// 方式
// 1. 发送消息带上 server
// 2. 初始化带上 server
class Client extends Actor {
  def receive = {
    case StartWith(server) =>
      println("Client: is starting...")
      server ! Request("REQ-1")
    case Reply(what) =>
      println("Client: received response: " + what)
    case _ =>
      println("Client: received unexpected message")
  }
}

/**
  * 一个 Actor 对象可以通过下列方式获得其他 Actor 对象的地址
  * 1. 一个 Actor 对象创建了另一个 Actor 对象
  * 2. 一个 Actor 对象收到消息，包含其他 Actor 对象地址
  * 3. 有时 Actor 对象可以根据名称（selection）查询 Actor，但是这么做会带来不合适的定义和实现束缚
  */
class Server extends Actor {
  def receive = {
    case Request(what) =>
      println("Server: received request value: " + what)
      sender ! Reply("RESP-1 for " + what)
    case _ =>
      println("Server: received unexpected message")
  }
}


object RequestReplyDemo extends App {
  val system = ActorSystem("demo")
  val client = system.actorOf(Props[Client], "client")
  val server = system.actorOf(Props[Server], "server")
  client ! StartWith(server)

  println("RequestReply: is completed.")
  system.terminate()
}
```

#### 管道与过滤器

```scala
package io.binglau.scala.akka.demo

import akka.actor.{Actor, ActorRef, ActorSystem, Props}

case class ProcessIncomingOrder(orderInfo: Array[Byte])

class Authenticator(nextFilter: ActorRef) extends Actor {
  def receive = {
    case message: ProcessIncomingOrder =>
      val text = new String(message.orderInfo)
      println(s"Authenticator: processing $text")
      val orderText = text.replace("(certificate)", "")
      nextFilter ! ProcessIncomingOrder(orderText.toCharArray.map(_.toByte))
  }
}

class Decrypter(nextFilter: ActorRef) extends Actor {
  def receive = {
    case message: ProcessIncomingOrder =>
      val text = new String(message.orderInfo)
      println(s"Decrypter: processing $text")
      val orderText = text.replace("(encryption)", "")
      nextFilter ! ProcessIncomingOrder(orderText.toCharArray.map(_.toByte))
  }
}

class Deduplicator(nextFilter: ActorRef) extends Actor {
  val processedOrderIds = scala.collection.mutable.Set[String]()

  def orderIdFrom(orderText: String): String = {
    val orderIdIndex = orderText.indexOf("id='") + 4
    val orderIdLastIndex = orderText.indexOf("'", orderIdIndex)
    orderText.substring(orderIdIndex, orderIdLastIndex)
  }

  def receive = {
    case message: ProcessIncomingOrder =>
      val text = new String(message.orderInfo)
      println(s"Deduplicator: processing $text")
      val orderId = orderIdFrom(text)
      if (processedOrderIds.add(orderId)) {
        nextFilter ! message
      } else {
        println(s"Deduplicator: found duplicate order $orderId")
      }
  }
}

class OrderAcceptanceEndpoint(nextFilter: ActorRef) extends Actor {
  def receive = {
    case rawOrder: Array[Byte] =>
      val text = new String(rawOrder)
      println(s"OrderAcceptanceEndpoint: processing $text")
      nextFilter ! ProcessIncomingOrder(rawOrder)
  }
}

class OrderManagementSystem extends Actor {
  def receive = {
    case message: ProcessIncomingOrder =>
      val text = new String(message.orderInfo)
      println(s"OrderManagementSystem: processing unique order: $text")
  }
}

object PipesAndFilterDemo extends App{
  val system = ActorSystem("demo")
  val orderText = "(encryption)(certificate)<order id='123'>...</order>"
  val rawOrderBytes = orderText.toCharArray.map(_.toByte)

  // 链式过滤器
  val filter5 = system.actorOf(Props[OrderManagementSystem], "orderManagementSystem")
  val filter4 = system.actorOf(Props(classOf[Deduplicator], filter5), "deduplicator")
  val filter3 = system.actorOf(Props(classOf[Authenticator], filter4), "authenticator")
  val filter2 = system.actorOf(Props(classOf[Decrypter], filter3), "decrypter")
  val filter1 = system.actorOf(Props(classOf[OrderAcceptanceEndpoint], filter2), "orderAcceptanceEndpoint")

  filter1 ! rawOrderBytes
  filter1 ! rawOrderBytes

  println("PipesAndFilters: is completed.")

  system.terminate()
}
```

#### 消息有效期

```scala
package io.binglau.scala.akka.demo

import java.util.Date
import java.util.concurrent.TimeUnit

import akka.actor.{Actor, ActorRef, ActorSystem, Props}

import scala.concurrent.duration.Duration
import scala.util.Random
import scala.concurrent.ExecutionContext.Implicits.global


// 将过期判断操作信息加入消息中
trait ExpiringMessage {
  val occurredOn = System.currentTimeMillis()
  val timeToLive: Long

  def isExpired: Boolean = {
    val elapsed = System.currentTimeMillis() - occurredOn

    elapsed > timeToLive
  }
}

case class PlaceOrder(id: String, itemId: String, price: Double, timeToLive: Long) extends ExpiringMessage

// 模拟各种原因导致的消息传输协议延迟
class PurchaseRouter(purchaseAgent: ActorRef) extends Actor {
  val random = new Random((new Date()).getTime)

  def receive = {
    case message: Any =>
      val millis = random.nextInt(100) + 1
      println(s"PurchaseRouter: delaying delivery of $message for $millis milliseconds")
      val duration = Duration.create(millis, TimeUnit.MILLISECONDS)
      context.system.scheduler.scheduleOnce(duration, purchaseAgent, message)
  }
}

class PurchaseAgent extends Actor {
  def receive = {
    case placeOrder: PlaceOrder =>
      if (placeOrder.isExpired) {
        context.system.deadLetters ! placeOrder
        println(s"PurchaseAgent: delivered expired $placeOrder to dead letters")
      } else {
        println(s"PurchaseAgent: placing order for $placeOrder")
      }

    case message: Any =>
      println(s"PurchaseAgent: received unexpected: $message")
  }
}

object MessageExpirationDemo extends App{
  val system = ActorSystem("demo")
  val purchaseAgent = system.actorOf(Props[PurchaseAgent], "purchaseAgent")
  val purchaseRouter = system.actorOf(Props(classOf[PurchaseRouter], purchaseAgent), "purchaseRouter")

  purchaseRouter ! PlaceOrder("1", "11", 50.00, 1000)
  purchaseRouter ! PlaceOrder("2", "22", 250.00, 100)
  purchaseRouter ! PlaceOrder("3", "33", 32.95, 10)

  Thread.sleep(3000)

  println("MessageExpiration: is completed.")
  system.terminate()
}
```



### 参考文档

[](http://wiki.fnil.net/index.php?title=Clojure%E5%B9%B6%E5%8F%91)

[Akka 官方文档](http://doc.akka.io/docs/akka/current/scala/index-actors.html)
《响应式架构——消息模式 Actor 实现与 Scala、Akka 应用集成》
