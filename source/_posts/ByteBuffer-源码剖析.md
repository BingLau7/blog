---
title: ByteBuffer 源码剖析
date: 2018-11-12 17:11:18
tags:
    - NIO
    - Java
    - Netty源码剖析系列
categories: 源码分析
---

要点:
1. JDK 原生支持，主要用于 NIO
2. 包含两个实现
    1. HeapByteBuffer: 基于 Java 堆实现
    2. DirectByteBuffer: 使用 unsafe 的 API 进行堆外操作
3. 核心方法为 `put(byte)` 与 `get()`。分别是往ByteBuffer里写一个字节，和读一个字节。
4. 读写模式分离，正常的应用场景是：往ByteBuffer里写一些数据，然后 `flip()`，然后再读出来。
5. 在 JDK11 中 MappedByteBuffer 的创建实际是 DirectByteBuffer
6. DirectByteBuffer 的垃圾回收利用了幻象引用进行回收，详见下面的 `Cleaner`

<!-- more -->

### Demo

```java
    @Test
    public void test() {
        // write
        ByteBuffer buffer = ByteBuffer.allocate(8);
        System.out.printf("before pos/cap/limit: %d, %d, %d\n", buffer.position(), buffer.capacity(), buffer.limit());
        // before pos/cap/limit: 0, 8, 8

        buffer.putShort((short)20);
        buffer.put("test".getBytes());
        System.out.printf("write pos/cap/limit: %d, %d, %d\n", buffer.position(), buffer.capacity(), buffer.limit());
        // write pos/cap/limit: 6, 8, 8

        buffer.flip();
        System.out.printf("flip pos/cap/limit: %d, %d, %d\n", buffer.position(), buffer.capacity(), buffer.limit());
        // flip pos/cap/limit: 0, 8, 6

        short num = buffer.getShort();
        System.out.printf("get short pos/cap/limit: %d, %d, %d\n", buffer.position(), buffer.capacity(), buffer.limit());
        // get short pos/cap/limit: 2, 8, 6
        byte[] strArr = new byte[4];
        buffer.get(strArr, 0, 4);
        System.out.printf("get array pos/cap/limit: %d, %d, %d\n", buffer.position(), buffer.capacity(), buffer.limit());
        // get array pos/cap/limit: 6, 8, 6

        System.out.println("num: " + num + " str: " + new String(strArr));
        // num: 20 str: test
    }
```

### 主要工作原理

**内部字段**

- `byte[] hb`: 缓存数组
- `position`: 当前操作位置
- `capacity`: 初始化 Buffer 容量
- `limit`: 读写的上限，limit <= capacity
- `mark`: 为某一读过的位置做标记，便于某些时候回退到该位置。

主要操作即操作这几个字段用于协同
1. `put/get` 将会增加 `position`，其操作也是从 `position` 开始
2. `put` 将会增加 `limit`，检测 `capacity`
3. `get` 将会检测 `limit`
4. `filp` 将会复位 `position` 一般用于读写转换

### 实现原理

#### HeapByteBuffer 解析

##### init

就 ByteBuffer 来看，`HeapByteBuffer` 中初始化链条**第一环**是 Buffer 的构造器，代码过于简单不进行说明，总体来说是初始化了 `capacity、limit、position、mark`。
这里大家会疑惑 `capacity` 可能通过 `allocate` 传递了，但其他的从哪儿来？  
1. limit = capacity
2. mark = -1 会在构造器中直接置为 0 
3. position = 0

**第二环:** ByteBuffer 的构造器会将 byte[] 初始化并设置 capacity，同时设置 offset = 0.

##### put
核心函数：

- `public ByteBuffer put(byte[] src, int offset, int length)`  
- `public final Buffer position(int newPosition)`  

要点:
1. `checkBounds` 是个有趣的函数，若其中（offset、length、src.length）任一一个值为负数则结果即超出，若 `(size - (off + len))` 为负则结果也为异常。其实现使用了 `|`
2. position 将会重新设置位置，这里面，若 mark 的值大于 postition 了 mark 会重置为 -1
3. 若对比 `ByteBuffer` 父类与 `HeapByteBuffer` 的实现来看会发现，`ByteBuffer` 使用 `put(byte)` 逐个工作而后者则借助了 `System.arrayCopy`


既然都看到这儿了，那我们就了解一下 `System.arrayCopy`，native 源码翻起来

该代码位于 `src/share/native/lang/System.c` 中，主要通过 `Java_java_lang_System_registerNatives` 注册，其真实实现在 `src/share/vm/prims/jvm.cpp`

说实话，得补一下 C艹...

