---
title: LinkedHashMap源码解析
date: 2017-05-25 16:48:41
tags:
---

### 概述

>  Hash table and linked list implementation of the Map interface, with predictable iteration order. This implementation differs fromHashMap in that it maintains a doubly-linked list running through all of its entries. This linked list defines the iteration ordering, which is normally the order in which keys were inserted into the map (_insertion-order_). Note that insertion order is not affected if a key is _re-inserted_ into the map. (A key k is reinserted into a map m if m.put(k, v) is invoked when m.containsKey(k) would return trueimmediately prior to the invocation.)

<!-- more -->

其`header`头定义

``` java
    /**
     * The head of the doubly linked list.
     */
    private transient Entry<K,V> header;

    /**
     * Called by superclass constructors and pseudoconstructors (clone,
     * readObject) before any entries are inserted into the map.  Initializes
     * the chain.
     */
    @Override
    void init() {
        header = new Entry<>(-1, null, null, null);
        header.before = header.after = header;
    }
```
### `put`方法：

LinkedHashMap的Hash算法是沿用了HashMap的算法。其没有实现`put`方法，实现了`get`方法为了方便`accessOrder`方法。

``` java

    public V put(K key, V value) {
        if (table == EMPTY_TABLE) {
            inflateTable(threshold);
        }
        if (key == null)
            return putForNullKey(value); // 对于null的特殊处理
        int hash = hash(key); 
        int i = indexFor(hash, table.length); 
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this); //这里是Entry自己实现的，HashMap中保留为空方法
                return oldValue;
            }
        }

        modCount++;
        addEntry(hash, key, value, i);
        return null;
    }
```

因为其Entry是一个双向链表，其中定义是

``` java
        // These fields comprise the doubly linked list used for iteration.
        Entry<K,V> before, after;
```

我们先看一下不需要按照访问顺序进行排序的(即`accessOrder = false`)的情况：

每个元素如果没有冲突都会执行`addEntry`方法
#### `addEntry`与`createEntry`

``` java
    /**
     * This override alters behavior of superclass put method. It causes newly
     * allocated entry to get inserted at the end of the linked list and
     * removes the eldest entry if appropriate.
     */    
    void addEntry(int hash, K key, V value, int bucketIndex) {
        super.addEntry(hash, key, value, bucketIndex);

        // Remove eldest entry if instructed
        Entry<K,V> eldest = header.after;
        if (removeEldestEntry(eldest)) { // 始终返回false
            removeEntryForKey(eldest.key);
        }
    }

    /**
     * This override differs from addEntry in that it doesn't resize the
     * table or remove the eldest entry.
     */
    void createEntry(int hash, K key, V value, int bucketIndex) {
        HashMap.Entry<K,V> old = table[bucketIndex];
        Entry<K,V> e = new Entry<>(hash, key, value, old);
        table[bucketIndex] = e;
        e.addBefore(header);
        size++;
    }

    Entry:
        /**
         * Inserts this entry before the specified existing entry in the list.
         */
        private void addBefore(Entry<K,V> existingEntry) {
            after  = existingEntry;
            before = existingEntry.before;
            before.after = this;
            after.before = this;
        }
```

在执行`super.addEntry`的时候回执行`createEntry`方法，此时最主要的区别就是`addBefore`方法，此方法是将自己这个新增的Entry放置在`existingEntry`之后，也就是其头结点`header`之后。可以通过 `header`来遍历这个双向链表。

再分析需要按照访问顺序进行排序的(即`accessOrder = false`)的情况：

此时put方法中如果key原本就是有值，即相当于我们队access进行了一次访问，此时执行`recordAccess`方法

``` java
        void recordAccess(HashMap<K,V> m) {
            LinkedHashMap<K,V> lm = (LinkedHashMap<K,V>)m;
            if (lm.accessOrder) {
                lm.modCount++;
                remove(); // 先将该节点双向链表中删除
                addBefore(lm.header); // 再将该结点防止与header之后
            }
        }

        /**
         * Removes this entry from the linked list.
         */
        private void remove() {
            before.after = after;
            after.before = before;
        }

```
### `get`方法：

``` java
    public V get(Object key) {
        Entry<K,V> e = (Entry<K,V>)getEntry(key);
        if (e == null)
            return null;
        e.recordAccess(this);
        return e.value;
    }
```

此处可同理上述，访问时候执行`recordAccess`然后根据`accessOrder`的值决定其行为。
