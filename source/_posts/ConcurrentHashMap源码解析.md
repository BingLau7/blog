---
title: ConcurrentHashMap源码解析
date: 2017-05-03 16:28:56
tags:
    - Java
    - 并发编程
categories: 源码分析
---

### ConcurrentMap接口说明

``` java
public interface ConcurrentMap<K, V> extends Map<K, V> {
    V putIfAbsent(K key, V value); // 如果指定key没有与任何值绑定，则它与value绑定
    /* 等同于
    if (!map.containsKey(key))
       return map.put(key, value);
    else
       return map.get(key);
    */
    boolean remove(Object key, Object value); // 只有当该key与value相映射时候删除掉该元素
    /* 等同于
    if (map.containsKey(key) && map.get(key).equals(value)) {
        map.remove(key);
        return true;
    } else return false;
    */
    boolean replace(K key, V oldValue, V newValue); // 当key的值为oldValue时用newValue代替
   /* 等同于
   if (map.containsKey(key) && map.get(key).equals(oldValue)) {
       map.put(key, newValue);
       return true;
   } else return false;
   */
    V replace(K key, V value); // 当map中包含有key时用value代替
   /* 等同于
   if (map.containsKey(key)) {
       return map.put(key, value);
   } else return null;
   */
}
```

<!-- more -->

注意：
-  ConcurrentMap 中不能存放值null
-  配合java.util.concurrent.atomic使用
### 为什么不使用 HashTable

`HashTable`容器使用`synchronized`来保证线程安全，但在线程竞争激烈的情况下`HashTable`的效率非常低下。因为当一个线程访问`HashTable`的同步方法时，其他线程访问`HashTable`的同步方法时，可能会进入阻塞或轮询状态。如线程1使用`put`进行添加元素，线程2不但不能使用`put`方法添加元素，并且也不能使用`get`方法来获取元素，所以竞争越激烈效率越低。
### 锁分段技术

`HashTable`容器在竞争激烈的并发环境下表现出效率低下的原因是所有访问`HashTable`的线程都必须竞争同一把锁，那假如容器里有多把锁，每一把锁用于锁容器其中一部分数据，那么当多线程访问容器里不同数据段的数据时，线程间就不会存在锁竞争，从而可以有效的提高并发访问效率，这就是`ConcurrentHashMap`所使用的锁分段技术，首先将数据分成一段一段的存储，然后给每一段数据配一把锁，当一个线程占用锁访问其中一个段数据的时候，其他段的数据也能被其他线程访问。
### ConcurrentHashMap的结构

`ConcurrentHashMap`是由`Segment`数组结构和`HashEntry`数组结构组成。`Segment`是一种可重入锁`ReentrantLock`，在`ConcurrentHashMap`里扮演锁的角色，`HashEntry`则用于存储键值对数据。一个`ConcurrentHashMap`里包含一个`Segment`数组，`Segment`的结构和`HashMap`类似，是一种数组和链表结构， 一个`Segment`里包含一个`HashEntry`数组，每个`HashEntry`是一个链表结构的元素， 每个`Segment`守护者一个`HashEntry`数组里的元素,当对`HashEntry`数组的数据进行修改时，必须首先获得它对应的`Segment`锁。