```cpp
JVM_ENTRY(void, JVM_ArrayCopy(JNIEnv *env, jclass ignored, jobject src, jint src_pos,
                               jobject dst, jint dst_pos, jint length))
  JVMWrapper("JVM_ArrayCopy");
  // Check if we have null pointers
  if (src == NULL || dst == NULL) {
    THROW(vmSymbols::java_lang_NullPointerException());
  }
  arrayOop s = arrayOop(JNIHandles::resolve_non_null(src));
  arrayOop d = arrayOop(JNIHandles::resolve_non_null(dst));
  assert(s->is_oop(), "JVM_ArrayCopy: src not an oop");
  assert(d->is_oop(), "JVM_ArrayCopy: dst not an oop");
  // Do copy
  s->klass()->copy_array(s, src_pos, d, dst_pos, length, thread);
JVM_END
```

可以看出，主要逻辑肯定是 `copy_array` 中（这里可能涉及到 `TypeArrayKlass（基础类型）/ObjArrayKlass`，我们看前者）

核心逻辑： 
```cpp

  ...
  // This is an attempt to make the copy_array fast.
  int l2es = log2_element_size();
  int ihs = array_header_in_bytes() / wordSize;
  char* src = (char*) ((oop*)s + ihs) + ((size_t)src_pos << l2es);
  char* dst = (char*) ((oop*)d + ihs) + ((size_t)dst_pos << l2es);
  Copy::conjoint_memory_atomic(src, dst, (size_t)length << l2es);
```

最后会发现调用了平台相关的 `pd_conjoint_bytes`，看了看 linux 下面的实现居然都直接汇编了，卧槽... 追到这儿暂停吧，真看不懂了[捂脸]

但到现在也差不多已知猜测是通过操作整块内存一起使用指针串联提高执行速度的。

##### get

get 流程与 put 几乎一致。即空数组作为 arrayCopy 的 dst，然后将 dst 作为结果返回。


#### MappedByteBuffer 解析

