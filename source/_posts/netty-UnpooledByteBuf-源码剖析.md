---
title: Netty-UnpooledByteBuf 源码剖析
date: 2018-11-17 18:13:28
tags:
    - NIO
    - Netty源码分析系列
categories: 源码分析
---

### 要点

1. UnpooledByteBufAllocator 的 Heap 与 Direct 的实现
2. Heap 内部实现使用了 byte[]
3. Direct 内部依托于 `PlatformDependent0` 的各种 native 方法
4. toLeakAwareBuffer 部分主要讨论了 Netty 是如何应对内存泄露，以及检测内存泄露跟踪的四个级别是如何实现的

<!-- more -->

**Demo**

```java
ByteBuf buf = Unpooled.buffer();
buf.writeBytes("test".getBytes());

byte[] readBytes = new byte[buf.readableBytes()];
buf.readBytes(readBytes);
System.out.println("read content: " + new String(readBytes));
```

> 入口 `Unpooled.buffer`

### UnpooledByteBufAllocator 的 Heap 部分解析

#### 构造器说明

```java
public UnpooledByteBufAllocator(boolean preferDirect, boolean disableLeakDetector, boolean tryNoCleaner) {
    super(preferDirect);
    this.disableLeakDetector = disableLeakDetector;
    noCleaner = tryNoCleaner && PlatformDependent.hasUnsafe()
            && PlatformDependent.hasDirectBufferNoCleanerConstructor();
}
```

1. preferDirect 设置为 true 时, allocator.buffer 会先尝试分配 direct buffer 而非 heap buffer （如果能分配 direct buffer）
2. disableLeakDetector 设置为 true 则表示该 allocator 将关闭 leak-detection. 这个用处在于用户仅想依靠 gc 来管理 direct buffer 而不希望显式释放内存。
3. tryNoCleaner 设置为 true 则表示我们将尝试去使用 PlatformDependent.allocateDirectNoCleaner(int) 去分配 direct buffer（如果其他条件允许）


#### buffer() 主要逻辑

由方法追踪，跟踪 heapBuffer 的逻辑则我们能找到稍微内容多一些的方法即下面的：

```java
public ByteBuf heapBuffer(int initialCapacity, int maxCapacity) {
    if (initialCapacity == 0 && maxCapacity == 0) {
        return emptyBuf;
    }
    validate(initialCapacity, maxCapacity); // 验证 initialCapacity 的合法性
    return newHeapBuffer(initialCapacity, maxCapacity); // core code
}
```

顺便一说，heapBuffer 其默认值（即我们之间调用 `buffer()` 结果）
```java
static final int DEFAULT_INITIAL_CAPACITY = 256;
static final int DEFAULT_MAX_CAPACITY = Integer.MAX_VALUE;
```

```java
protected ByteBuf newHeapBuffer(int initialCapacity, int maxCapacity) {
    return PlatformDependent.hasUnsafe() ? // 判断一下是否能使用 unsafe
            new InstrumentedUnpooledUnsafeHeapByteBuf(this, initialCapacity, maxCapacity) :
            new InstrumentedUnpooledHeapByteBuf(this, initialCapacity, maxCapacity);
}
```

如果可以使用 unsafe

```java
private static final class InstrumentedUnpooledUnsafeHeapByteBuf extends UnpooledUnsafeHeapByteBuf {
    InstrumentedUnpooledUnsafeHeapByteBuf(UnpooledByteBufAllocator alloc, int initialCapacity, int maxCapacity) {
        super(alloc, initialCapacity, maxCapacity);
    }
    ...
}
```

构造器追踪到 `UnpooledHeapByteBuf` 时候会发现一段这样的代码

```java
public UnpooledHeapByteBuf(ByteBufAllocator alloc, int initialCapacity, int maxCapacity) {
    super(maxCapacity); // 设置 maxCapacity 值

    checkNotNull(alloc, "alloc");

    if (initialCapacity > maxCapacity) {
        throw new IllegalArgumentException(String.format(
                "initialCapacity(%d) > maxCapacity(%d)", initialCapacity, maxCapacity));
    }

    this.alloc = alloc;
    setArray(allocateArray(initialCapacity)); // 单纯 setArray 逻辑即是将参数中的数组设置为 ByteBuf 的数组变量
    setIndex(0, 0); // 设置 readIndex 与 writeIndex 均为 0
}
```

