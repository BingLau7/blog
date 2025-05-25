---
title: Java并发基础-锁
date: 2025-05-25 17:58:46
tags:
    - Java
---

## 锁
+ synchronized
+ ReentranLock
+ ReentranReadWriteLock
+ StampedLock

### AQS(`AbstractQueuedSynchronizer`)
锁（ReentranLock）的基础类实现，也是很多并发工具类的基础类依赖（比如 Semaphore, CountDownLatch）。

通过一个 `int state` 成员变量表示同步状态（如锁的持有次数），并通过内置的 **FIFO 队列 管理等待获取资源的线程**，从而实现**多线程对共享资源的有序访问**。

#### 关键方法
1. **状态管理方法**

`**tryAcquire(int arg)**`**：**独占式尝试获取同步状态。需要子类实现，若成功返回 `true`，否则返回 `false`。

`**tryRelease(int arg)**`**：**独占式释放同步状态。子类实现。成功返回 `true`否则 `false`

`**tryAcquireShared(int args)**`**：**共享式尝试获取同步状态。子类实现，返回值表示剩余可用资源，负数失败，0/正数成功。

`**tryReleaseShared(int args)**`**：**共享式释放状态。子类实现。

2. **模板方法**

`**acquire(int arg)**`**：**独占式获取同步状态。调用 `tryAcquire`尝试获取，失败则将线程加入等待队列并**自旋**，直到前驱节点为头节点且成功获取资源。

`**release(int arg)**`**：**独占式释放同步状态。调用 `tryRelease`释放后，唤醒后继节点的线程。

`**acquireShared(int arg)**`**：**共享式获取同步状态。调用 `tryAcquiredShared`成功后，传播唤醒后续节点。

`**releaseShared(int arg)**`**：**共享式释放同步状态。调用 `tryReleaseShared`后，唤醒后续等待的线程。

3. **条件队列管理**

`**newCondition()**`**：**创建与 AQS 实例绑定的 `Condition`对象，用于实现线程等待/通知机制（如 `await()`和 `signal()`）

4. **辅助方法**

`**isHeldExclusively()**`**：**判断当前同步状态是否被独占（如 ReentranLock 中检测锁持有线程是否为当前线程）

`**acquireInterruptibly(int arg)**`**：**可中断的获取同步状态。在自旋过程中若检测到线程中断，直接抛出异常并退出。

#### 核心设计原理
+ **同步状态管理 **：通过 `**>volatile int state**` 变量表示状态，依赖 CAS 操作保证原子性（如 `**>compareAndSetState**`）<font style="background-color:rgb(224, 223, 255);">
+ **队列同步 **：维护一个双向 FIFO 队列（CLH 队列变体），将未获取资源的线程封装为节点（Node）并阻塞，前驱节点释放后唤醒后继节点 <font style="background-color:rgb(224, 223, 255);">
+ **子类扩展 **：子类通过实现 `**>tryAcquire**`/`**>tryRelease**` 等方法定义同步逻辑，AQS 负责底层线程阻塞、唤醒及队列管理

#### 典型使用场景
+ **独占锁 **：如 `**>ReentrantLock**`，通过 `**>tryAcquire**` 和 `**>tryRelease**` 实现可重入锁逻辑。
+ **共享锁 **：如 `**>Semaphore**` 和 `**>CountDownLatch**`，通过 `**>tryAcquireShared**` 和 `**>tryReleaseShared**` 控制资源访问。
+ **条件变量 **：结合 `**>Condition**` 实现线程间的精确等待与通知

#### 学习收获
1. AQS 明显使用了**模板方法**的模式，在确定使用用途的时候，通过固定的代码框架可以避免代码的重复性。但是需要对后续实现功能有比较确切的了解才能使用，否则后续容易导致子类变更影响的扩散。
2. `volatile int state`+ CAS 的操作以**无锁化**的方式实现了线程安全。
3. AQS 内部维护了一个基于 **CLH 锁**（基于单向链表的高性能、公平的自旋锁，核心是将竞争锁的线程组织称一个隐式的队列，每个线程仅在本地变量上自旋，通过轮询前驱节点的状态来判断是否可以获取锁）变体的 FIFO 同步队列，用于管理等待获取资源的线程，其中关键技巧：
    1. 节点自旋与阻塞 ：等待线程通过自旋检查前驱节点状态，减少不必要的线程唤醒开销；若长时间未获取资源，则调用 LockSupport.park 阻塞线程 
    2. 传播唤醒机制 ：在共享模式下（如 releaseShared），唤醒后继节点的同时可能继续传播唤醒后续节点，避免线程饥饿 
