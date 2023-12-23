---
title: Netty中的ByteBuf-Pooled与内存管理
date: 2018-12-10 21:11:33
tags:
    - NIO
    - Netty源码分析系列
categories: 源码分析
---

**Demo**

```java
ByteBuf buf = PooledByteBufAllocator.DEFAULT.buffer();
buf.writeBytes("test".getBytes());

byte[] readBytes = new byte[buf.readableBytes()];
buf.readBytes(readBytes);
System.out.println("read content: " + new String(readBytes));
```

直接看 `PooledByteBufAllocator.newHeapBuffer(int initialCapacity, int maxCapacity)`

```java
protected ByteBuf newHeapBuffer(int initialCapacity, int maxCapacity) {
    PoolThreadCache cache = threadCache.get();
    PoolArena<byte[]> heapArena = cache.heapArena;

    final ByteBuf buf;
    if (heapArena != null) {
        buf = heapArena.allocate(cache, initialCapacity, maxCapacity);
    } else {
        buf = PlatformDependent.hasUnsafe() ?
                new UnpooledUnsafeHeapByteBuf(this, initialCapacity, maxCapacity) :
                new UnpooledHeapByteBuf(this, initialCapacity, maxCapacity);
    }

    return toLeakAwareBuffer(buf);
}
```

<!-- more -->

## Netty 的内存管理 - Arena

从 `heapArena.allocate` 中我们可以窥见整个 Netty 的内存管理机制 - Arean

参考文章：