这么一看其核心逻辑就在 allocateArray() 方法，而且可以确定的是 `InstrumentedUnpooledUnsafeHeapByteBuf` 与 `InstrumentedUnpooledHeapByteBuf` 的区别也仅在 **分配(allocateArray)** 与 **释放(freeArray)** 方法上，但是当你仔细看着两个类的代码时你会发现，他们的核心在于 `UnpooledUnsafeHeapByteBuf` 与  `UnpooledHeapByteBuf` 的 `allocateArray` 与 `freeArray` 中，这二者余下操作是针对具体 ByteBuf 内部字段的调整/监控.

`UnpooledHeapByteBuf` 过于简单，就是初始化一个数组而已 `new byte[]` 就不做介绍了，让我们主要来看看 `UnpooledUnsafeHeapByteBuf` 内部调用的 `PlatformDependent.allocateUninitializedArray(initialCapacity)`。

这个方法内部逻辑是

1. 判断分配大小是否超过 `UNINITIALIZED_ARRAY_ALLOCATION_THRESHOLD` （默认 1024，可设置，对于 java9 之前或者是 java9 不给开 Unsafe 而言这个值就是 -1）
2. 若分配 size 大小小于 `UNINITIALIZED_ARRAY_ALLOCATION_THRESHOLD` 则直接创建数组了事（new byte[]）（即**对 java9 之前及不支持 Unsafe 而言，都是创建数组**）
3. 若分配 size 太大，则调用 ` PlatformDependent0.allocateUninitializedArray(size)`

```java
static byte[] allocateUninitializedArray(int size) {
    try {
        return (byte[]) ALLOCATE_ARRAY_METHOD.invoke(INTERNAL_UNSAFE, byte.class, size);
    } catch (IllegalAccessException e) {
        throw new Error(e);
    } catch (InvocationTargetException e) {
        throw new Error(e);
    }
}
```

ALLOCATE_ARRAY_METHOD 是针对 Java9+ 的 `jdk.internal.misc.Unsafe.allocateUninitializedArray` 创建 Method, 从名称来看应该是使用了底层系统调用创建了一个未初始化的数组，一个针对性能的优化。其代码

```java
/**
 * Allocates an array of a given type, but does not do zeroing.
 * <p>
 * This method should only be used in the very rare cases where a high-performance code
 * overwrites the destination array completely, and compilers cannot assist in zeroing elimination.
 * In an overwhelming majority of cases, a normal Java allocation should be used instead.
 * <p>
 * Users of this method are <b>required</b> to overwrite the initial (garbage) array contents
 * before allowing untrusted code, or code in other threads, to observe the reference
 * to the newly allocated array. In addition, the publication of the array reference must be
 * safe according to the Java Memory Model requirements.
 * <p>
 * The safest approach to deal with an uninitialized array is to keep the reference to it in local
 * variable at least until the initialization is complete, and then publish it <b>once</b>, either
 * by writing it to a <em>volatile</em> field, or storing it into a <em>final</em> field in constructor,
 * or issuing a {@link #storeFence} before publishing the reference.
 * <p>
 * @implnote This method can only allocate primitive arrays, to avoid garbage reference
 * elements that could break heap integrity.
 *
 * @param componentType array component type to allocate
 * @param length array size to allocate
 * @throws IllegalArgumentException if component type is null, or not a primitive class;
 *                                  or the length is negative
 */
public Object allocateUninitializedArray(Class<?> componentType, int length) {
   if (componentType == null) {
       throw new IllegalArgumentException("Component type is null");
   }
   if (!componentType.isPrimitive()) {
       throw new IllegalArgumentException("Component type is not primitive");
   }
   if (length < 0) {
       throw new IllegalArgumentException("Negative length");
   }
   return allocateUninitializedArray0(componentType, length);
}

@HotSpotIntrinsicCandidate
private Object allocateUninitializedArray0(Class<?> componentType, int length) {
   // These fallbacks provide zeroed arrays, but intrinsic is not required to
   // return the zeroed arrays.
   if (componentType == byte.class)    return new byte[length];
   if (componentType == boolean.class) return new boolean[length];
   if (componentType == short.class)   return new short[length];
   if (componentType == char.class)    return new char[length];
   if (componentType == int.class)     return new int[length];
   if (componentType == float.class)   return new float[length];
   if (componentType == long.class)    return new long[length];
   if (componentType == double.class)  return new double[length];
   return null;
}
```