4. 条件变量与多路通知。AQS 提供的 **Condition 接口支持线程等待/通知机制**，允许线程在特定条件下释放锁并进入等待队列，待条件满足后被唤醒 。其设计体现了：
    1. 分离等待队列 ：每个 Condition 维护独立的等待队列，避免多线程竞争导致的唤醒冲突。
    2. 精确唤醒 ：通过 signal 和 signalAll 可选择性地唤醒等待线程，而非盲目通知所有线程

#### CHL 锁说明介绍
> 1. 单机多核系统的线程同步（AQS）
> 2. 需要公平性保证的场景
> 3. 地空间复杂度需求：CLH 锁空间复杂度是 O(L+n) (L 是锁数量，n 是线程数)
>
> 相较于传统操作系统的互斥锁（Mutex）通过自旋减少上下文切换的开销，适合低延迟的场景。
>
> Java 中的 synchronized 是基于 CPU 指令(monitor 对象)实现的，而 ReentrantLock 基于 AQS 实现的。
>
> **CLH 锁实现原理简单概述？**
>
> 线程通过自旋检查前驱节点状态来决定是否获取锁。其核心原理是：
>
> 1. **队列结构 **：线程竞争锁时，会以 FIFO（先进先出）顺序加入链表队列，每个节点仅需关注前驱节点 <font style="background-color:rgb(224, 223, 255);">
> 2. **自旋机制 **：线程在未获取锁时持续自旋检查前驱节点的状态，当前驱节点释放锁后，当前线程才能尝试获取锁 <font style="background-color:rgb(224, 223, 255);">
> 3. **公平性 **：保证线程按申请顺序获取锁，避免饥饿问题
> 4. **无锁化操作 **：通过 CAS 原子操作维护队列，减少锁竞争
>
> **实现原理描述**
>
> 数据结构定义
>

```java
class CLHNode:
    prev: CLHNode  // 前驱节点
    next: CLHNode  // 后继节点
    status: Boolean  // 状态（是否持有锁）

class CLHLock:
    tail: CLHNode  // 尾指针，指向队列最后一个节点
```

> lock 方法
>

```java
function lock():
    node = new CLHNode()
    node.status = false  // 初始状态为未持有锁
    prev_node = CASExchange(tail, node)  // 原子操作：将当前节点插入队列尾部
    if prev_node == null:
        // 队列为空，当前线程直接获取锁
        return true
    else:
        node.prev = prev_node  // 设置前驱节点
        // 自旋等待前驱节点释放锁
        while prev_node.status == true:
            // 等待...
```

> unlock 方法
>

```java
function unlock():
    current_node = getCurrentNode()  // 获取当前线程的节点
    if current_node.next == null:
        // 如果没有后继节点，尝试将尾指针置空（释放队列）
        if CASCompareAndSet(tail, current_node, null):
            return
    else:
        // 唤醒后继节点
        current_node.next.status = true  // 标记后继节点可以获取锁
    current_node.status = true  // 当前线程释放锁
```

### synchronized 的实现原理
`synchronized` 的实现原理主要依赖于 **Monitor（监视器）机制** 和 **JVM 的锁管理**，具体如下：

1. **基于 Monitor 的同步机制**  
在 JVM 中，每个对象都关联一个 Monitor 对象。当线程执行 `synchronized` 代码块或方法时，会通过 `monitorenter` 和 `monitorexit` 指令尝试获取或释放 Monitor 锁。若锁已被其他线程持有，则当前线程会被阻塞，直到锁被释放 。
2. **锁的获取与释放**  
    - **代码块同步**：通过 `monitorenter` 指令标记同步代码块的起始，线程进入时需获取锁；执行完成后通过 `monitorexit` 释放锁。
    - **方法同步**：通过方法访问标志中的 `ACC_SYNCHRONIZED` 标志实现，线程调用方法时自动尝试获取锁，方法执行结束或抛出异常时释放锁 。
3. **保证并发安全的三大特性**  
    - **原子性**：通过锁机制确保同一时间只有一个线程执行同步代码，避免数据竞争 。
    - **可见性**：锁的获取和释放隐含内存屏障操作，确保线程对共享变量的修改对其他线程可见 。
    - **有序性**：通过限制指令重排序，保证代码按预期顺序执行 。