由于 DirectByteBuffer 继承了 MappedByteBuffer 且个人对其机制比较感兴趣，所以先介绍一下 MappedByteBuffer。这儿有一篇文章可以参考一下: [深入浅出MappedByteBuffer](https://www.jianshu.com/p/f90866dcbffc)。文章要点:

1. MappedByteBuffer 继承自 ByteBuffer，内部维护了一个逻辑地址 address。
2. 除了正常 READ/WRITE 还支持 copy_on_write（MapMode.PRIVATE）
3. 解析了 MappedByteBuffer 的创建过程

文中所包含的代码可以在 [FileChannelImpl.java](https://github.com/AdoptOpenJDK/openjdk-jdk11/blob/master/src/java.base/share/classes/sun/nio/ch/FileChannelImpl.java) 查看

创建主要核心代码 `Utils.newMappedByteBuffer` 主要是创建了一个 `DirectByteBuffer` （失望了吧？）

#### DirectByteBuffer 解析

即堆外内存。关于介绍这里有篇文章可以参考一下 [堆外内存之 DirectByteBuffer 详解](http://www.importnew.com/26334.html)。文章要点:

1. DirectByteBuffer 该类本身还是位于Java内存模型的堆中。堆内内存是 JVM 可以直接管控、操纵。而 DirectByteBuffer 中的 `unsafe.allocateMemory(size);` 是个一个 native 方法，这个方法分配的是堆外内存，通过 C 的 malloc 来进行分配的。分配的内存是系统本地的内存，并不在 Java 的内存中，也不属于 JVM 管控范围。
2. DirectByteBuffer 中的 `address` 表示分配的堆外内存地址。
3. 利用 PhantomReference 跟踪垃圾回收过程
4. 最后介绍了一些 DirectByteBuffer 使用的要点

还有笨神的文章[JVM源码分析之堆外内存完全解读](http://lovestblog.cn/blog/2015/05/12/direct-buffer/)。文章要点:

1. 区分广义堆外内存（jvm 本身在运行过程中分配的内存，codecache，jni 里分配的内存，DirectByteBuffer 分配的内存等等），狭义堆外内存（DirectByteBuffer）
2. `Bits.reserveMemory` 用以分配内存
3. 默认堆外内存实际是在 java.lang.System 初始化时候通过 native 方法 `Runtime.getRuntime().maxMemory()` 设置的（若不设置 `-Dsun.nio.MaxDirectMemorySize`）。
4. `Bits.reserveMemory` 过程中主动触发了一次 `System.gc()` 。堆外内存不会对 gc 造成什么影响(这里的 System.gc 除外)，但是堆外内存的回收其实依赖于我们的 gc 机制，gc 能通过操作 DirectByteBuffer 对象来间接操作对应的堆外内存了。
5. DirectByteBuffer 对象在创建的时候关联了一个 PhantomReference，其主要是用来跟踪对象何时被回收的，它不能影响 gc 决策，但是 gc 过程中如果发现某个对象除了只有 PhantomReference 引用它之外，并没有其他的地方引用它了，那将会把这个引用放到 java.lang.ref.Reference.pending 队列里，在 gc 完毕的时候通知 ReferenceHandler 这个守护线程去执行一些后置处理，而 DirectByteBuffer 关联的 PhantomReference 是 PhantomReference 的一个子类，在最终的处理里会通过 Unsafe 的 free 接口来释放 DirectByteBuffer 对应的堆外内存块。
6. 在通信阶段存在于 Heap 的内存最后都要 copy 一份到堆外，这样直接使用堆外更好
7. 堆外内存不会对 gc 造成影响，同时也不受 gc 影响
8. 对于需要频繁操作的内存，并且仅仅是临时存在一会的，都建议使用堆外内存，并且做成缓冲池，不断循环利用这块内存。
9. 堆外内存的回收不是被直接控制的，当然你可以通过别的一些途径，比如反射，直接使用 Unsafe 接口等，但是这些务必给你带来了一些烦恼。如果大面积使用迟早会发生内存泄露。

两篇文章都讲得足够好了，就不赘述了。

##### Cleaner 详解

先简单介绍一下 PhantomReference，所谓幻象引用。

1. 不能通过该引用去访问对象
2. 提供一种确保对象被 `finalize` 后做某些事的机制
3. 一般使用幻想应用模式：
    1. 指定一个 ReferenceQueue 作为引用队列
    2. 当 finalize(非空实现)被触发时，JVM 会将对象推入引用队列中

**PhantomReference Demo**

```java
public class PhantomReferenceTest {
    private static final ReferenceQueue rq = new ReferenceQueue();

    public static void main(String[] args) throws Throwable {
        Object t = new Object();
        PhantomReference pr = new PhantomReference(t, rq);
        t = null;
        System.gc();
        System.out.println("finalize: " + (rq.remove(1000) != null)); // finalize: true
    }
}
```

这个符合预期，但是若对象实现了 `finalizer` 就比较有意思了

```java
public class PhantomReferenceTest {
    private static final ReferenceQueue rq = new ReferenceQueue();

    private static class TestDemo {

        @Override
        protected void finalize() throws Throwable {
            System.out.println("test");
        }
    }

    public static void main(String[] args) throws Throwable {
        TestDemo t = new TestDemo();
        PhantomReference pr = new PhantomReference(t, rq);
        t = null;
        System.gc();
        Thread.sleep(100);
        System.out.println("finalize: " + (rq.remove(1000) != null)); // finalize: false
        System.gc();
        System.out.println("finalize: " + (rq.remove(1000) != null)); // finalize: true
    }
}
```

以下引用以下 《深入理解 Java 虚拟机》 书中内容

> 即使在可达性分析算法中不可达的对象，也并非是『非死不可』的，这时候它们暂时处于『缓刑』阶段，要真正宣告一个对象死亡，至少要经历两次标记过程：如果对象在进行可达性分析后发现没有与 GC Roots 相连接的引用链，那它将会被第一次标记并进行一次筛选，筛选的条件是此对象是否有必要执行 `finalize()` 方法。**当对象没有覆盖 `finalizer()` 方法，或者 `finalizer()` 方法已经被虚拟机调用过，虚拟机将这两种情况都视为『没有必要执行』。**

即上述之所以需要第二次 gc 才能触发引用队列的原因是对象在第一次时并未真正的死亡。

接下来是 Cleaner 的源码，其核心即是 `public class Cleaner extends PhantomReference<Object>`

其中 `clean()` 方法是核心清理代码，这里是在 `java.lang.ref.Reference` 初始化时创建了一个 `ReferenceHandler` 的守护线程进行调用。

而其调用的清理任务是来自于 DirectByteBuffer 的 Deallocator Runnable。主要记录了 `address` 内存地址信息，以及内存容量信息，然后使用 ` unsafe.freeMemory(address)` 释放该内存地址，最后再调用 `Bits.unreserveMemory` 来清理内存计数。 

关于 Cleaner 与 finalizer 最新的还有下面这些可以参考:

[Deprecate Object.finalize](https://bugs.openjdk.java.net/browse/JDK-8165641)
[java.lang.ref.Cleaner - an alternative to finalization](https://bugs.openjdk.java.net/browse/JDK-8138696)
[RFR 9: 8165641 : Deprecate Object.finalize](http://mail.openjdk.java.net/pipermail/core-libs-dev/2017-March/046650.html)

### ByteBuffer 与 ByteBuf

Netty 中的 ByteBuf 是由于 ByteBuffer 在使用上的困难，以及一些在网络 IO 上的常用操作的缺失决定了其存在性。

其特性参考 [官方 API](https://netty.io/4.1/api/index.html)

1. 可自定义 buffer type
2. composite buffer type 可以实现零拷贝
3. 动态长度 Buffer type
4. 不需要调用 filp() 在读写模式下转换
5. 通常比 ByteBuffer 快

接下来的文章，我们将深入 ByteBuf 源码进行分析。