看了这儿我也不太能理解为什么会绕弯选择这个优化[捂脸]，可能是期望一个性能优化的承诺？

就前面的理解可知，我们此时获取到的 buffer 通常是 `InstrumentedUnpooledUnsafeHeapByteBuf extends UnpooledUnsafeHeapByteBuf` /`InstrumentedUnpooledHeapByteBuf extends UnpooledHeapByteBuf`。而 `UnpooledUnsafeHeapByteBuf extends UnpooledHeapByteBuf`，所以接下来我们解析的顺序先从 `UnpooledHeapByteBuf` 开始，然后再看 `UnpooledUnsafeHeapByteBuf` 有什么不一样的地方.


#### writeByte 逻辑

在 `AbstractByteBuf` 中

```java
public ByteBuf writeBytes(byte[] src, int srcIndex, int length) {
    ensureWritable(length); // 这里就是自动调节长度的位置了
    setBytes(writerIndex, src, srcIndex, length); // 被 UnpooledHeapByteBuf 重载
    writerIndex += length; // 增加 writerIndex 索引值
    return this;
}
```

`ensureWriteable0` // 核心逻辑

```java
final void ensureWritable0(int minWritableBytes) {
    ensureAccessible(); // 明确是否可访问
    if (minWritableBytes <= writableBytes()) {
        return;
    }
    if (checkBounds) { // 用户 jvm 配置是否需要 check bound，默认为 true
        if (minWritableBytes > maxCapacity - writerIndex) { // maxCapacity 前文提过默认 int 上限
            throw new IndexOutOfBoundsException(String.format(
                    "writerIndex(%d) + minWritableBytes(%d) exceeds maxCapacity(%d): %s",
                    writerIndex, minWritableBytes, maxCapacity, this));
        }
    }

    // Normalize the current capacity to the power of 2.
    int newCapacity = alloc().calculateNewCapacity(writerIndex + minWritableBytes, maxCapacity);

    // Adjust to the new capacity.
    capacity(newCapacity);
}
```

> set minNewCapacity = writerIndex + minWritableBytes;

calculateNewCapacity 增加的逻辑有点意思，它在预设值 `threshold = 4M` 的场景下 ，有这几种情况:

1. minWritableBytes == threshold 则直接返回 threshold
2. minWritableBytes > threshold 则  
```java
if (minNewCapacity > threshold) {
    int newCapacity = minNewCapacity / threshold * threshold; // threshold 倍数级增加（向下取整）
    if (newCapacity > maxCapacity - threshold) { // 如果是最后一个 threshold 则将最大值空间分配返回
        newCapacity = maxCapacity;
    } else {
        newCapacity += threshold; // 否则增加一个 4M 即可
    }
    return newCapacity;
}
```
3. 最后一种情况即 minWritableBytes 小于 threshold，此时  
```java
// Not over threshold. Double up to 4 MiB, starting from 64.
int newCapacity = 64; // 从 64 开始 double 直到大于 minNewCapacity
while (newCapacity < minNewCapacity) {
    newCapacity <<= 1;
}

return Math.min(newCapacity, maxCapacity); // 如果 maxCapacity < threshold 这儿就有用了
```

接下来我们看看被重载的 `setBytes` 方法（其实 `UnpooledHeapByteBuf` 与 `UnpooledUnsafeHeapByteBuf` 差异点也在这里）

`UnpooledHeapByteBuf` 非常简单，copy 一把跑路

```java
public ByteBuf setBytes(int index, byte[] src, int srcIndex, int length) {
    checkSrcIndex(index, length, srcIndex, src.length);
    System.arraycopy(src, srcIndex, array, index, length);
    return this;
}
```

`UnpooledUnsafeHeapByteBuf` 的就需要稍微深入一下了

```java
public ByteBuf setByte(int index, int value) {
    checkIndex(index);
    _setByte(index, value);
    return this;
}

protected void _setByte(int index, int value) {
    UnsafeByteBufUtil.setByte(array, index, value);
}
...
// 最后起作用的，此时 data 及是 array 即原 buffer 的存储数组，第二个参数是其插入值的 index，value 是需要插入的值
UNSAFE.putByte(data, BYTE_ARRAY_BASE_OFFSET + index, value);
```


### UnpooledByteBufAllocator 的 Direct 部分解析