4. **锁的优化（JDK 6 及以后）**  
JDK 6 引入了 **自适应自旋锁**，即自旋次数不再固定，而是根据前一次在同一锁上的自旋效果动态调整。例如，若某锁的自旋成功率较低，则减少后续自旋次数，从而减少 CPU 资源浪费 。
5. **底层实现的互斥性**  
`synchronized` 是由 JVM 直接支持的互斥同步机制，其核心是通过操作系统层面的互斥锁（如 pthread_mutex）实现线程阻塞与唤醒 。

```java
public class SynchronizedDemo {
    private int i = 0;

    public static void main(String[] args) throws Exception {
        SynchronizedDemo demo = new SynchronizedDemo();
        for (int i = 0; i < 5; i++) {
            Thread t = new Thread(demo::run);
            t.start();
        }
        Thread.sleep(1000);
        System.out.println(demo.i);
    }

    public void run() {
        synchronized (this){
            for (int j = 0; j < 100; j++) {
                i++;
            }
        }
    }
}

对应字节码
// class version 65.0 (65)
// access flags 0x21
public class com/oceanbase/oms/SynchronizedDemo {

    // compiled from: SynchronizedDemo.java
    // access flags 0x19
    public final static INNERCLASS java/lang/invoke/MethodHandles$Lookup java/lang/invoke/MethodHandles Lookup

    // access flags 0x2
    private I i

    // access flags 0x1
    public <init>()V
    L0
    LINENUMBER 3 L0
    ALOAD 0
    INVOKESPECIAL java/lang/Object.<init> ()V
    L1
    LINENUMBER 4 L1
    ALOAD 0
    ICONST_0
    PUTFIELD com/oceanbase/oms/SynchronizedDemo.i : I
    RETURN
    L2
    LOCALVARIABLE this Lcom/oceanbase/oms/SynchronizedDemo; L0 L2 0
    MAXSTACK = 2
    MAXLOCALS = 1

    // access flags 0x9
    public static main([Ljava/lang/String;)V throws java/lang/Exception 
    L0
    LINENUMBER 7 L0
    NEW com/oceanbase/oms/SynchronizedDemo
    DUP
    INVOKESPECIAL com/oceanbase/oms/SynchronizedDemo.<init> ()V
    ASTORE 1
    L1
    LINENUMBER 8 L1
    ICONST_0
    ISTORE 2
    L2
    FRAME APPEND [com/oceanbase/oms/SynchronizedDemo I]
    ILOAD 2
    ICONST_5
    IF_ICMPGE L3
    L4
    LINENUMBER 9 L4
    NEW java/lang/Thread
    DUP
    ALOAD 1
    DUP
    INVOKESTATIC java/util/Objects.requireNonNull (Ljava/lang/Object;)Ljava/lang/Object;
    POP
    INVOKEDYNAMIC run(Lcom/oceanbase/oms/SynchronizedDemo;)Ljava/lang/Runnable; [
    // handle kind 0x6 : INVOKESTATIC
    java/lang/invoke/LambdaMetafactory.metafactory(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
    // arguments:
    ()V, 
    // handle kind 0x5 : INVOKEVIRTUAL
    com/oceanbase/oms/SynchronizedDemo.run()V, 
    ()V
    ]
    INVOKESPECIAL java/lang/Thread.<init> (Ljava/lang/Runnable;)V
    ASTORE 3
    L5
    LINENUMBER 10 L5
    ALOAD 3
    INVOKEVIRTUAL java/lang/Thread.start ()V
    L6
    LINENUMBER 8 L6
    IINC 2 1
    GOTO L2
    L3
    LINENUMBER 12 L3
    FRAME CHOP 1
    LDC 1000
    INVOKESTATIC java/lang/Thread.sleep (J)V
    L7
    LINENUMBER 13 L7
    GETSTATIC java/lang/System.out : Ljava/io/PrintStream;
    ALOAD 1
    GETFIELD com/oceanbase/oms/SynchronizedDemo.i : I
    INVOKEVIRTUAL java/io/PrintStream.println (I)V
    L8
    LINENUMBER 14 L8
    RETURN
    L9
    LOCALVARIABLE t Ljava/lang/Thread; L5 L6 3
    LOCALVARIABLE i I L2 L3 2
    LOCALVARIABLE args [Ljava/lang/String; L0 L9 0
    LOCALVARIABLE demo Lcom/oceanbase/oms/SynchronizedDemo; L1 L9 1
    MAXSTACK = 4
    MAXLOCALS = 4

  // access flags 0x1
  public run()V
    TRYCATCHBLOCK L0 L1 L2 null
    TRYCATCHBLOCK L2 L3 L2 null
   L4
    LINENUMBER 17 L4
    ALOAD 0
    DUP
    ASTORE 1
    MONITORENTER
   L0
    LINENUMBER 18 L0
    ICONST_0
    ISTORE 2
   L5
   FRAME APPEND [java/lang/Object I]
    ILOAD 2
    BIPUSH 100
    IF_ICMPGE L6
   L7
    LINENUMBER 19 L7
    ALOAD 0
    DUP
    GETFIELD com/oceanbase/oms/SynchronizedDemo.i : I
    ICONST_1
    IADD
    PUTFIELD com/oceanbase/oms/SynchronizedDemo.i : I
   L8
    LINENUMBER 18 L8
    IINC 2 1
    GOTO L5
   L6
    LINENUMBER 21 L6
   FRAME CHOP 1
    ALOAD 1
    MONITOREXIT
   L1
    GOTO L9
   L2
   FRAME SAME1 java/lang/Throwable
    ASTORE 3
    ALOAD 1
    MONITOREXIT
   L3
    ALOAD 3
    ATHROW
   L9
    LINENUMBER 22 L9
   FRAME CHOP 1
    RETURN
   L10
    LOCALVARIABLE j I L5 L6 2
    LOCALVARIABLE this Lcom/oceanbase/oms/SynchronizedDemo; L4 L10 0
    MAXSTACK = 3
    MAXLOCALS = 4
}

```

