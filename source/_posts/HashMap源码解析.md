---
title: HashMap源码解析
date: 2017-04-21 16:28:17
tags:
    - Java
categories: 源码分析
---

``` java

// 常量列表
DEFAULT_INITIAL_CAPACITY = 1 << 4;  //The capacity is the number of buckets in the hash table
MAXIMUM_CAPACITY = 1 << 30;
/*
**The load factor is a measure of how full the hash table is allowed to get before its capacity is automatically increased. When the number of entries in the hash table exceeds the product of the load factor and the current capacity, the hash table is rehashed (that is, internal data structures are rebuilt) so that the hash table has approximately twice the number of buckets.
*/
DEFAULT_LOAD_FACTOR = 0.75f;        //填充因子
```

<!-- more -->

### `put`方法解析：
1. 判断是否是空 table ,初始化

``` java
        if (table == EMPTY_TABLE) {
            inflateTable(threshold);
        }
```
1.    如果key == null 则通过遍历来设置其中的值，可知null不参与hash

   ``` java
             for (Entry<K,V> e = table[0]; e != null; e = e.next) {
                 if (e.key == null) {
                     V oldValue = e.value;
                     e.value = value;
                     e.recordAccess(this);
                     return oldValue;
                 }
             }
             modCount++;
             addEntry(0, null, value, 0);
   ```
2.    进行hash运算

   ``` java
                  final int hash(Object k) {
                      int h = hashSeed;
                      if (0 != h && k instanceof String) {
                          return sun.misc.Hashing.stringHash32((String) k); //猜测是为字符串做了优化提供的Hash，并没有追踪到里面的源代码
                      }

                      h ^= k.hashCode();

                      // This function ensures that hashCodes that differ only by
                      // constant multiples at each bit position have a bounded
                      // number of collisions (approximately 8 at default load factor).
                      h ^= (h >>> 20) ^ (h >>> 12);
                      return h ^ (h >>> 7) ^ (h >>> 4);
                  }
   ```
3.    通过hash值与table的长度获取其在数组中的index

   ``` java
                      return h & (length-1);
   ```

   ```
    当 length 总是 2 的倍数时，h & (length-1)将是一个非常巧妙的设计：假设 h=5, length=16, 那么 h & length - 1 将得到 5；如果 h=6, length=16, 那么 h & length - 1 将得到 6 ……如果 h=15,length=16, 那么 h & length - 1 将得到 15；但是当 h=16 时 , length=16 时，那么 h & length - 1 将得到 0 了；当 h=17 时 , length=16 时，那么 h & length - 1 将得到 1 了……这样保证计算得到的索引值总是位于 table 数组的索引之内。`
   ```
4.    替换老旧的值，这里循环的目的是为了避免冲突，这里避免冲突的策略是依靠 Entry 这张链表(看其e.next)。

   ``` java
                      for (Entry<K,V> e = table[i]; e != null; e = e.next) {
                          Object k;
                          if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                              V oldValue = e.value;
                              e.value = value;
                              e.recordAccess(this);
                              return oldValue;
                          }
                      }
   ```
5.    添加计数

   ``` java
                      modCount++;
                  /**
     * The number of times this HashMap has been structurally modified
     * Structural modifications are those that change the number of mappings in
     * the HashMap or otherwise modify its internal structure (e.g.,
     * rehash).  This field is used to make iterators on Collection-views of
     * the HashMap fail-fast.  (See ConcurrentModificationException).
     */
   ```
6.    如果key不存在则

   ``` java
                      addEntry(hash, key, value, i);
                      return null;

                  void addEntry(int hash, K key, V value, int bucketIndex) {
                      if ((size >= threshold) && (null != table[bucketIndex])) { //是否需要重新定义table的大小
                          resize(2 * table.length);
                          hash = (null != key) ? hash(key) : 0;
                          bucketIndex = indexFor(hash, table.length);
                      }

                      createEntry(hash, key, value, bucketIndex);
                  }

                  void createEntry(int hash, K key, V value, int bucketIndex) {
                      Entry<K,V> e = table[bucketIndex];
                      table[bucketIndex] = new Entry<>(hash, key, value, e);
                      size++;//计数length
                  }
   ```
### `get`方法解析
1. 取 key = null 的值是直接遍历
2. 获取值

   ``` java
   Entry<K,V> entry = getEntry(key);
   return null == entry ? null : entry.getValue();

      final Entry<K,V> getEntry(Object key) {
          if (size == 0) {
              return null;
          }

          int hash = (key == null) ? 0 : hash(key); // 获取hash的key
          for (Entry<K,V> e = table[indexFor(hash, table.length)]; //避免冲突
               e != null;
               e = e.next) {
              Object k;
              if (e.hash == hash &&
                  ((k = e.key) == key || (key != null && key.equals(k))))
                  return e;
          }
          return null;
      }
   ```
### `Table`扩充方案

``` java
if ((size >= threshold) && (null != table[bucketIndex])) { //是否需要重新定义table的大小
               resize(2 * table.length);
               hash = (null != key) ? hash(key) : 0;
               bucketIndex = indexFor(hash, table.length);
           }
```

在 addEntry 中有这一串代码，其中的 `resize`方法是作为

``` java
    void resize(int newCapacity) {
        Entry[] oldTable = table;
        int oldCapacity = oldTable.length;
        if (oldCapacity == MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        }

        Entry[] newTable = new Entry[newCapacity];
        transfer(newTable, initHashSeedAsNeeded(newCapacity));
        table = newTable;
        threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
    }
```
#### HashMap的并发问题

``` java
    void transfer(Entry[] newTable, boolean rehash) {
        int newCapacity = newTable.length 
        for (Entry<K,V> e : table) {
            while(null != e) {
                Entry<K,V> next = e.next; // (1)
                if (rehash) {
                    e.hash = null == e.key ? 0 : hash(e.key);
                }
                int i = indexFor(e.hash, newCapacity);
                //将e放在冲突链表最前面
                e.next = newTable[i]; // (2) 
                newTable[i] = e; // (3) 
                e = next; // (4)
            }
        }
    }
```

现在假设我们的oldTable.length = 2, 而 newTable.length = 4。

原有的oldTable的值为 3, 7, 5. 则这三个值是以链表的形式存在于oldTable[1]中的。

此时执行函数 transfer， 如果是单线程程序的话则结果变为 newTable[1]=5->null, newTable[3] = 7->3->null。

如果是多线程，我们假定有一个线程一，一个线程二：
1. 线程一执行到代码中的(1)位置进行调度停止，则此时 e = 3, next = 7
2. 线程二执行完成，则此时的newTable[1] = 5->null, newTable[3] = 7->3->null
3. 此时线程一启动：
   1. 在 (2) e.next = newTable[3] = 7 -> 3 -> null.
   2. 在 (3) 的时候 newTable[3] = 3 -> null. 
   3. 在 (4) 的时候 e = next = 7 -> 3
4. 此时会形成一个循环 e = 3, e.next = 7, e.next.next = 3 = e


## 参考文档

[疫苗：Java HashMap的死循环](http://coolshell.cn/articles/9606.html)