我们经过上面的铺垫，通过 `directBuffer()` 方法可以直接追踪到 `UnpooledByteBufAllocator.newDirectBuffer()` 方法

```java
protected ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity) {
    final ByteBuf buf;
    if (PlatformDependent.hasUnsafe()) {
        buf = noCleaner ? new InstrumentedUnpooledUnsafeNoCleanerDirectByteBuf(this, initialCapacity, maxCapacity) :
                new InstrumentedUnpooledUnsafeDirectByteBuf(this, initialCapacity, maxCapacity);
    } else {
        buf = new InstrumentedUnpooledDirectByteBuf(this, initialCapacity, maxCapacity);
    }
    return disableLeakDetector ? buf : toLeakAwareBuffer(buf);
}
```

在此之前我们需要回忆一下 noCleaner 与 disableLeakDetector 两个参数

>disableLeakDetector 设置为 true 则表示该 allocator 将关闭 leak-detection. 这个用处在于用户仅想依靠 gc 来管理 direct buffer 而不希望显式释放内存。 
>
>tryNoCleaner 设置为 true 则表示我们将尝试去使用 PlatformDependent.allocateDirectNoCleaner(int) 去分配 direct buffer（如果其他条件允许）
>
>noCleaner = tryNoCleaner && PlatformDependent.hasUnsafe() && PlatformDependent.hasDirectBufferNoCleanerConstructor();



看来接下来需要介绍

1. InstrumentedUnpooledUnsafeNoCleanerDirectByteBuf、InstrumentedUnpooledUnsafeDirectByteBuf、 InstrumentedUnpooledDirectByteBuf 之间的异同
2. toLeakAwareBuffer(buf) 的用途

先说三个 ByteBuf。

这三个其实都只是添加了一些监控功能的类，具体还需要查看相对应的父类：UnpooledUnsafeNoCleanerDirectByteBuf, UnpooledUnsafeDirectByteBuf, UnpooledDirectByteBuf。其关系：