其中 MONITORENTER / MONITOREXIT 构成了互斥对

![image-1](/image/java/concurrency-lock-1.png)

#### Monitor 对象的介绍
JVM 中的 **Monitor 对象** 是 Java 实现线程同步的核心机制之一，它与对象（或类）紧密关联，用于管理线程对共享资源的互斥访问和协作通信。以下是其核心要点：

**1. Monitor 的本质与作用**

+ **定义**：Monitor 是 JVM 中每个对象或类在逻辑上关联的一个监视器对象，负责管理线程的同步与协作 。  
+ **功能**：  
    - **互斥性**：确保同一时间只有一个线程能持有锁（即 Monitor 的 `owner`），其他线程需等待 。  
    - **线程协作**：通过 `wait()`、`notify()`、`notifyAll()` 等方法实现线程间的条件等待和唤醒 。

**2. Monitor 的实现结构**

在 HotSpot 虚拟机中，Monitor 基于 C++ 的 `ObjectMonitor` 类实现，其核心成员包括：  

+ `_owner`：指向当前持有 Monitor 的线程 。  
+ `_EntryList`：存储等待获取锁的线程队列 。  
+ `_WaitSet`：存储调用 `wait()` 后进入等待状态的线程队列 。  
+ `_count`：记录锁的重入次数 。

此外，Monitor 的状态通过 **Mark Word**（对象头的一部分）与实际数据结构关联。Mark Word 中存储了指向 Monitor 的指针（当锁升级为重量级锁时）。

**3. Monitor 与 **`synchronized`** 的关系**

+ **锁的获取**：当线程执行 `synchronized` 代码块或方法时，JVM 会通过 `monitorenter` 指令尝试获取对象的 Monitor。若 Monitor 未被占用，则线程成为 `owner`；若已被占用，则线程进入 `_EntryList` 等待 。  
+ **锁的释放**：线程执行完同步代码块或方法后，通过 `monitorexit` 指令释放 Monitor，并唤醒 `_EntryList` 中的等待线程 。

**4. Monitor 的生命周期与优化**

+ **存储位置**：Monitor 对象本身存储在 JVM 的非堆内存中（如 C++ 堆），而对象头的 Mark Word 会指向 Monitor 的地址（仅在重量级锁状态下）。  
+ **线程本地分配**：每个线程维护两个 `ObjectMonitor` 列表（`free` 和 `used`），优先从线程本地分配，减少全局竞争 。  
+ **锁优化**：JDK 6 后引入偏向锁、轻量级锁等优化，减少 Monitor 的直接使用，提升性能 。

**5. Monitor 的应用场景**

+ **重量级锁**：当多个线程竞争锁时，Monitor 升级为重量级锁，依赖操作系统互斥量（如 pthread_mutex）实现阻塞与唤醒 。  
+ **线程协作**：通过 `wait()` 和 `notify()` 在 Monitor 的 `_WaitSet` 和 `_EntryList` 之间转移线程，实现生产者-消费者等模式 。