![](http://cdn4.infoqstatic.com/statics_s2_20160927-0250/resource/articles/ConcurrentHashMap/zh/resources/2.jpg)

>  ReentrantLock介绍
>
>  **ReentrantLock**的实现不仅可以替代隐式的**synchronized**关键字，而且能够提供超过关键字本身的多种功能。
>
>  这里提到一个锁获取的公平性问题，如果在绝对时间上，先对锁进行获取的请求一定被先满足，那么这个锁是公平的，反之，是不公平的，也就是说等待时间最长的线程最有机会获取锁，也可以说锁的获取是有序的。ReentrantLock这个锁提供了一个构造函数，能够控制这个锁是否是公平的。
>
>  而锁的名字也是说明了这个锁具备了重复进入的可能，也就是说能够让当前线程多次的进行对锁的获取操作，这样的最大次数限制是Integer.MAX_VALUE，约21亿次左右。
>
>  `nonfairTryAcquire`是非公平的获取锁的方式
>
>  ``` java
>  final boolean nonfairTryAcquire(int acquires) {
>      final Thread current = Thread.currentThread();
>      int c = getState();
>      // 如果当前状态为初始状态，那就尝试设置状态
>      if (c == 0) {
>          if (compareAndSetState(0, acquires)) {
>              // 状态设置成功就返回
>              setExclusiveOwnerThread(current);
>              return true;
>          }
>      }
>      // 如果状态被设置，且获取锁的线程又是当前线程的时候，进行状态的自增；
>      else if (current == getExclusiveOwnerThread()) {
>          int nextc = c + acquires;
>          if (nextc < 0) // overflow
>              throw new Error("Maximum lock count exceeded");
>          setState(nextc);
>          return true;
>      }
>      // 如果未设置成功状态且当前线程不是获取锁的线程，那么返回失败。
>      return false;
>  }
>  ```
### 初始化

``` Java
    public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
        if (concurrencyLevel > MAX_SEGMENTS)
            concurrencyLevel = MAX_SEGMENTS;
        // Find power-of-two sizes best matching arguments
        int sshift = 0;
        int ssize = 1;
        while (ssize < concurrencyLevel) {
            ++sshift;
            ssize <<= 1;
        }
        // hash算法中需要使用
        this.segmentShift = 32 - sshift; // 用于定位参与hash运算的位数
        this.segmentMask = ssize - 1; // hash运算的掩码
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        int c = initialCapacity / ssize;
        if (c * ssize < initialCapacity)
            ++c;
        int cap = MIN_SEGMENT_TABLE_CAPACITY;
        while (cap < c)
            cap <<= 1;
        // create segments and segments[0]
        Segment<K,V> s0 =
            new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                             (HashEntry<K,V>[])new HashEntry[cap]);
        Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize]; 
        UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
        this.segments = ss;
    }
```

通过`concurrencyLevel`(默认是16)得出`Segment`数组的长度，为了能通过按位与的哈希算法来定位`segments`数组的索引，必须保证`segments`数组的长度是2的N次方，所以必须计算出一个是大于或等于`concurrencyLevel`的最好的2得N次方值来作为`segments`数组的长度。假设为14/15/16，则ssize都为16，即容器里锁的个数也为16。

> transient关键字
>
> 这个字段的生命周期仅存于调用者的内存中而不会写到磁盘里持久化。
>
> 简单来说就是当一个对象需要序列化时，这个方法用于修饰不需要被序列化的字段。
### putIfAbsent

``` java
    public V putIfAbsent(K key, V value) {
        Segment<K,V> s;
        if (value == null)
            throw new NullPointerException();
        int hash = hash(key);
        int j = (hash >>> segmentShift) & segmentMask;
        // sun.misc.unsafe类中使用getObject来检测实际值的字段
        if ((s = (Segment<K,V>)UNSAFE.getObject  
             (segments, (j << SSHIFT) + SBASE)) == null)
            // 如果不存在Segment，则创建一个sengment然后再插入HashEntry
            s = ensureSegment(j);
        return s.put(key, hash, value, true);
    }
```
#### ensureSegment方法

``` java
    private Segment<K,V> ensureSegment(int k) {
        final Segment<K,V>[] ss = this.segments;
        long u = (k << SSHIFT) + SBASE; // raw offset
        Segment<K,V> seg;
        // getObjectVolatile是以Volatile的方式获得目标的Segment，Volatile是为了保证可见性。
        if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) {
            // 不存在该segment则需要重新创建，其中是以ss[0]为镜像进行创建
            Segment<K,V> proto = ss[0]; // use segment 0 as prototype
            int cap = proto.table.length;
            float lf = proto.loadFactor;
            int threshold = (int)(cap * lf);
            HashEntry<K,V>[] tab = (HashEntry<K,V>[])new HashEntry[cap];
            // 重复检测
            if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                == null) { // recheck
                // 创建新的Segment
                Segment<K,V> s = new Segment<K,V>(lf, threshold, tab);
                //以CAS的方式，将新建的Segment，set到指定的位置。
                while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                       == null) {
                    if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))
                        break;
                }
            }
        }
        return seg;
    }
```

> CAS（Compare and swap）比较和替换是设计并发算法时用到的一种技术。简单来说，比较和替换是使用一个期望值和一个变量的当前值进行比较，如果当前变量的值与我们期望的值相等，就使用一个新值替换当前变量的值。
#### Segment的put方法

``` java
        final V put(K key, int hash, V value, boolean onlyIfAbsent) {
            // 尝试获取锁，如果能获取锁则根据hash的地址获取其HashEntry实例
            HashEntry<K,V> node = tryLock() ? null :
                scanAndLockForPut(key, hash, value);
            V oldValue;
            try {
                HashEntry<K,V>[] tab = table; // table是segment中独有的table
                int index = (tab.length - 1) & hash;
                /*
                通过位运算找出HashEntry数组中index位置的值，
                这里的位运算运用到了UNSAFE中的方法。
                */
                HashEntry<K,V> first = entryAt(tab, index);
                for (HashEntry<K,V> e = first;;) {
                    if (e != null) {
                        K k;
                        // 该位置是否有可替代的值
                        if ((k = e.key) == key ||
                            (e.hash == hash && key.equals(k))) {
                            oldValue = e.value;
                            // 这儿是 ConcurrentMap的put方法和putIfAbsent方法的区别
                            if (!onlyIfAbsent) {
                                e.value = value;
                                ++modCount;
                            }
                            break;
                        }
                        e = e.next;
                    }
                    else {
                        if (node != null)
                            // 通过链表来解决冲突
                            // 这个地方实现就是粗暴的 UNSAFE.putOrderedObject(this, nextOffset, n);
                            node.setNext(first);
                        else
                            // 没有冲突则创建一个新的 HashEntry
                            node = new HashEntry<K,V>(hash, key, value, first);
                        int c = count + 1;
                        // 查看是否需要更新HashEntry
                        if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                            rehash(node);
                        else
                            // 重置HashEntry的位置
                            setEntryAt(tab, index, node);
                        ++modCount;
                        count = c;
                        oldValue = null;
                        break;
                    }
                }
            } finally {
                // 解锁
                unlock();
            }
            return oldValue;
        }
```
#### Segment中的`scanAndLockForPut`方法

``` java
        /**
         * Scans for a node containing given key while trying to
         * acquire lock, creating and returning one if not found. Upon
         * return, guarantees that lock is held. UNlike in most
         * methods, calls to method equals are not screened: Since
         * traversal speed doesn't matter, we might as well help warm
         * up the associated code and accesses as well.
         *
         * @return a new node if key not found, else null
         */        
         private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
            HashEntry<K,V> first = entryForHash(this, hash);
            HashEntry<K,V> e = first;
            HashEntry<K,V> node = null;
            int retries = -1; // negative while locating node
            while (!tryLock()) {
                HashEntry<K,V> f; // to recheck first below
                if (retries < 0) {
                    if (e == null) {
                        if (node == null) // speculatively create node
                            node = new HashEntry<K,V>(hash, key, value, null);
                        retries = 0;
                    }
                    else if (key.equals(e.key))
                        retries = 0;
                    else
                        e = e.next;
                }
                else if (++retries > MAX_SCAN_RETRIES) {
                    lock();
                    break;
                }
                else if ((retries & 1) == 0 &&
                         (f = entryForHash(this, hash)) != first) {
                    e = first = f; // re-traverse if entry changed
                    retries = -1;
                }
            }
            return node;
        }
```
#### Segment的`rehash`方法

``` java
        private void rehash(HashEntry<K,V> node) {
            /*
             * Reclassify nodes in each list to new table.  Because we
             * are using power-of-two expansion, the elements from
             * each bin must either stay at same index, or move with a
             * power of two offset. We eliminate unnecessary node
             * creation by catching cases where old nodes can be
             * reused because their next fields won't change.
             * Statistically, at the default threshold, only about
             * one-sixth of them need cloning when a table
             * doubles. The nodes they replace will be garbage
             * collectable as soon as they are no longer referenced by
             * any reader thread that may be in the midst of
             * concurrently traversing table. Entry accesses use plain
             * array indexing because they are followed by volatile
             * table write.
             */
            HashEntry<K,V>[] oldTable = table;
            int oldCapacity = oldTable.length;
            int newCapacity = oldCapacity << 1;
            threshold = (int)(newCapacity * loadFactor);
            HashEntry<K,V>[] newTable =
                (HashEntry<K,V>[]) new HashEntry[newCapacity];
            int sizeMask = newCapacity - 1;
            // 遍历旧的HashEntry[]填充到新的里面去, 由于之前的trylock()操作所以在该Segment不需要担心并发问题
            for (int i = 0; i < oldCapacity ; i++) {
                HashEntry<K,V> e = oldTable[i];
                if (e != null) {
                    HashEntry<K,V> next = e.next;
                    int idx = e.hash & sizeMask;
                    if (next == null)   //  Single node on list
                        newTable[idx] = e;
                    else { // Reuse consecutive sequence at same slot
                        HashEntry<K,V> lastRun = e;
                        int lastIdx = idx;
                        for (HashEntry<K,V> last = next;
                             last != null;
                             last = last.next) {
                            int k = last.hash & sizeMask;
                            if (k != lastIdx) {
                                lastIdx = k;
                                lastRun = last;
                            }
                        }
                        newTable[lastIdx] = lastRun;
                        // Clone remaining nodes
                        for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
                            V v = p.value;
                            int h = p.hash;
                            int k = h & sizeMask;
                            HashEntry<K,V> n = newTable[k];
                            newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
                        }
                    }
                }
            }
            int nodeIndex = node.hash & sizeMask; // add the new node
            node.setNext(newTable[nodeIndex]);
            newTable[nodeIndex] = node;
            table = newTable;
        }
```
### get

``` java
    public V get(Object key) {
        Segment<K,V> s; // manually integrate access methods to reduce overhead
        HashEntry<K,V>[] tab;
        int h = hash(key);
        long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
        if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
            (tab = s.table) != null) {
            // 由于Segment的table是volatile的，所以不需要加锁来获取
            for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                     (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
                 e != null; e = e.next) {
                K k;
                if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                    return e.value;
            }
        }
        return null;
    }
```

> Volatile 关键字
>
> 在多线程并发编程中synchronized和Volatile都扮演着重要的角色，Volatile是**轻量级的synchronized**，它在多处理器开发中保证了共享变量的“可见性”。可见性的意思是当一个线程修改一个共享变量时，另外一个线程能读到这个修改的值。
### remove

``` java
    public boolean remove(Object key, Object value) {
        int hash = hash(key);
        Segment<K,V> s;
        return value != null && (s = segmentForHash(hash)) != null &&
            s.remove(key, hash, value) != null;
    }
```
#### Segment的`remove`方法

``` java
        final V remove(Object key, int hash, Object value) {
            if (!tryLock()) //尝试获取锁，有次数限制的
                scanAndLock(key, hash);
            V oldValue = null;
            try {
                HashEntry<K,V>[] tab = table;
                int index = (tab.length - 1) & hash;
                HashEntry<K,V> e = entryAt(tab, index);
                HashEntry<K,V> pred = null;
                while (e != null) {
                    K k;
                    //删除流程
                    HashEntry<K,V> next = e.next;
                    if ((k = e.key) == key ||
                        (e.hash == hash && key.equals(k))) {
                        V v = e.value;
                        if (value == null || value == v || value.equals(v)) {
                            if (pred == null)
                                //如果没有冲突重新设置next的位置
                                setEntryAt(tab, index, next);
                            else
                                // 确定前面的节点，然后前面节点连接被删除节点的next节点
                                pred.setNext(next);
                            ++modCount;
                            --count;
                            oldValue = v;
                        }
                        break;
                    }
                    pred = e;
                    e = next;
                }
            } finally {
                unlock();
            }
            return oldValue;
        }
```
#### Segment的`scanAndLock`

``` java
        /**
         * Scans for a node containing the given key while trying to
         * acquire lock for a remove or replace operation. Upon
         * return, guarantees that lock is held.  Note that we must
         * lock even if the key is not found, to ensure sequential
         * consistency of updates.
         */
        private void scanAndLock(Object key, int hash) {
            // similar to but simpler than scanAndLockForPut
            HashEntry<K,V> first = entryForHash(this, hash);
            HashEntry<K,V> e = first;
            int retries = -1;
            while (!tryLock()) {
                HashEntry<K,V> f;
                if (retries < 0) {
                    if (e == null || key.equals(e.key))
                        retries = 0;
                    else
                        e = e.next;
                }
                else if (++retries > MAX_SCAN_RETRIES) {
                    lock();
                    break;
                }
                else if ((retries & 1) == 0 &&
                         (f = entryForHash(this, hash)) != first) {
                    e = first = f;
                    retries = -1;
                }
            }
        }
```
### replace(K key, V value)

``` java
    public V replace(K key, V value) {
        int hash = hash(key);
        if (value == null)
            throw new NullPointerException();
        Segment<K,V> s = segmentForHash(hash);
        return s == null ? null : s.replace(key, hash, value);
    }
```
#### Segment的`V replace(K key, int hash, V value)`

``` java
        final V replace(K key, int hash, V value) {
            if (!tryLock())
                scanAndLock(key, hash);
            V oldValue = null;
            try {
                HashEntry<K,V> e;
                for (e = entryForHash(this, hash); e != null; e = e.next) { //只有key才能replace
                    K k;
                    if ((k = e.key) == key ||
                        (e.hash == hash && key.equals(k))) {
                        oldValue = e.value;
                        e.value = value;
                        ++modCount;
                        break;
                    }
                }
            } finally {
                unlock();
            }
            return oldValue;
        }
```
### replace(K key, V oldValue, V newValue)

``` java
    public boolean replace(K key, V oldValue, V newValue) {
        int hash = hash(key);
        if (oldValue == null || newValue == null)
            throw new NullPointerException();
        Segment<K,V> s = segmentForHash(hash);
        return s != null && s.replace(key, hash, oldValue, newValue);
    }
```

基本同上

> sun.misc.Unsafe
>
> 可以有意的执行一些不安全、容易犯错的操作。
>
> 其使用场景：
> -  对变量和数组内容的原子访问，自定义内存屏障
> -  对序列化的支持
> -  自定义内存管理/高效的内存布局
> -  与原生代码和其他JVM进行互操作
> -  对高级锁的支持


## 参考文档

[聊聊并发（四）——深入分析ConcurrentHashMap](http://www.infoq.com/cn/articles/ConcurrentHashMap)

[sun.misc.Unsafe的后启示录](http://www.infoq.com/cn/articles/A-Post-Apocalyptic-sun.misc.Unsafe-World)

[Java并发编程之CAS](http://ifeve.com/compare-and-swap/)

[深入理解ConcurrentHashmap](http://www.jianshu.com/p/bd972088a494)

[ReentrantLock(重入锁)以及公平性](http://ifeve.com/reentrantlock-and-fairness/)

[为什么ConcurrentHashMap是弱一致的](http://ifeve.com/concurrenthashmap-weakly-consistent/)

[聊聊并发（一）——深入分析Volatile的实现原理](http://www.infoq.com/cn/articles/ftf-java-volatile)