![6152fbef5328a4f62ec298ab588b5a86.png](evernotecid://030B6737-EB59-4FA5-B258-B85DAAD049D0/appyinxiangcom/7066058/ENResource/p7345)

先说一下 UnpooledUnsafeNoCleanerDirectByteBuf, UnpooledUnsafeDirectByteBuf 两位的区别

NoCleaner 其实就是指 ByteBuffer 在 allocateDirect 时候会使用 `PlatformDependent.allocateDirectNoCleaner` 而 freeDirect 时候会使用 `PlatformDependent.freeDirectNoCleaner`。而后者则是使用 `ByteBuffer.allocateDirect` 与 `PlatformDependent.freeDirectBuffer`

那就来看看这两对方法的两两比较即可

#### Unsafe 的 `allocateDirect` 组比较

```java
public static ByteBuffer allocateDirectNoCleaner(int capacity) {
    assert USE_DIRECT_BUFFER_NO_CLEANER;

    incrementMemoryCounter(capacity); // 内存使用记录增加 capacity
    try {
        return PlatformDependent0.allocateDirectNoCleaner(capacity);
    } catch (Throwable e) {
        decrementMemoryCounter(capacity); // 若分配失败则内存使用记录减少对应的值
        throwException(e);
        return null;
    }
}
```

`PlatformDependent0.allocateDirectNoCleaner` 

```java
static ByteBuffer allocateDirectNoCleaner(int capacity) {
    // Calling malloc with capacity of 0 may return a null ptr or a memory address that can be used.
    // Just use 1 to make it safe to use in all cases:
    // See: http://pubs.opengroup.org/onlinepubs/009695399/functions/malloc.html
    return newDirectBuffer(UNSAFE.allocateMemory(Math.max(1, capacity)), capacity); // 先分配内存并获取内存地址，再调用 newDirectBuffer
}

static ByteBuffer newDirectBuffer(long address, int capacity) {
    ObjectUtil.checkPositiveOrZero(capacity, "capacity");

    try {
        return (ByteBuffer) DIRECT_BUFFER_CONSTRUCTOR.newInstance(address, capacity); // 通过 DirectByteBuffer 暴露了 address 的构造器来创建 ByteBuffer
    } catch (Throwable cause) {
        // Not expected to ever throw!
        if (cause instanceof Error) {
            throw (Error) cause;
        }
        throw new Error(cause);
    }
}
```


而后者，直接创建了一个 DirectByteBuffer，借助 JDK 的 DirectByteBuffer 本身的垃圾回收进行内存管理（可参考[Netty 中的 ByteBuf — ByteBuffer 与 ByteBuf]()）

```java
public static ByteBuffer allocateDirect(int capacity) {
    return new DirectByteBuffer(capacity);
}
```

##### Unsafe 的 `freeDirect` 组比较 

`UnpooledUnsafeNoCleanerDirectByteBuf.freeDirect` 调用  `PlatformDependent`

```java
public static void freeDirectNoCleaner(ByteBuffer buffer) {
    assert USE_DIRECT_BUFFER_NO_CLEANER;

    int capacity = buffer.capacity();
    PlatformDependent0.freeMemory(PlatformDependent0.directBufferAddress(buffer)); // 释放内存
    decrementMemoryCounter(capacity); // 降低记录总内存数量
}
```

`PlatformDependent0.directBufferAddress` 是获取其 buffer 的内存地址信息，这个信息获取逻辑是通过获取 `DirectByteBuffer` 的 address 字段信息，而这个字段是在 PlatformDependent0 初始化时通过反射获取的（其中验证 Buffer 是否是 DirecitByteBuffer 使用了 Unsafe 获取值来判断）。

当获取到了内存地址之后自然就可以通过 Unsafe.freeMemory(address) 来释放内存了。

`UnpooledUnsafeDirectByteBuf.freeDirect` 调用 `PlatformDependent`

```java
public static void freeDirectBuffer(ByteBuffer buffer) {
    CLEANER.freeDirectBuffer(buffer);
}
```

CLEANER 是一个针对 Java9+ 版本的类（`io.netty.util.internal.CleanerJava9`），Java9 为空操作。CleanerJava9 主要起用的是 `sun.misc.Unsafe.invokeCleaner`. 此方法即是调用 `DirectByteBuffer` 的 Cleaner，这是一个利用 PhantomReference 来管理内存的手段。

`UnpooledDirectByteBuf` 就相对来说毕竟简单，`allocate` 还是直接调用 JDK 的 `DirectByteBuffer` 相关 API，`free` 还是同样是使用 `Cleaner`

### `toLeakAwareBuffer` 

```java
protected static ByteBuf toLeakAwareBuffer(ByteBuf buf) {
    ResourceLeakTracker<ByteBuf> leak;
    switch (ResourceLeakDetector.getLevel()) {
        case SIMPLE:
            leak = AbstractByteBuf.leakDetector.track(buf);
            if (leak != null) {
                buf = new SimpleLeakAwareByteBuf(buf, leak);
            }
            break;
        case ADVANCED:
        case PARANOID:
            leak = AbstractByteBuf.leakDetector.track(buf);
            if (leak != null) {
                buf = new AdvancedLeakAwareByteBuf(buf, leak);
            }
            break;
        default:
            break;
    }
    return buf;
}
```

这里涉及到 Netty 的一个内存泄露监控措施。

#### ResourceLeakDetector 介绍

主要参考文章: [Netty 的资源泄露探测机制](https://ylgrgyq.github.io/2017/11/11/netty-resource-leack-detector/)

1. ResourceLeakDetector 能对占用资源的对象进行监控，如果对象被 GC 之前没有主动释放资源，则 ResourceLeakDetector 会发现这个泄露，并记录日志
2. ResourceLeakDetector 主要用于 ByteBuf（Heap And Direct） 的记录，若未调用 release 时候可以通过日志告知出现泄漏。


Level 说明:

- SIMPLE: 默认级别，简单抽样，仅报告是否存在泄漏  
- PARANOID: 所有均采样，报告对象最后一次访问的地址，消耗极高，只测试使用  
- ADVANCED: 抽样，报告对象最后一次访问的地址，消耗较高
- DISABLED: 关闭

ResourceLeakDetector 准备流程即 `AbstractByteBuf.leakDetector.track()`

```java
static final ResourceLeakDetector<ByteBuf> leakDetector =
        ResourceLeakDetectorFactory.instance().newResourceLeakDetector(ByteBuf.class);
```

实际调用 `DefaultResourceLeakDetectorFactory.newResourceLeakDetector` 方法（非 deprecation），传入时候

```java
public <T> ResourceLeakDetector<T> newResourceLeakDetector(Class<T> resource, int samplingInterval) {
    if (customClassConstructor != null) {
        try {
            @SuppressWarnings("unchecked")
            ResourceLeakDetector<T> leakDetector =
                    (ResourceLeakDetector<T>) customClassConstructor.newInstance(resource, samplingInterval);
            logger.debug("Loaded custom ResourceLeakDetector: {}",
                    customClassConstructor.getDeclaringClass().getName());
            return leakDetector;
        } catch (Throwable t) {
            logger.error(
                    "Could not load custom resource leak detector provided: {} with the given resource: {}",
                    customClassConstructor.getDeclaringClass().getName(), resource, t);
        }
    }

    ResourceLeakDetector<T> resourceLeakDetector = new ResourceLeakDetector<T>(resource, samplingInterval);
    logger.debug("Loaded default ResourceLeakDetector: {}", resourceLeakDetector);
    return resourceLeakDetector;
}
```

1. check 是否有通过 `io.netty.customResourceLeakDetector` 引入的自定义 ResourceLeakDetector，若有则创建该类型的 ResourceLeakDetector
2. samplingInterval 指代采样间隔时间，默认 128 millsSecends，可通过 `io.netty.leakDetection.samplingInterval` 设置


##### `track()`

创建 `ResourceLeakTracker`，该类将会在 Resource 被 deallocate 后自动调用 close() 方法，从其创建方法中我们能看出 PARANOID 模式与其他的区别

部分代码

```java
if (level.ordinal() < Level.PARANOID.ordinal()) {
    if ((PlatformDependent.threadLocalRandom().nextInt(samplingInterval)) == 0) { // 使用间隔抽样
        reportLeak();
        return new DefaultResourceLeak(obj, refQueue, allLeaks);
    }
    return null;
}
reportLeak();
return new DefaultResourceLeak(obj, refQueue, allLeaks);
```

这里抽样的逻辑是，随机值实现是 0 - samplingInterval 均态分布的，即获取到 0 的时候则为使用间隔是符合预期的（调用大量时）。

先说说 DefaultResourceLeak 的实现

DefaultResourceLeak 是继承的 WeakReference，其 ResourceLeakDetector 初始化时候会持有一个 ReferenceQueue(refQ)，初始化 DefaultResourceLeak 时会将其队列加入构造器，这样 refQ 就与该 DefaultResourceLeak 绑定，在对象被 `close()` 的时候执行 `WeakReference.clear` 让弱引用不因为 gc 的原因进入 ReferenceQueue。**若是没有调用 `close()` 随着 gc 将其引用的对象回收，其 WeakReference 也会被回收，其 ref 将会进入 refQ**，此时队列中存在的就是未调用 `close()` 的 Resource 了。

接下来我们看看 `reportLeak()` 干了啥

```java
private void reportLeak() {
    if (!logger.isErrorEnabled()) { // 若没有答应日志开口，就清理队列即可
        clearRefQueue();
        return;
    }

    // Detect and report previous leaks.
    for (;;) {
        @SuppressWarnings("unchecked")
        DefaultResourceLeak ref = (DefaultResourceLeak) refQueue.poll(); // 获取每一个被 clear 的 DefaultResourceLeak
        if (ref == null) { // 队列中没有对象，完成
            break;
        }

        if (!ref.dispose()) { // 返回值表示已被处理，应该是个 double check
            continue;
        }

        String records = ref.toString();
        if (reportedLeaks.putIfAbsent(records, Boolean.TRUE) == null) { // 打印记录的信息
            if (records.isEmpty()) { // 有无详情
                reportUntracedLeak(resourceType);
            } else {
                reportTracedLeak(resourceType, records);
            }
        }
    }
}
```

我们看 `DefaultResourceLeak` 会知道它的信息是会由 `record` 方法记录生成，这里我们记住即可。

在回到 `AbstractByteBufAllocator.toLeakAwareBuffer` （我们的起点）。这里会创建 SimpleLeakAwareByteBuf 或者 AdvancedLeakAwareByteBuf，其在构建起中会关联上面创建的 DefaultResourceLeak，而其中的 `release()` 方法将会调用 `ResourceLeak.close()` 方法来执行上面所说的逻辑，其中 AdvancedLeakAwareByteBuf 几乎对每一个 ByteBuf 的读写操作都进行了 `DefaultResourceLeak.record()` 的调用。