### ReentrantLock 的实现
1. **可重入性 **：同一个线程可以多次获取同一把锁，避免死锁。例如，一个线程在持有锁的情况下再次进入同步代码块时无需重新竞争锁 <font style="background-color:rgb(224, 223, 255);">。
2. **支持公平与非公平模式 **：
    - **公平模式 **：线程按照请求锁的顺序获取锁，避免“插队”现象，但可能降低吞吐量。
    - **非公平模式 **：允许线程“插队”获取锁（如刚释放锁的线程可能立即重新获取），提高性能但可能导致某些线程饥饿 <font style="background-color:rgb(224, 223, 255);">。
3. **灵活的锁控制 **：提供 `**>tryLock()**`（尝试获取锁）、`**>tryLock(long timeout, TimeUnit unit)**`（带超时的尝试获取锁）、`**>lockInterruptibly()**`（可中断的锁获取）等方法，增强对锁行为的细粒度控制 

典型使用

```java
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    // 访问共享资源
} finally {
    lock.unlock(); // 必须在 finally 块中释放锁
}
```

**核心实现说明**

依托于 AQS 实现，其中内部类 `Sync` 集成了 AQS，利用 AQS 的状态管理（state） 、线程队列（CLH） 和 CAS 机制实现锁机制。

`Sync`存在 `FairSync`与 `NonfairSync`两类实现，即公平锁与非公平锁。

**一、获取锁**

1. **公平锁**  
    - **检查等待队列**：线程尝试获取锁时，会先检查 AQS 阻塞队列中是否有等待线程(`hasQueuedPredecessors()`)。如果存在等待线程（即当前线程不是队列的第一个节点），则当前线程获取锁失败，并加入阻塞队列尾部等待(`**acquireQueued(addWaiter(Node.EXCLUSIVE), arg))**`) 。  
    - **CAS 修改状态**：若队列为空或当前线程位于队列头部，则通过 CAS 操作尝试将 `state` 从 0 修改为 1。成功则成为锁的持有者；失败则继续等待 。
2. **非公平锁**  
    - **直接尝试 CAS**：线程不检查等待队列，直接通过 CAS 操作尝试获取锁（即使队列中有等待线程）。若 CAS 成功，则成为锁的持有者 。  
    - **失败后入队**：若 CAS 失败，则检查当前线程是否已持有锁（可重入性）。若未持有，则加入 AQS 阻塞队列等待 。

---

**二、释放锁**

公平锁与非公平锁的释放流程完全一致，均基于 AQS 的通用机制：  

1. **减少同步状态**：线程调用 `unlock()` 方法时，`state` 值减 1（若 `state > 0`）。  
2. **完全释放锁**：当 `state` 减至 0 时，锁被完全释放，AQS 队列中的头节点（等待最久的线程）被唤醒，重新尝试获取锁 。  
3. **唤醒后续线程**：释放锁后，通过 `unparkSuccessor()` 方法唤醒阻塞队列中的下一个线程，确保锁的公平传递 。

**核心差异总结**

| **步骤** | **公平锁** | **非公平锁** |
| --- | --- | --- |
| **获取锁** | 严格按队列顺序获取，避免“插队” | 先尝试“插队”获取锁，失败后再入队 |
| **释放锁** | 与非公平锁完全一致 | 与公平锁完全一致 |


### **关键实现原理**
+ **公平锁**：通过 `hasQueuedPredecessors()` 方法检查队列中是否有前驱节点，确保“先来先服务” 。  
+ **非公平锁**：通过 `nonfairTryAcquire()` 方法直接尝试 CAS，允许新线程“插队”获取锁 。

通过上述机制，公平锁保证了线程获取锁的顺序性，而非公平锁通过牺牲公平性提升了性能 。

### ReentrantReadWriteLock 的实现
ReentrantReadWriteLock 是 Java 中用于管理读写并发访问的可重入锁，其核心用途在于 **允许多个读线程同时访问共享资源，而写线程独占资源**，从而优化读多写少场景下的性能 。它适用于缓存、共享数据结构等需要高效读取的场景 。

**实现原理与核心思想**

1. **基于 AQS 的状态设计**  
ReentrantReadWriteLock 通过内部类 `Sync` 继承 AQS（AbstractQueuedSynchronizer），利用 AQS 的同步状态（state）管理读写锁的获取与释放。其核心思想是通过 **位运算分割 state**：高 16 位表示读锁的持有次数**（读状态）**，低 16 位表示写锁的持有次数**（写状态）**。

