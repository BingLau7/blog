---
title: Akka 实例
date: 2017-09-21 16:55:04
tags:
    - akka
    - scala
categories: 分布式
---

### 动态路由器

![动态路由器](https://github.com/BingLau7/blog/blob/master/images/blog_26/%E5%8A%A8%E6%80%81%E8%B7%AF%E7%94%B1%E5%99%A8.png?raw=true)

动态路由器会使用规则，这些规则具有一定的复杂性。

<!-- more -->

要接收来自动态路由器的消息，Actor 对象必须注册它感兴趣的消息，通过最基础的注册流程，Actor 对象可以告诉路由器，它对哪种消息感兴趣。也可以通过更精细的规则，要求动态路由器执行多层次的查询操作，以便为 Actor 对象传输指定的消息。

```scala
// TypedMessageInterestRouter.scala

package io.binglau.scala.akka.demo.dynamic_router

import reflect.runtime.currentMirror
import akka.actor.{Actor, ActorRef}
import scala.collection._

case class InterestedIn(messageType: String)
case class NoLongerInterestedIn(messageType: String)

case class TypeAMessage(description: String)
case class TypeBMessage(description: String)
case class TypeCMessage(description: String)
case class TypeDMessage(description: String)

// 动态路由操作
class TypedMessageInterestRouter(dunnoInterested: ActorRef,
                                 canStartAfterRegistered: Int,
                                 canCompleteAfterUnregistered: Int) extends Actor {
  // 主要接收者
  val interestRegistry: mutable.Map[String, ActorRef] = mutable.Map[String, ActorRef]()
  // 次要接收者
  val secondaryInterestRegistry: mutable.Map[String, ActorRef] = mutable.Map[String, ActorRef]()

  override def receive: Receive = {
    case interestedIn: InterestedIn =>
      registerInterest(interestedIn)
    // 主要接受者由次要接收者取代(如果有)
    case noLongerInterestedIn: NoLongerInterestedIn =>
      unregisterInterest(noLongerInterestedIn)
    case message: Any =>
      sendFor(message)
  }

  // 注册即加入 interestRegistry Map 中
  def registerInterest(interestedIn: InterestedIn): Unit = {
    val messageType = typeOfMessage(interestedIn.messageType)
    if (!interestRegistry.contains(messageType)) {
      interestRegistry(messageType) = sender
    } else {
      secondaryInterestRegistry(messageType) = sender
    }

    if (interestRegistry.size + secondaryInterestRegistry.size >= canStartAfterRegistered) {
      println("registry exceed limit")
      context.system.terminate()
    }
  }

  // 查看消息是否注册及注册的 sender
  def sendFor(message: Any): Unit = {
    val messageType = typeOfMessage(currentMirror.reflect(message).symbol.toString)

    if (interestRegistry.contains(messageType)) {
      interestRegistry(messageType) forward message
    } else {
      dunnoInterested ! message
    }
  }

  def typeOfMessage(rawMessageType: String): String = {
    rawMessageType.replace('$', ' ').replace('.', ' ').split(' ').last.trim
  }

  var unregisterCount: Int = 0

  // 取消注册
  def unregisterInterest(noLongerInterestedIn: NoLongerInterestedIn): Unit = {
    val messageType = typeOfMessage(noLongerInterestedIn.messageType)

    if (interestRegistry.contains(messageType)) {
      val wasInterested = interestRegistry(messageType)

      if (wasInterested.compareTo(sender) == 0) {
        if (secondaryInterestRegistry.contains(messageType)) {
          val nowInterested = secondaryInterestRegistry.remove(messageType)

          interestRegistry(messageType) = nowInterested.get
        } else {
          interestRegistry.remove(messageType)
        }

        unregisterCount = unregisterCount + 1
        if (unregisterCount >= this.canCompleteAfterUnregistered) {
          println("unregister count > can CompleteAfterUnregistered")
          context.system.terminate()
        }
      }
    }
  }
}

```



```scala
// 各种业务 sender

package io.binglau.scala.akka.demo.dynamic_router

import akka.actor.{Actor, ActorRef}

class TypeAInterested(interestRouter: ActorRef) extends Actor {
  interestRouter ! InterestedIn(TypeAMessage.getClass.getName)

  override def receive: Receive = {
    case message: TypeAMessage =>
      println(s"TypeAInterested: received: $message")
    case message: Any =>
      println(s"TypeAInterested: received unexpected message: $message")
  }
}

class TypeBInterested(interestRouter: ActorRef) extends Actor {
  interestRouter ! InterestedIn(TypeBMessage.getClass.getName)

  override def receive: Receive = {
    case message: TypeBMessage =>
      println(s"TypeBInterested: received: $message")
    case message: Any =>
      println(s"TypeBInterested: received unexpected message: $message")
  }
}

class TypeCInterested(interestRouter: ActorRef) extends Actor {
  interestRouter ! InterestedIn(TypeCMessage.getClass.getName)

  override def receive: Receive = {
    case message: TypeCMessage =>
      println(s"TypeCInterested: received: $message")
      interestRouter ! NoLongerInterestedIn(TypeCMessage.getClass.getName)
    case message: Any =>
      println(s"TypeCInterested: received unexpected message: $message")
  }
}

class TypeCAlsoInterested(interestRouter: ActorRef) extends Actor {
  interestRouter ! InterestedIn(TypeCMessage.getClass.getName)

  def receive = {
    case message: TypeCMessage =>
      println(s"TypeCAlsoInterested: received: $message")

      interestRouter ! NoLongerInterestedIn(TypeCMessage.getClass.getName)
    case message: Any =>
      println(s"TypeCAlsoInterested: received unexpected message: $message")
  }
}
```

```scala
// 不感兴趣的消息

package io.binglau.scala.akka.demo.dynamic_router

import akka.actor.Actor

// 未注册消息
class DunnoInterested extends Actor {
  override def receive: Receive = {
    case message: Any =>
      println(s"DunnoInterest: received undeliverable message: $message")
  }
}
```

```scala
// 执行 demo
package io.binglau.scala.akka.demo.dynamic_router

import akka.actor.{ActorSystem, Props}

import scala.concurrent.Await
import scala.concurrent.duration._

object DynamicRouter extends App {
  val system = ActorSystem("dynamicRouter")

  val dunnoInterested = system.actorOf(Props[DunnoInterested], "dunnoInterested")

  val canStartAfterRegistered: Int = 5
  val canCompleteAfterUnregistered: Int = 1
  val typedMessageInterestRouter =
    system.actorOf(Props(
      new TypedMessageInterestRouter(dunnoInterested, canStartAfterRegistered, canCompleteAfterUnregistered)),
      "typedMessageInterestRouter")

  val typeAInterest = system.actorOf(Props(classOf[TypeAInterested], typedMessageInterestRouter), "typeAInterest")
  val typeBInterest = system.actorOf(Props(classOf[TypeBInterested], typedMessageInterestRouter), "typeBInterest")
  val typeCInterest = system.actorOf(Props(classOf[TypeCInterested], typedMessageInterestRouter), "typeCInterest")
  val typeCAlsoInterested = system.actorOf(Props(classOf[TypeCAlsoInterested], typedMessageInterestRouter), "typeCAlsoInterested")

  typedMessageInterestRouter ! TypeAMessage("Message of TypeA.")
  typedMessageInterestRouter ! TypeBMessage("Message of TypeB.")
  typedMessageInterestRouter ! TypeCMessage("Message of TypeC.")

  typedMessageInterestRouter ! TypeCMessage("Another message of TypeC.")
  typedMessageInterestRouter ! TypeDMessage("Message of TypeD.")

  println("DynamicRouter: is completed.")

  Await.ready(system.whenTerminated, 5.seconds)
}

/**
[DEBUG] [09/19/2017 00:37:03.655] [main] [EventStream] StandardOutLogger started
[DEBUG] [09/19/2017 00:37:04.361] [main] [EventStream(akka://dynamicRouter)] logger log1-Logging$DefaultLogger started
[DEBUG] [09/19/2017 00:37:04.361] [main] [EventStream(akka://dynamicRouter)] logger log1-Logging$DefaultLogger started
[DEBUG] [09/19/2017 00:37:04.364] [main] [EventStream(akka://dynamicRouter)] Default Loggers started
[DEBUG] [09/19/2017 00:37:04.364] [main] [EventStream(akka://dynamicRouter)] Default Loggers started
DynamicRouter: is completed.
TypeAInterested: received: TypeAMessage(Message of TypeA.)
TypeBInterested: received: TypeBMessage(Message of TypeB.)
TypeCInterested: received: TypeCMessage(Message of TypeC.)
TypeCInterested: received: TypeCMessage(Another message of TypeC.)
DunnoInterest: received undeliverable message: TypeDMessage(Message of TypeD.)
unregister count > can CompleteAfterUnregistered
[DEBUG] [09/19/2017 00:37:05.974] [dynamicRouter-akka.actor.default-dispatcher-3] [EventStream] shutting down: StandardOutLogger started
[DEBUG] [09/19/2017 00:37:05.974] [dynamicRouter-akka.actor.default-dispatcher-3] [EventStream] shutting down: StandardOutLogger started
[DEBUG] [09/19/2017 00:37:05.981] [dynamicRouter-akka.actor.default-dispatcher-3] [EventStream] all default loggers stopped
**/
```

### 分散-聚集路由器

#### 分离器

当需要将较大的消息分割成多个独立部分，并将这些独立部分作为消息发送时，可使用分离器。分离器主要用于将构成一条消息的各个独立部分，分发给不同的子系统。

#### 聚合器

将一些消息整合作为整体发送。

![分散-聚集路由器](https://github.com/BingLau7/blog/blob/master/images/blog_26/%E5%88%86%E6%95%A3%E2%80%94%E8%81%9A%E9%9B%86%E8%B7%AF%E7%94%B1%E5%99%A8.png?raw=true)

```scala
package io.binglau.scala.akka.demo.scatter_gather

import java.util.concurrent.TimeUnit
import scala.concurrent._
import scala.concurrent.duration._
import ExecutionContext.Implicits.global
import akka.actor._

case class RequestForQuotation(rfqId: String, retailItems: Seq[RetailItem]) {
  val totalRetailPrice: Double = retailItems.map(retailItem => retailItem.retailPrice).sum
}

case class RetailItem(itemId: String, retailPrice: Double)

case class RequestPriceQuote(rfqId: String, itemId: String, retailPrice: Double, orderTotalRetailPrice: Double)

case class PriceQuote(quoterId: String, rfqId: String, itemId: String, retailPrice: Double, discountPrice: Double)

case class PriceQuoteFulfilled(priceQuote: PriceQuote)

case class PriceQuoteTimedOut(rfqId: String)

case class RequiredPriceQuotesForFulfillment(rfqId: String, quotesRequested: Int)

case class QuotationFulfillment(rfqId: String, quotesRequested: Int, priceQuotes: Seq[PriceQuote], requester: ActorRef)

case class BestPriceQuotation(rfqId: String, priceQuotes: Seq[PriceQuote])

case class SubscribeToPriceQuoteRequests(quoterId: String, quoteProcessor: ActorRef)

// 使用发布-订阅，将消息发送给已注册的接收者
object ScatterGather extends App {
  val system = ActorSystem("demo")
  val priceQuoteAggregator = system.actorOf(Props[PriceQuoteAggregator], "priceQuoteAggregator")

  val orderProcessor = system.actorOf(Props(classOf[MountaineeringSuppliesOrderProcessor], priceQuoteAggregator), "orderProcessor")

  // 对于报价不同的业务处理
  system.actorOf(Props(classOf[BudgetHikersPriceQuotes], orderProcessor), "budgetHikers")
  system.actorOf(Props(classOf[HighSierraPriceQuotes], orderProcessor), "highSierra")
  system.actorOf(Props(classOf[MountainAscentPriceQuotes], orderProcessor), "mountainAscent")
  system.actorOf(Props(classOf[PinnacleGearPriceQuotes], orderProcessor), "pinnacleGear")
  system.actorOf(Props(classOf[RockBottomOuterwearPriceQuotes], orderProcessor), "rockBottomOuterwear")

  orderProcessor ! RequestForQuotation("123",
    Vector(RetailItem("1", 29.95),
      RetailItem("2", 99.95),
      RetailItem("3", 14.95)))

  orderProcessor ! RequestForQuotation("125",
    Vector(RetailItem("4", 39.99),
      RetailItem("5", 199.95),
      RetailItem("6", 149.95),
      RetailItem("7", 724.99)))

  orderProcessor ! RequestForQuotation("129",
    Vector(RetailItem("8", 119.99),
      RetailItem("9", 499.95),
      RetailItem("10", 519.00),
      RetailItem("11", 209.50)))

  orderProcessor ! RequestForQuotation("135",
    Vector(RetailItem("12", 0.97),
      RetailItem("13", 9.50),
      RetailItem("14", 1.99)))

  orderProcessor ! RequestForQuotation("140",
    Vector(RetailItem("15", 1295.50),
      RetailItem("16", 9.50),
      RetailItem("17", 599.99),
      RetailItem("18", 249.95),
      RetailItem("19", 789.99)))

  println("Scatter-Gather: is completed.")

  Await.ready(system.whenTerminated, 5.seconds)
}

class MountaineeringSuppliesOrderProcessor(priceQuoteAggregator: ActorRef) extends Actor {
  val subscribers: scala.collection.mutable.Map[String, SubscribeToPriceQuoteRequests] =
    scala.collection.mutable.Map[String, SubscribeToPriceQuoteRequests]()

  // 将 RequestPriceQuote 消息发送给所有订阅者
  def dispatch(rfq: RequestForQuotation): Unit = {
    subscribers.values.foreach { subscriber =>
      val quoteProcessor = subscriber.quoteProcessor
      rfq.retailItems.foreach { retailItem =>
        println("OrderProcessor: " + rfq.rfqId + " item: " + retailItem.itemId + " to: " + subscriber.quoterId)
        quoteProcessor ! RequestPriceQuote(rfq.rfqId, retailItem.itemId, retailItem.retailPrice, rfq.totalRetailPrice)
      }
    }
  }

  override def receive: Receive = {
    case subscriber: SubscribeToPriceQuoteRequests =>
      subscribers(subscriber.quoterId) = subscriber
    case priceQuote: PriceQuote =>
      priceQuoteAggregator ! PriceQuoteFulfilled(priceQuote)
      println(s"OrderProcessor: received: $priceQuote")
    case rfq: RequestForQuotation =>
      priceQuoteAggregator ! RequiredPriceQuotesForFulfillment(rfq.rfqId, subscribers.size * rfq.retailItems.size)
      dispatch(rfq)
    case bestPriceQuotation: BestPriceQuotation =>
      println(s"OrderProcessor: received: $bestPriceQuotation")
      // 产生最佳报价，结束
      context.system.terminate()
    case message: Any =>
      println(s"OrderProcessor: received unexpected message: $message")
  }
}

class PriceQuoteAggregator extends Actor {
  val fulfilledPriceQuotes: scala.collection.mutable.Map[String, QuotationFulfillment] =
    scala.collection.mutable.Map[String, QuotationFulfillment]()

  // BestPriceQuotation 消息包含通过多个报价引擎提供的 PriceQuote 实例合并出的最佳报价
  def bestPriceQuotationFrom(quotationFulfillment: QuotationFulfillment): BestPriceQuotation = {
    val bestPrices = scala.collection.mutable.Map[String, PriceQuote]()

    quotationFulfillment.priceQuotes.foreach { priceQuote =>
      if (bestPrices.contains(priceQuote.itemId)) {
        if (bestPrices(priceQuote.itemId).discountPrice > priceQuote.discountPrice) {
          bestPrices(priceQuote.itemId) = priceQuote
        }
      } else {
        bestPrices(priceQuote.itemId) = priceQuote
      }
    }

    BestPriceQuotation(quotationFulfillment.rfqId, bestPrices.values.toVector)
  }

  override def receive: Receive = {
    case required: RequiredPriceQuotesForFulfillment =>
      fulfilledPriceQuotes(required.rfqId) = QuotationFulfillment(required.rfqId, required.quotesRequested, Vector(), sender)
      // 超时设置
      // scheduler 将会在 duration 后将 message 送给 self
      val duration = Duration.create(2, TimeUnit.SECONDS)
      context.system.scheduler.scheduleOnce(duration, self, PriceQuoteTimedOut(required.rfqId))
    case priceQuoteFulfilled: PriceQuoteFulfilled =>
      priceQuoteRequestFulfilled(priceQuoteFulfilled)
      println(s"PriceQuoteAggregator: fulfilled price quote: $PriceQuoteFulfilled")
    case priceQuoteTimedOut: PriceQuoteTimedOut => // 报价超时设置
      priceQuoteRequestTimedOut(priceQuoteTimedOut.rfqId)
    case message: Any =>
      println(s"PriceQuoteAggregator: received unexpected message: $message")
  }

  def priceQuoteRequestFulfilled(priceQuoteFulfilled: PriceQuoteFulfilled): Unit = {
    if (fulfilledPriceQuotes.contains(priceQuoteFulfilled.priceQuote.rfqId)) {
      val previousFulfillment = fulfilledPriceQuotes(priceQuoteFulfilled.priceQuote.rfqId)
      val currentPriceQuotes = previousFulfillment.priceQuotes :+ priceQuoteFulfilled.priceQuote
      val currentFulfillment =
        QuotationFulfillment(
          previousFulfillment.rfqId,
          previousFulfillment.quotesRequested,
          currentPriceQuotes,
          previousFulfillment.requester)

      if (currentPriceQuotes.size >= currentFulfillment.quotesRequested) {
        quoteBestPrice(currentFulfillment)
      } else {
        fulfilledPriceQuotes(priceQuoteFulfilled.priceQuote.rfqId) = currentFulfillment
      }
    }
  }

  // 出现超时情况，PriceQuoteAggregator 会根据它收到的 PriceQuoteFulfilled 消息
  // 的数量，为 MountaineeringSuppliesOrderProcessor 对象提供一条 BestPriceQuotation 消息
  def priceQuoteRequestTimedOut(rfqId: String): Unit = {
    if (fulfilledPriceQuotes.contains(rfqId)) {
      quoteBestPrice(fulfilledPriceQuotes(rfqId))
    }
  }

  def quoteBestPrice(quotationFulfillment: QuotationFulfillment): Unit = {
    if (fulfilledPriceQuotes.contains(quotationFulfillment.rfqId)) {
      quotationFulfillment.requester ! bestPriceQuotationFrom(quotationFulfillment)
      fulfilledPriceQuotes.remove(quotationFulfillment.rfqId)
    }
  }
}

class BudgetHikersPriceQuotes(priceQuoteRequestPublisher: ActorRef) extends Actor {
  val quoterId: String = self.path.name
  // 注册
  priceQuoteRequestPublisher ! SubscribeToPriceQuoteRequests(quoterId, self)

  override def receive: Receive = {
    case rpq: RequestPriceQuote =>
      if (rpq.orderTotalRetailPrice < 1000.00) {
        val discount = discountPercentage(rpq.orderTotalRetailPrice) * rpq.retailPrice
        sender ! PriceQuote(quoterId, rpq.rfqId, rpq.itemId, rpq.retailPrice, rpq.retailPrice - discount)
      } else {
        println(s"BudgetHikersPriceQuotes: ignoring: $rpq")
      }

    case message: Any =>
      println(s"BudgetHikersPriceQuotes: received unexpected message: $message")
  }

  def discountPercentage(orderTotalRetailPrice: Double): Double = {
    if (orderTotalRetailPrice <= 100.00) 0.02
    else if (orderTotalRetailPrice <= 399.99) 0.03
    else if (orderTotalRetailPrice <= 499.99) 0.05
    else if (orderTotalRetailPrice <= 799.99) 0.07
    else 0.075
  }
}

class HighSierraPriceQuotes(priceQuoteRequestPublisher: ActorRef) extends Actor {
  val quoterId: String = self.path.name
  priceQuoteRequestPublisher ! SubscribeToPriceQuoteRequests(quoterId, self)

  override def receive: Receive = {
    case rpq: RequestPriceQuote =>
      val discount = discountPercentage(rpq.orderTotalRetailPrice) * rpq.retailPrice
      sender ! PriceQuote(quoterId, rpq.rfqId, rpq.itemId, rpq.retailPrice, rpq.retailPrice - discount)

    case message: Any =>
      println(s"HighSierraPriceQuotes: received unexpected message: $message")
  }

  def discountPercentage(orderTotalRetailPrice: Double): Double = {
    if (orderTotalRetailPrice <= 150.00) 0.015
    else if (orderTotalRetailPrice <= 499.99) 0.02
    else if (orderTotalRetailPrice <= 999.99) 0.03
    else if (orderTotalRetailPrice <= 4999.99) 0.04
    else 0.05
  }
}

class MountainAscentPriceQuotes(priceQuoteRequestPublisher: ActorRef) extends Actor {
  val quoterId: String = self.path.name
  priceQuoteRequestPublisher ! SubscribeToPriceQuoteRequests(quoterId, self)

  override def receive: Receive = {
    case rpq: RequestPriceQuote =>
      val discount = discountPercentage(rpq.orderTotalRetailPrice) * rpq.retailPrice
      sender ! PriceQuote(quoterId, rpq.rfqId, rpq.itemId, rpq.retailPrice, rpq.retailPrice - discount)

    case message: Any =>
      println(s"MountainAscentPriceQuotes: received unexpected message: $message")
  }

  def discountPercentage(orderTotalRetailPrice: Double): Double = {
    if (orderTotalRetailPrice <= 99.99) 0.01
    else if (orderTotalRetailPrice <= 199.99) 0.02
    else if (orderTotalRetailPrice <= 499.99) 0.03
    else if (orderTotalRetailPrice <= 799.99) 0.04
    else if (orderTotalRetailPrice <= 999.99) 0.045
    else if (orderTotalRetailPrice <= 2999.99) 0.0475
    else 0.05
  }
}

class PinnacleGearPriceQuotes(priceQuoteRequestPublisher: ActorRef) extends Actor {
  val quoterId: String = self.path.name
  priceQuoteRequestPublisher ! SubscribeToPriceQuoteRequests(quoterId, self)

  override def receive: Receive = {
    case rpq: RequestPriceQuote =>
      val discount = discountPercentage(rpq.orderTotalRetailPrice) * rpq.retailPrice
      sender ! PriceQuote(quoterId, rpq.rfqId, rpq.itemId, rpq.retailPrice, rpq.retailPrice - discount)

    case message: Any =>
      println(s"PinnacleGearPriceQuotes: received unexpected message: $message")
  }

  def discountPercentage(orderTotalRetailPrice: Double): Double = {
    if (orderTotalRetailPrice <= 299.99) 0.015
    else if (orderTotalRetailPrice <= 399.99) 0.0175
    else if (orderTotalRetailPrice <= 499.99) 0.02
    else if (orderTotalRetailPrice <= 999.99) 0.03
    else if (orderTotalRetailPrice <= 1199.99) 0.035
    else if (orderTotalRetailPrice <= 4999.99) 0.04
    else if (orderTotalRetailPrice <= 7999.99) 0.05
    else 0.06
  }
}

class RockBottomOuterwearPriceQuotes(priceQuoteRequestPublisher: ActorRef) extends Actor {
  val quoterId: String = self.path.name
  priceQuoteRequestPublisher ! SubscribeToPriceQuoteRequests(quoterId, self)

  override def receive: Receive = {
    case rpq: RequestPriceQuote =>
      if (rpq.orderTotalRetailPrice < 2000.00) {
        val discount = discountPercentage(rpq.orderTotalRetailPrice) * rpq.retailPrice
        sender ! PriceQuote(quoterId, rpq.rfqId, rpq.itemId, rpq.retailPrice, rpq.retailPrice - discount)
      } else {
        println(s"RockBottomOuterwearPriceQuotes: ignoring: $rpq")
      }

    case message: Any =>
      println(s"RockBottomOuterwearPriceQuotes: received unexpected message: $message")
  }

  def discountPercentage(orderTotalRetailPrice: Double): Double = {
    if (orderTotalRetailPrice <= 100.00) 0.015
    else if (orderTotalRetailPrice <= 399.99) 0.02
    else if (orderTotalRetailPrice <= 499.99) 0.03
    else if (orderTotalRetailPrice <= 799.99) 0.04
    else if (orderTotalRetailPrice <= 999.99) 0.05
    else if (orderTotalRetailPrice <= 2999.99) 0.06
    else if (orderTotalRetailPrice <= 4999.99) 0.07
    else if (orderTotalRetailPrice <= 5999.99) 0.075
    else 0.08
  }
}
```

### 轮询消费者

![轮询消费者](https://github.com/BingLau7/blog/blob/master/images/blog_26/%E8%BD%AE%E8%AF%A2%E6%B6%88%E8%B4%B9%E8%80%85.png?raw=true)

在这种模式中，消费者通过轮询方式向指定资源请求获取信息。在资源能够提供该信息前，需要阻塞消费者。与此相反，在使用 Actor 模型时，无法使 Actor 对象以轮询方式向另一个 Actor 对象请求信息，因为 Actor 对象之间的协同操作不会被阻塞。一个 Actor 对象从另一个 Actor 对象获取信息的唯一方式是使用请求—回复模式。也就是说，需要使用请求—回复模式高效地模拟轮询消费者模式。

在典型的过程化轮询环境中，工作消费者会从工作提供者那里请求获取工作。在工作提供者能够将被请求的工作分配给工作消费者前，工作消费者会被阻塞。Actor 模型中肯定不会出现这种情况。在使用 Actor 对象前，当工作消费者告诉工作提供者它需要获得工作时，工作消费者仍旧会在它本身的线程中继续运行，而且被发送给工作提供者的请求，过一段时间（这段时间可能长一点也可能短一点）才会被收到。不论接收和处理请求所需实际时间有多久，过程化轮询模式要求的在分配和回复被请求的工作时，使用的阻塞操作都无法被实现。

```scala
package io.binglau.scala.akka.demo.polling_consumer

import scala.collection.immutable.List
import akka.actor._
import scala.concurrent.duration._

import scala.concurrent.Await

object PollingConsumerDriver extends App {
  val system = ActorSystem("demo")

  val workItemsProvider: ActorRef = system.actorOf(Props[WorkItemsProvider], "workItemsProvider")
  val workConsumer: ActorRef = system.actorOf(Props(classOf[WorkConsumer], workItemsProvider), "workConsumer")

  workConsumer ! WorkNeeded()

  println("PollingConsumerDriver: completed.")
  Await.ready(system.whenTerminated, 5.seconds)
}

case class WorkNeeded()
case class WorkOnItem(workItem: WorkItem)

class WorkConsumer(workItemsProvider: ActorRef) extends Actor {
  var totalItemsWorkedOn = 0

  def performWorkOn(workItem: WorkItem): Unit = {
    // 总工作数限制
    totalItemsWorkedOn = totalItemsWorkedOn + 1
    if (totalItemsWorkedOn >= 15) {
      context.stop(self)
      context.system.terminate()
    }
  }

  override def postStop(): Unit = {
    context.stop(workItemsProvider)
  }

  override def receive: Receive = {
    case allocated: WorkItemsAllocated =>
      // 获取资源，消耗资源，再发送申请资源请求
      println("WorkItemsAllocated...")
      allocated.workItems map { workItem =>
        self ! WorkOnItem(workItem)
      }
      self ! WorkNeeded()
    // 轮询起点，申请一次轮询数
    case workNeeded: WorkNeeded =>
      println("WorkNeeded...")
      workItemsProvider ! AllocateWorkItems(5)
    // 处理资源
    case workOnItem: WorkOnItem =>
      println(s"Performed work on: ${workOnItem.workItem.name}")
      performWorkOn(workOnItem.workItem)
  }
}

case class AllocateWorkItems(numberOfItems: Int)
case class WorkItemsAllocated(workItems: List[WorkItem])
case class WorkItem(name: String)

class WorkItemsProvider extends Actor {
  var workItemsNamed: Int = 0

  def allocateWorkItems(numberOfItems: Int): List[WorkItem] = {
    var allocatedWorkItems = List[WorkItem]()
    for (itemCount <- 1 to numberOfItems) {
      val nameIndex = workItemsNamed + itemCount
      allocatedWorkItems = allocatedWorkItems :+ WorkItem("WorkItem" + nameIndex)
    }
    workItemsNamed = workItemsNamed + numberOfItems
    allocatedWorkItems
  }

  override def receive: Receive = {
    case request: AllocateWorkItems =>
      // 发送轮询资源
      sender ! WorkItemsAllocated(allocateWorkItems(request.numberOfItems))
  }
}
```



### 最后

总体上来说，我们一开始接触使用到 Akka 的目的或多或少都是因为其简单易用的并发开发方式，就是说，Akka 虽然说我可以利用它来模拟我们正在开发的任何应用，但是它本质上还是为了解决并发场景下编程的难度问题的，所以其最适合的场景，自然也是并发下的场景。

#### 使用场景

-  事务处理（Transaction Processing）

   在线游戏系统、金融/银行系统、交易系统、投注系统、社交媒体系统、电信服务系统。

-  并行计算（Concurrency/Parallelism）

   任何具有并发/并行计算需求的行业，基于JVM的应用都可以使用，如使用编程语言 Scala、Java、Groovy、JRuby 开发。

-  复杂事件流处理（Complex Event Stream Processing）

   Akka 本身提供的 Actor 就适合处理基于事件驱动的应用，所以可以更加容易处理具有复杂事件流的应用。

-  后端服务（Service Backend）

   任何行业的任何类型的应用都可以使用，比如提供 REST、SOAP 等风格的服务，类似于一个服务总线，Akka 支持纵向 & 横向扩展，以及容错/高可用（HA）的特性。

### 番外

另外随便说一下，之前在搜集资料的时候，看到了这样的一篇文章

[Using Scala and Akka with Domain-Driven Design](https://vaughnvernon.co/?p=770)

所谓 DDD 领域驱动设计(DDD:Domain-Driven Design)。维基百科是有着这样的解释：

>**Domain-driven design** (**DDD**) is an approach to [software development](https://en.wikipedia.org/wiki/Software_development) for complex needs by connecting the [implementation](https://en.wikipedia.org/wiki/Implementation) to an evolving model.[[1\]](https://en.wikipedia.org/wiki/Domain-driven_design#cite_note-definition-1) The premise of domain-driven design is the following:
>
>-  placing the project's primary focus on the core [domain](https://en.wikipedia.org/wiki/Domain_(software_engineering)) and domain logic;（关注项目中涉及到的核心领域以及其领域逻辑）
>-  basing complex designs on a model of the domain;（基于领域模型上进行复杂的架构）
>-  initiating a creative collaboration between technical and [domain experts](https://en.wikipedia.org/wiki/Domain_expert) to iteratively refine a conceptual model that addresses particular domain problems.（易于开启技术与领域专家之间的合作，以迭代的方式改进在特定领域下的特定问题的解决模型）

虽说设计与语言无关，但是在 Akka 编程的时候有特别大的优势是，Actor 模型限制了你，在一个 Actor 中你只能获取有限的数据做有限的事，很好的将你所要完成的事情分为一个个单独的模块，而且一个 Actor 获知其他 Actor 的方式及其有限，只要不是瞎写（不受限制的创建子 Actor 会内存泄露）基本是模块分明，在加上路由器的设置，可以将中心领域问题凸显出来。



### 参考文档

[Akka 官方文档](http://doc.akka.io/docs/akka/current/scala/index-actors.html)
《响应式架构——消息模式 Actor 实现与 Scala、Akka 应用集成》