- [深入浅出Netty内存管理：PoolChunk](http://blog.jobbole.com/106001/)
- [深入浅出Netty内存管理：PoolSubpage](http://blog.jobbole.com/106037/)
- [深入浅出Netty内存管理：PoolChunkList](http://blog.jobbole.com/106085/)
- [深入浅出Netty内存管理：PoolArena](http://blog.jobbole.com/106344/)
- [自顶向下深入分析Netty（十）--JEMalloc分配算法](https://www.jianshu.com/p/15304cd63175)
- [自顶向下深入分析Netty（十）--PoolThreadCache](https://www.jianshu.com/p/9177b7dabd37)

### allocate 流程

保留主流程，略过 cache 的代码：

```java
private void allocate(PoolThreadCache cache, PooledByteBuf<T> buf, final int reqCapacity) {
    final int normCapacity = normalizeCapacity(reqCapacity); // 将需要申请的容量格式为 2^N
    if (isTinyOrSmall(normCapacity)) { // capacity < pageSize。需要容量小于 chunk 每页数量，默认 8k
        ...
        if (tiny) { // < 512
            // 将分配区域转移到 tinySubpagePools 中
        } else {
            // 将分配区域转移到 smallSubpagePools 中
        }

        final PoolSubpage<T> head = table[tableIdx]; // 获取要分配的具体 PoolSubpage

        /**
         * Synchronize on the head. This is needed as {@link PoolChunk#allocateSubpage(int)} and
         * {@link PoolChunk#free(long)} may modify the doubly linked list as well.
         */
        synchronized (head) {
            final PoolSubpage<T> s = head.next;
            if (s != head) { // 如果此时 subpage 已经被分配过内存了执行下文，如果只是初始化过，则跳过该分支
                long handle = s.allocate(); // 获取分配的具体内存位置，具体可看上面参考文章中关于 PoolChunk 中的 PoolSubpage 描述
                s.chunk.initBufWithSubpage(buf, null, handle, reqCapacity); // 初始化 PoolByteBuf 说明其位置被分配到该区域，但此时尚未分配内存
                return;
            }
        }
        synchronized (this) { // 如果此时 subpage 无内存分配记录则从 chunk 中分配空间
            allocateNormal(buf, reqCapacity, normCapacity);
        }
        return;
    }
    if (normCapacity <= chunkSize) { // 分配空间大于 page 空间但是小于 chunkSize 空间，chunkSize 默认 16M
        synchronized (this) {
            allocateNormal(buf, reqCapacity, normCapacity); // 分配 Chunk
        }
    } else {
        // Huge allocations are never served via the cache so just call allocateHuge
        allocateHuge(buf, reqCapacity); // 超过 16 M 的分配方法
    }
}
```

我们通过不同的内存需求将出现的几个存储组件 `PoolArena`，`PoolSubpage`，`PoolChunk` 与 `PoolChunkList` 串联起来说明他们的关系与在 Netty 内存分配中所起到的作用

#### 小空间存储

Netty 在应对小内存（默认小于 8K）空间时候常会选择使用 PoolSubpage，就算在上述描述过程中 PoolArena 已经没有了 PoolSubpage 的空间而转到了 PoolChunk，但 PoolChunk 里实际也在维护着一个 PoolSubpage 数组用于应对这种情景。

在 PoolArena 中有两个 PoolSubpage 数组，一个是 tiny 用于分配内存小于 512 字节的空间，另外则负责 512 - 8192 字节的空间内存分配。

当决定完分配到哪个 PoolSubpage 数组之后需要去决定分配在哪个具体的 PoolSubpage 中，这时候 tiny 对于 512 的所有大小做了一个集体映射到 32 位数组的操作（32 也是 tinySubpagePools 的大小）。而 smallSubpagePools 的默认数组大小为 `pageShifts - 9` 即 3。2^8 - 2^12 分为三个区域存储。

这个时候对于新分配的 PoolSubpage，它的 head.next == head，会跳入 PoolChunk 分配 PoolSubpage 内存这时候涉及到 `allocateNormal`。

```java
private void allocateNormal(PooledByteBuf<T> buf, int reqCapacity, int normCapacity) {
    if (q050.allocate(buf, reqCapacity, normCapacity) || q025.allocate(buf, reqCapacity, normCapacity) ||
        q000.allocate(buf, reqCapacity, normCapacity) || qInit.allocate(buf, reqCapacity, normCapacity) ||
        q075.allocate(buf, reqCapacity, normCapacity)) {
        return;
    } // 这一串 allocate 一律通过不了

    // Add a new chunk.
    PoolChunk<T> c = newChunk(pageSize, maxOrder, pageShifts, chunkSize);
    boolean success = c.allocate(buf, reqCapacity, normCapacity);  // 真正分配内存的地方
    qInit.add(c); // 将分配的 PoolChunk 加入 qInit 链表管理，下文有提及 PoolChunkList 
}
```

对于 PoolChunk.allocate 我们直接摘取它对于 PoolSubpage 内存分配的部分来说明

```java
boolean allocate(PooledByteBuf<T> buf, int reqCapacity, int normCapacity) {
    final long handle;
    if ((normCapacity & subpageOverflowMask) != 0) { // >= pageSize
        handle =  allocateRun(normCapacity); // 这个是正常分配 Chunk 大内存的分支
    } else {
        handle = allocateSubpage(normCapacity); // 这个是我们此刻需要的分支
    }
    ...
}
```

这里有必要介绍一下 PoolChunk 的内存分布构造了。

PoolChunk 内部通过 memoryMap 数组维护了一颗平衡二叉树作为管理底层内存分布及回收的标记位，所有的子节点管理的内存也属于其父节点。

![3dd82dc68a0a48cf6a18b2deae1b01c7.jpeg](evernotecid://030B6737-EB59-4FA5-B258-B85DAAD049D0/appyinxiangcom/7066058/ENResource/p7346)

> poolChunk默认由2048个page组成，一个page默认大小为8k，图中节点的值为在数组memoryMap的下标。
> 
> 1. 如果需要分配大小8k的内存，则只需要在第11层，找到第一个可用节点即可。
> 2. 如果需要分配大小16k的内存，则只需要在第10层，找到第一个可用节点即可。
> 3. 如果节点1024存在一个已经被分配的子节点2048，则该节点不能被分配，如需要分配大小16k的内存，这个时候节点2048已被分配，节点2049未被分配，就不能直接分配节点1024，因为该节点目前只剩下8k内存。

这和 PoolSubpage 有什么关系呢？PoolSubpage 分配的内存即是 PoolChunk 的叶子节点标记。

```java
/**
 * Create / initialize a new PoolSubpage of normCapacity
 * Any PoolSubpage created / initialized here is added to subpage pool in the PoolArena that owns this PoolChunk
 *
 * @param normCapacity normalized capacity
 * @return index in memoryMap
 */
private long allocateSubpage(int normCapacity) {
    // Obtain the head of the PoolSubPage pool that is owned by the PoolArena and synchronize on it.
    // This is need as we may add it back and so alter the linked-list structure.
    PoolSubpage<T> head = arena.findSubpagePoolHead(normCapacity); // PoolArena 来管理 PoolChunk 和 PoolSubpage 用于查找合适的 PoolSubpage（查找逻辑就是之前描述过的 tiny 与 small 查找逻辑），找到合适的上一个 Subpage 节点（head，因为进入这而均是没有分配到内存的 PoolSubpage）
    int d = maxOrder; // subpages are only be allocated from pages i.e., leaves
    synchronized (head) {
        int id = allocateNode(d); // 试图在 chunk 中分配节点
        if (id < 0) {
            return id;
        }

        final PoolSubpage<T>[] subpages = this.subpages;
        final int pageSize = this.pageSize;

        freeBytes -= pageSize; // 记录 Chunk 可用内存

        int subpageIdx = subpageIdx(id); // 这里涉及到 PoolChunk 中对于 subpages 的数量初始化为 memoryMap 的叶子数。查询映射返回的叶子节点
        PoolSubpage<T> subpage = subpages[subpageIdx]; // 获取对应叶子节点上的 PoolSubpage
        if (subpage == null) { // 叶子节点暂无分配内存，则创建 PoolSubpage 并初始化
            subpage = new PoolSubpage<T>(head, this, id, runOffset(id), pageSize, normCapacity);
            subpages[subpageIdx] = subpage;
        } else { // 叶子节点已经存在，则初始化叶子节点，为什么会存在这个分支？PoolSubpage 为一个链表，若 PoolChunk 某个叶子节点已经分配过了，只需要给链表加入一个节点即可
            subpage.init(head, normCapacity);
        }
        return subpage.allocate(); // 分配内存标识并返回标识（handle，用于记录 chunk 和内部的偏移）
    }
}
```

##### `subpage.init()`

这里要开始说是 PageSubpage 是怎么做的标识了

在创建 PoolSubpage 时候会有个专用于记录是否有内存暂用的字段 `bitmap = new long[pageSize >>> 10]; // pageSize / 16 / 64`，考虑到 pageSize 默认是 8k，则 bitmap 默认长度则为 8，为什么 8 个 long 就足够了呢？因为 subpage 中允许分配的最小内存单元是 16，而 long 有 64 位标志位，则 8196 / 16 / 64 即可描述所有内存段。

接下来就是 init 阶段

```java
void init(PoolSubpage<T> head, int elemSize) {
    doNotDestroy = true;
    this.elemSize = elemSize;
    if (elemSize != 0) {
        maxNumElems = numAvail = pageSize / elemSize;
        nextAvail = 0;
        bitmapLength = maxNumElems >>> 6;
        if ((maxNumElems & 63) != 0) {
            bitmapLength ++;
        }

        for (int i = 0; i < bitmapLength; i ++) {
            bitmap[i] = 0;
        }
    }
    addToPool(head); // 将初始化的 subpage 加入列表中
}
```

下面举两个实例

1. 申请 4096 的内存
    1. maxNumElems 与 numAvali 为 2，说明 1 个 page 被拆分为 2 个
    2. bitmapLength = maxNumElems >>> 6 = 0 && maxNnmElems & 63 != 0 说明此时 bitmapLength = 1
    3. 此时则说明只需要一个 long 就能保存两个内存端的状态了
2. 申请 32 的内存
    1. maxNumElems 与 numAvali 为 256，说明 1 个 page 被拆分为 256 个内存段
    2. bitmapLength = maxNumElems >>> 6 = 4 && maxNnmElems & 63 == 0 说明此时 bitmapLength = 4
    3. 此时则说明需要 4 个 long 来描述 256 个内存段状态

下面转而看向内存如何分配

##### `subpage.allocate()`

```java
/**
 * Returns the bitmap index of the subpage allocation.
 */
long allocate() {
    if (elemSize == 0) {
        return toHandle(0);
    }

    if (numAvail == 0 || !doNotDestroy) {
        return -1;
    }

    final int bitmapIdx = getNextAvail(); // 获取可分配下个内存段的 bitmapIdx
    int q = bitmapIdx >>> 6; // 这里跟初始步骤计算思路一致，为了计算映射出 bitmap 数组下标，用来描述 bitmapIdx 的内存状态
    int r = bitmapIdx & 63; // 得到一个小于 64 的数 r
    assert (bitmap[q] >>> r & 1) == 0;
    bitmap[q] |= 1L << r; // 标记 bitmap[q] 中 r 位置为已分配内存

    if (-- numAvail == 0) 
        removeFromPool();
    }

    return toHandle(bitmapIdx); // 返回内存段标识符 handle
}
```

举例说明

若是需要分配 128 的内存，则 bitmap 被分拆为 64 个内存段 (8196 / 128) ，只需要 1 个 long(64位)即可描述，此时 bitmap 只会用到第一个元素, getNextAvail 通过位运算得到 long 中描述内存段状态的值，然后通过 bitmap[q] |= 1L << r 将该位数值置为 1 表示内存已被占用。

##### 初始化 buf

回到 `PoolChunk.allocate()`

```java
boolean allocate(PooledByteBuf<T> buf, int reqCapacity, int normCapacity) {
    final long handle;
    
    ... // 分配 Chunk or Subpage 逻辑
    
    ByteBuffer nioBuffer = cachedNioBuffers != null ? cachedNioBuffers.pollLast() : null;
    initBuf(buf, nioBuffer, handle, reqCapacity); 
    return true;
}
```

**initBuf()** 

initBuf 中判断传入的 handle 表示的内存是否被初始化过，若没有则直接不计算偏移量使用，若初始化过需要根据 bitmap 计算出在数组中的偏移值来进行初始化（这段代码涉及太复杂了[捂脸]，而且不是很能理解 tmpNioBuf 在哪个地方使用了）。真正存储使用的是 PoolChunk 中 memory 字段，而在 HeapByteBuf 中创建的就是 byte 数组。

此时我们回到 PoolArena.allocate() 再看若 tiny / small 的 subpage 数组中已经存在可分配的内存的话，就会显得比较简单了。

若 PoolSubpage 其实已经分配过内存了，其实此时它就会进入下面的 if 模块

```java
if (s != head) {
    long handle = s.allocate(); // 获取分配的具体内存位置 handle
    s.chunk.initBufWithSubpage(buf, null, handle, reqCapacity); // 初始化 PoolByteBuf 说明其位置被分配到该区域及初始化 Subpage 中 byte[] 偏移量
    return;
}
```

#### 大于 PageSize 空间

```java
private void allocate(PoolThreadCache cache, PooledByteBuf<T> buf, final int reqCapacity) {
    ...
    if (normCapacity <= chunkSize) {
        ... // 缓存
        synchronized (this) {
            allocateNormal(buf, reqCapacity, normCapacity);
            ++allocationsNormal;
        }
    } else {
        ...
    }
}
```

这里实际与上述小容量分配区别在于在 `PoolChunk.allocate()` 中会进入 `allocateRun()` 分支

```java
boolean allocate(PooledByteBuf<T> buf, int reqCapacity, int normCapacity) {
    final long handle;
    if ((normCapacity & subpageOverflowMask) != 0) { // >= pageSize
        handle =  allocateRun(normCapacity); // 这次进入这儿
    } else {
        handle = allocateSubpage(normCapacity); // 小容量内存分配进入
    }

    if (handle < 0) {
        return false;
    }
    ByteBuffer nioBuffer = cachedNioBuffers != null ? cachedNioBuffers.pollLast() : null;
    initBuf(buf, nioBuffer, handle, reqCapacity);
    return true;
}
```

实际上 allocateRun 里面逻辑就很简单了（在有上文铺垫情况下），在二叉树中找到可分配的节点，记录已分配，返回 handle。余下的流程与之前均相同。


#### 大于 ChunkSize 空间

`PoolArena.allocateHuge` 简单粗暴了，申请 Unpooled 不入池。

### allocate 流程中的 PoolThreadCache

PoolThreadCache 是从 PoolThreadLocalCache 中获取的。

这里先说明一下 PoolThreadLocalCache 的父类 FastThreadLocal。这里我参考了这几篇文章：

- [【源起Netty 外传】FastThreadLocal怎么Fast？](https://segmentfault.com/a/1190000012926809)
- [Netty 源码之 FastThreadLocal](https://www.jianshu.com/p/92c3832c0a8b)


简单来说 Fast 是因为 ThrealLocal 是使用线行探查法（下一个一个个查找）来解决 hash 冲突的，而 FastThreadLocal 依托于 `InternalThreadLocalMap.nextVariableIndex` (AtomInteger++) 来获取唯一 index 避免 hash 冲突带来的性能损耗。

这里为什么用 ThreadLocal 呢？在防止线程冲突情况下，找到最近使用的 Arena。

在 PoolThreadCache 中，使用到了 MemoryRegionCache 这个新的数据结构用于存储 ByteBuf 的实际内存数据。MemoryRegionCache 本质上是一个用于存储 PoolChunk (PoolThreacCache.Entry 中)的队列，同样也是根据了 Tiny(0-512b)，Small(512b - 4k)，Normal(8k-32M)，以及 Direct/Heap 区别。

PoolThreadCache.Entry 中存储 PoolChunk 的队列使用的是 MpscArrayQueue / MpscAtomicArrayQueue，一种用于多生产者而只有一个消费者的高并发队列.

#### 初始化

初始化入口在 `PoolArena$PoolThreadLocalCache.initialValue` 中，通过 `FastThreadLocal` 机制触发。PoolThreadCache 中用于分配的 PoolArena 是使用最近被使用到的 PooledByteBufAllocator 中的 PoolArena 数组。然后再分别初始化每个 MemoryRegionCache 即可。

#### 内存分配

在内存分配期间涉及到 PoolThreadCache 无论是 Tiny/Small/Normal 最终都是找到对应的 MemoryRegionCache 走到 allocate 这个方法中

```java
private boolean allocate(MemoryRegionCache<?> cache, PooledByteBuf buf, int reqCapacity) {
    if (cache == null) {
        // no cache found so just return false here
        return false;
    }
    boolean allocated = cache.allocate(buf, reqCapacity);
    if (++ allocations >= freeSweepAllocationThreshold) {
        allocations = 0;
        trim();
    }
    return allocated;
}
```

首先，分配内存

`MemoryRegionCache.allocate`

```java
/**
 * Allocate something out of the cache if possible and remove the entry from the cache.
 */
public final boolean allocate(PooledByteBuf<T> buf, int reqCapacity) {
    Entry<T> entry = queue.poll(); // 从队列中获取 Entry，MemoryRegionCache 初始化创建，free 时候会添加，也仅在 free 时候添加
    if (entry == null) {
        return false;
    }
    initBuf(entry.chunk, entry.nioBuffer, entry.handle, buf, reqCapacity); // 初始化 initBuf，这与上文描述的 initBuf 调用相同方法
    entry.recycle(); // 循环利用之后的触发通知器. 并进行一些资源的关闭工作.

    // allocations is not thread-safe which is fine as this is only called from the same thread all time.
    ++ allocations;
    return true;
}
```

然后判断分配内存次数是否达到了阈值，如果达到对齐并释放内存，所有 MemoryRegionCache 都将走到 `MemoryRegionCache.trim()`

```java
/**
 * Free up cached {@link PoolChunk}s if not allocated frequently enough.
 */
public final void trim() {
    int free = size - allocations; // 计算结余空间. size 数值：tiny: 512, small: 256, normal: 64
    allocations = 0;

    // We not even allocated all the number that are
    if (free > 0) {
        free(free); // 释放结余空间
    }
}
```

### 关于 PoolArena 中 PoolChunkList 逻辑说明

当调用 `allocateNormal` 的时候会先逐一调用 PoolChunkList.allocate，试图寻找已存在的可以利用的 PoolChunk，那什么作为可以利用的依据呢？

PoolArena 中存在 init-000-025-050-075-100 的 PoolChunkList 链表，分别对应了使用到的 PoolChunk 不同的内存利用率

- qInit：存储内存利用率 0-25% 的chunk
- q000：存储内存利用率 1-50% 的chunk
- q025：存储内存利用率 25-75% 的chunk
- q050：存储内存利用率 50-100% 的chunk
- q075：存储内存利用率 75-100% 的chunk
- q100：存储内存利用率 100% 的chunk

在执行 PoolChunkList.allocate 的时候首先会看当前要分配的内存大小是否符合该 PoolChunkList 的利用率要求，若满足，则 head(PoolChunk) 开始遍历分配直到成功，成功之后判断分配完成后的 PoolChunk 是否满足在此 PoolChunkList 内存利用率要求，若不满足，则将其移动置下个链表中.

#### q000存在的目的是什么？

> q000 是用来保存内存利用率在 1%-50% 的chunk，那么这里为什么不包括 0% 的 chunk？
> 
> 直接弄清楚这些，才好理解为什么不从 q000 开始分配。q000 中的 chunk，当内存利用率为0时，就从链表中删除，直接释放物理内存，避免越来越多的 chunk 导致内存被占满。
> 
> 想象一个场景，当应用在实际运行过程中，碰到访问高峰，这时需要分配的内存是平时的好几倍，当然也需要创建好几倍的chunk，如果先从 q000 开始，这些在高峰期创建的 chunk 被回收的概率会大大降低，延缓了内存的回收进度，造成内存使用的浪费。

#### 为什么选择从q050开始？

>
>1. q050 保存的是内存利用率 50%~100% 的 chunk，这应该是个折中的选择！这样大部分情况下，chunk 的利用率都会保持在一个较高水平，提高整个应用的内存利用率；
>2. qinit 的 chunk 利用率低，但不会被回收；
>3. q075 和 q100 由于内存利用率太高，导致内存分配的成功率大大降低，因此放到最后；

### free 流程

通过 `ReferenceCountUtil.release` or `ByteBuf.release` 来触发的 free 流程，其触发的起点是 `PooledByteBuf.deallocate`

```java
protected final void deallocate() {
    if (handle >= 0) {
        final long handle = this.handle;
        this.handle = -1;
        memory = null;
        chunk.arena.free(chunk, tmpNioBuf, handle, maxLength, cache); // PoolArena.free
        tmpNioBuf = null;
        chunk = null;
        recycle(); // 循环利用通知
    }
}
```

```java
void free(PoolChunk<T> chunk, ByteBuffer nioBuffer, long handle, int normCapacity, PoolThreadCache cache) {
    if (chunk.unpooled) { // 非池化的处理，非关注点
        int size = chunk.chunkSize();
        destroyChunk(chunk);
        activeBytesHuge.add(-size);
        deallocationsHuge.increment();
    } else {
        SizeClass sizeClass = sizeClass(normCapacity);
        if (cache != null && cache.add(this, chunk, nioBuffer, handle, normCapacity, sizeClass)) { // 若为缓存中的内存区域，不释放，直接预备下次使用
            // cached so not free it.
            return;
        }

        freeChunk(chunk, handle, sizeClass, nioBuffer); // 释放
    }
}
```

此处产生了两个分支，先看非缓存的部分 PoolArena.freeChunk

```java
void freeChunk(PoolChunk<T> chunk, long handle, SizeClass sizeClass, ByteBuffer nioBuffer) {
    final boolean destroyChunk;
    synchronized (this) {
        ... // 数据统计
        destroyChunk = !chunk.parent.free(chunk, handle, nioBuffer);
    }
    if (destroyChunk) {
        // destroyChunk not need to be called while holding the synchronized lock.
        destroyChunk(chunk);
    }
}
```

1. 调用对应 PoolChunkList 的 free 方法 `chunk.parent.free`，这里会区分是否是 Subpage，分别进行 free (memoryMapIdx 判断哪个 page, bitmapIdx 判断哪个内存块)。PoolChunkList 还会根据内存使用率移动 Chunk(如果有需要). 对于 Chunk 而言 free 主要是标记二叉树哪些位置内存需要清理。
2. 若 free 成功，则销毁 PoolChunk `PoolArena.destroyChunk`。此处是实际清理内存操作，Heap 等待 GC，Direct 调用 Unsafe.

再说缓存部分，这儿其实是**缓存的回收**流程，这里 PoolThreadCache 会将被释放的 chunk 放置入缓存中，供下次分配使用。

由于缓存获取使用的是其线程缓存 PoolThreadLocalCache，当其线程生命周期结束的时候会调用 `onRemoval` 对 PoolThreadCache 进行内存释放清理操作。