```java
/*
 * Read vs write count extraction constants and functions.
 * Lock state is logically divided into two unsigned shorts:
 * The lower one representing the exclusive (writer) lock hold count,
 * and the upper the shared (reader) hold count.
 */

static final int SHARED_SHIFT   = 16;
static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

/** Returns the number of shared holds represented in count. */
static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
/** Returns the number of exclusive holds represented in count. */
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```

2. **公平性与非公平性**  
支持公平锁和非公平锁，通过构造函数传入 `fair` 参数控制。默认为非公平模式，以提升吞吐量；公平模式则保证等待时间最长的线程优先获取锁 。
3. **读写锁的互斥规则**  
    - **读锁共享**：多个线程可同时获取读锁，但写锁未被占用时才允许 。  
    - **写锁独占**：写锁被占用时，其他读写线程均需等待 。  
    - **锁降级**：写锁可降级为读锁，但读锁不能升级为写锁 。
4. **可重入性**  
同一线程可多次获取读锁或写锁，通过记录持有次数实现重入，并在释放时递减计数 。

**主要实现方法**

1. `**tryAcquireShared**`** 方法**

该方法用于尝试获取读锁（共享锁），其核心流程如下：

+ **检查写锁状态**：如果当前存在写锁且持有者不是当前线程，则直接返回 `-1`（获取失败），避免读写冲突 。
+ **读锁计数更新**：通过位运算获取当前读锁的持有次数（`state` 高 16 位），并尝试增加 1。若超过最大重入次数（`65535`），则抛出异常 。
+ **CAS 更新状态**：使用原子操作 `compareAndSetState` 更新 `state` 的高 16 位（读锁计数），确保线程安全 。
+ **成功获取读锁**：若 CAS 成功，则记录当前线程的读锁重入次数（通过 `ThreadLocal` 管理），并返回 `1`，表示获取共享锁成功 。
+ **失败处理**：若因写锁占用或 CAS 冲突导致失败，返回负值，线程需进入 AQS 队列等待 。（参考下文 `fullTryAcquireShared`方法）
2. `**fullTryAcquireShared**`

fullTryAcquireShared 方法主要用于 在 tryAcquireShared 尝试获取读锁失败后，**进行更完整的重试逻辑。**

    1. **循环尝试获取锁**

fullTryAcquireShared 通过 自旋循环 不断尝试获取读锁，直到以下情况之一发生：

+ 成功更新 state 的高 16 位（读锁计数）。
+ 因写锁占用、公平性规则或线程中断需退出循环 
    2. **检查写锁状态**

如果当前存在写锁`（exclusiveCount(c) != 0）`且持有者不是当前线程，则直接返回 -1，线程需进入等待队列。这确保了**写锁独占**的原则，避免读写冲突 。

    3. **公平性判断**`**（readerShouldBlock()）**`

在公平模式下，调用 readerShouldBlock() 检查是否需要阻塞。例如：

+ 队列中存在更早等待的线程（写锁优先或公平锁规则）。
+ 非公平模式下，仅检查是否有写锁等待（避免写锁饥饿）。

若需阻塞，则线程需加入 AQS 队列等待 。

    4. **读锁重入次数上限**

若当前线程的读锁重入次数超过最大值（65535），抛出 Error（如 ErrorReadCounterOverflow）。因为 state 的高 16 位最多表示 2^16 - 1 = 65535 次重入，避免状态溢出 。

    5. **CAS 更新读锁计数**

通过 `compareAndSetState(c, c + SHARED_UNIT)` 尝试原子更新 state。若 CAS 成功，记录当前线程的读锁重入次数（通过 HoldCounter 或 ThreadLocal 管理），并返回成功状态。

3. `**tryReleaseShared**`** 方法**

该方法用于释放读锁，流程如下：

+ **递减读锁计数**：从 `state` 的高 16 位读取当前读锁次数，减 1 后重新计算新值 。
+ **CAS 更新状态**：使用原子操作更新 `state`，确保读锁计数的线程安全 。
+ **检查是否完全释放**：若读锁计数减至 0（即所有读锁释放），则唤醒 AQS 队列中等待的写锁线程（若有）。
+ **返回值**：始终返回 `true`，表示共享锁释放完成 。
4. `**tryReadLock**`** 方法**

该方法用于非阻塞地尝试获取读锁（类似 `tryLock()`），流程如下：

+ **快速尝试获取**：直接检查写锁状态和当前线程是否已持有写锁（允许锁降级），并尝试通过 CAS 更新读锁计数 。
+ **失败处理**：若写锁被其他线程占用或 CAS 冲突，则立即返回失败，不进入等待队列 。
+ **与 **`tryAcquireShared`** 的对比**：`tryReadLock` 是轻量级的单次尝试，而 `tryAcquireShared` 可能涉及多次重试和队列阻塞 。
5. `**>tryAcquire**`** 方法（写锁的获取）**

该方法用于尝试获取写锁（独占锁），核心流程如下：

+ **检查读锁状态 **：如果当前存在读锁（`**>sharedCount != 0**`），则直接返回 `**>false**`（读写冲突）。
+ **检查写锁持有者 **：若写锁已被其他线程持有（`**>exclusiveCount != 0**` 且持有者不是当前线程），返回 `**>false**`。
+ **重入处理 **：若当前线程已持有写锁，则递增写锁计数（`**>state + 1**`）。
+ **CAS 更新状态 **：尝试通过 `**>compareAndSetState**` 原子更新 `**>state**` 的低 16 位（写锁计数）。若成功，设置当前线程为写锁持有者。
+ **公平性判断 **：在公平模式下，若等待队列中有其他线程等待，则当前线程需进入队列并阻塞。
6. `**>tryRelease**`** 方法（写锁的释放）**

该方法用于释放写锁，流程如下：

+ **递减写锁计数 **：从 `**>state**` 的低 16 位读取当前写锁次数，减 1 后计算新值。
+ **检查是否完全释放 **：若写锁计数减至 0（即所有重入释放完成），则清空写锁持有者标识。
+ **CAS 更新状态 **：通过原子操作更新 `**>state**`，确保线程安全。
+ **唤醒等待线程 **：若写锁完全释放，调用 `**>unparkSuccessor**` 唤醒 AQS 队列中等待的读锁或写锁线程 。
7. `**>tryWriteLock**`** 方法（非阻塞尝试获取写锁）**

该方法用于非阻塞地尝试获取写锁（类似 `**>tryLock()**`），流程如下：

+ **快速检查 **：若当前存在读锁或写锁（且持有者不是当前线程），直接返回失败。
+ **CAS 尝试获取 **：尝试通过 `**>compareAndSetState**` 原子设置 `**>state**` 的低 16 位为 1（初始写锁计数）。若成功，设置当前线程为写锁持有者。
+ **失败处理 **：若因竞争或锁占用导致 CAS 失败，立即返回 `**>false**`，不进入等待队列。
+ **与 **`**>tryAcquire**`** 的对比 **：`**>tryWriteLock**` 是轻量级的单次尝试，而 `**>tryAcquire**` 可能涉及公平性判断和队列阻塞。

### StampedLock 的实现****
使用范例：如何实现一个线程安全的二维点（Point）类，支持并发读写操作，并体现 乐观读 和 锁升级 的典型场景

```java
import java.util.concurrent.locks.StampedLock;

public class Point {
    private double x;
    private double y;
    private final StampedLock lock = new StampedLock();

    // 写锁：移动点到指定坐标
    public void move(double deltaX, double deltaY) {
        long stamp = lock.writeLock();  // 获取写锁
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            lock.unlockWrite(stamp);  // 释放写锁
        }
    }

    // 乐观读：计算点到原点的距离
    public double distanceToOrigin() {
        long stamp = lock.tryOptimisticRead();  // 获取乐观读锁
        double currentX = x;
        double currentY = y;
        // 验证乐观读期间是否有写操作
        if (!lock.validate(stamp)) {
            // 若有写操作，升级为悲观读锁
            stamp = lock.readLock();
            try {
                currentX = x;
                currentY = y;
            } finally {
                lock.unlockRead(stamp);  // 释放悲观读锁
            }
        }
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }

    // 锁升级：如果点在原点，则移动到指定坐标
    public void moveIfAtOrigin(double newX, double newY) {
        long stamp = lock.tryOptimisticRead();  // 尝试乐观读
        double currentX = x;
        double currentY = y;
        if (currentX == 0 && currentY == 0) {
            // 需要升级为写锁
            long writeStamp = lock.convertToWriteLock(stamp);  // 升级为写锁
            try {
                // 二次检查避免其他线程已修改
                if (x == 0 && y == 0) {
                    x = newX;
                    y = newY;
                }
            } finally {
                lock.unlockWrite(writeStamp);  // 释放写锁
            }
        }
    }

    // 获取当前坐标（用于测试）
    public String getCurrentPosition() {
        long stamp = lock.readLock();
        try {
            return "Point(" + x + ", " + y + ")";
        } finally {
            lock.unlockRead(stamp);
        }
    }
}
```

**使用场景**

1. **高并发读多写少的场景**  
StampedLock 的核心优势在于 **乐观读机制**，允许多个线程在无写操作时并发读取共享数据，适用于缓存系统、频繁读取的共享数据结构等场景 。
2. **需避免写线程饥饿的场景**  
StampedLock 允许写线程在公平模式下优先获取锁，解决了传统读写锁中写线程可能因读线程过多而饥饿的问题 。
3. **无需重入锁的场景**  
StampedLock 是非重入锁，若线程不会在持有锁的代码块中再次尝试获取锁，则可使用它 。
4. **需要锁降级或升级的场景**  
支持将乐观读锁升级为写锁（需显式处理），适用于 `if-then-update` 的原子操作场景（如先读取数据，若条件不满足则写入更新）。以下是 **StampedLock** 与 **ReentrantReadWriteLock** 的优劣势对比表格：

**与 ReentrantReadWriteLock 的比较**

| **特性** | **StampedLock** | **ReentrantReadWriteLock** |
| --- | --- | --- |
| **优势** | 1. **高性能**：基于乐观读机制，减少锁竞争，显著提升读多写少场景的吞吐量 。   2. **写线程优先**：支持公平模式，避免写线程饥饿 。   3. **轻量级**：CAS 操作优化，降低资源消耗 。 | 1. **可重入性**：支持线程多次获取读写锁，简化代码逻辑 。   2. **成熟稳定**：广泛应用于传统并发场景，兼容性强 。   3. **精确控制**：适用于需精细管理缓存或共享数据的场景 。 |
| **劣势** | 1. **非重入性**：同一线程多次获取锁会导致死锁风险 。   2. **API 复杂**：需显式管理 `stamp`，增加使用难度 。   3. **不支持条件变量**：无法通过 `Condition` 实现线程通信 。 | 1. **性能瓶颈**：读写互斥，读多写少场景下性能低于 StampedLock 。   2. **写线程饥饿**：高并发下写线程可能长期等待 。   3. **锁降级限制**：需显式处理读写锁转换，逻辑复杂 。 |
| **适用场景** | 高并发读多写少、需高性能吞吐（如缓存系统、共享数据结构） 。 | 需要可重入性、精确控制缓存或低频写操作的场景（如配置管理、少量更新的共享资源） 。 |
| **典型改进点** | 通过乐观读避免线程阻塞，支持锁升级/降级 。 | 提供读写分离，但读写互斥导致性能限制 。 |


+ **StampedLock** 更适合 **高性能、读多写少** 的场景，但需容忍其 **非重入性** 和 **复杂 API** 。
+ **ReentrantReadWriteLock** 更适合 **需要可重入性** 和 **简单控制** 的场景，但需注意写线程饥饿问题 。

**实现原理**

1. **状态管理**  
StampedLock 的状态（`state`）由 **版本号（前 48 位）** 和 **模式（后 16 位）** 组成：
    - **模式位**：表示当前锁的状态（0 表示无锁，1 表示写锁，2 表示悲观读锁，3 表示乐观读锁）。
    - **版本号**：用于标识数据版本，每次写操作会递增版本号，确保乐观读的线程能检测到数据变化 。
2. **乐观读机制**  
    - **读操作**：通过 `tryOptimisticRead()` 获取当前版本号（stamp），读取数据后通过 `validate(stamp)` 检查版本号是否变化。若未变化，说明读期间无写操作；若变化，需升级为悲观读锁或重试 。
    - **写操作**：通过 `writeLock()` 获取独占锁，修改数据后更新版本号，确保其他线程能感知到变更 。
3. **锁升级与降级**  
    - **升级**：将乐观读锁通过 `validate()` 和 `writeLock()` 升级为写锁，但需重新验证数据一致性 。
    - **降级**：持有写锁的线程可通过 `readLock()` 降级为悲观读锁，避免重复竞争 。
4. **等待队列管理**  
基于 AQS 实现线程排队，写锁优先级高于读锁。在公平模式下，线程按 FIFO 顺序获取锁；在非公平模式下，允许写线程插队 。


## 参考文档
- [AQS基础——多图详解CLH锁的原理与实现](https://cloud.tencent.com/developer/article/1690060)
- [解决线程饥饿的神器StampedLock，你值得拥有！](https://cloud.tencent.com/developer/article/1829985)
- 《Java 并发编程深度解析与实战》
