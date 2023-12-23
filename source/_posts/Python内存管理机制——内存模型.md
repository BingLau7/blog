---
title: Python内存管理机制——内存模型
date: 2017-01-23 16:50:07
tags:
    - Python
    - 内存管理
categories: 源码分析
---

Python 中所有的内存管理机制都有两套实现，这两套实现由编译符号`PYMALLOC_DEBUG` 控制，当该符号被定义时，使用的是 debug 模式下的内存管理机制，这套机制在正常的内存管理动作之外，还会记录许多关于内存的信息，以方便 Python 在开发时进行调试；而当该符号未被定义时，Python 的内存管理机制只进行正常的内存管理动作。

```c
// /include/Python.h

#if defined(Py_DEBUG) && defined(WITH_PYMALLOC) && !defined(PYMALLOC_DEBUG)
#define PYMALLOC_DEBUG
#endif
```

## 内存管理架构

在 Python 中，内存管理机制被抽象成下图这样的层次似结果。

![Python 内存管理机制的层次结构](https://github.com/BingLau7/blog/blob/master/images/blog_18/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-04-04%20%E4%B8%8B%E5%8D%883.48.00.png?raw=true)

### Layer 0：

操作系统提供的内存管理接口，比如 C 运行时所提供的 malloc 和 free 接口。这层由操作系统实现并管理，Python 不能干涉这一层的行为。

### Layer 1：

Python 基于第 0 层操作系统的内存管理接口包装而成的，这一层并没有在第 0 层加入太多的冻灾，其目的仅仅是为 Python 提供一层同意的 raw memory 的管理接口。防止操作系统的差异。第一次实现就是一组以`PyMem_`为前缀的函数族。

```C
// Include/pymem.h

PyAPI_FUNC(void *) PyMem_Malloc(size_t size);
#if !defined(Py_LIMITED_API) || Py_LIMITED_API+0 >= 0x03050000
PyAPI_FUNC(void *) PyMem_Calloc(size_t nelem, size_t elsize);
#endif
PyAPI_FUNC(void *) PyMem_Realloc(void *ptr, size_t new_size);
PyAPI_FUNC(void) PyMem_Free(void *ptr);

// Object/obmalloc.c
// 这里使用了一个数据结构 PyMemAllocatorEx 里面定义了上下文及四种方法，上面有用 #ifdef...#endif 来初始化该数据结构方法
// 原始 C 语言方法也在该文件中

void *
PyMem_Malloc(size_t size)
{
    /* see PyMem_RawMalloc() */
    if (size > (size_t)PY_SSIZE_T_MAX)
        return NULL;
    return _PyMem.malloc(_PyMem.ctx, size);
}

void *
PyMem_Calloc(size_t nelem, size_t elsize)
{
    /* see PyMem_RawMalloc() */
    if (elsize != 0 && nelem > (size_t)PY_SSIZE_T_MAX / elsize)
        return NULL;
    return _PyMem.calloc(_PyMem.ctx, nelem, elsize);
}

void *
PyMem_Realloc(void *ptr, size_t new_size)
{
    /* see PyMem_RawMalloc() */
    if (new_size > (size_t)PY_SSIZE_T_MAX)
        return NULL;
    return _PyMem.realloc(_PyMem.ctx, ptr, new_size);
}

void
PyMem_Free(void *ptr)
{
    _PyMem.free(_PyMem.ctx, ptr);
}
```

在第一层中，Python 还提供了面向 Python 中类型的内存分配器

```c
// Include/pymem.h

/*
 * Type-oriented memory interface
 * ==============================
 *
 * Allocate memory for n objects of the given type.  Returns a new pointer
 * or NULL if the request was too large or memory allocation failed.  Use
 * these macros rather than doing the multiplication yourself so that proper
 * overflow checking is always done.
 */

#define PyMem_New(type, n) \
  ( ((size_t)(n) > PY_SSIZE_T_MAX / sizeof(type)) ? NULL :	\
	( (type *) PyMem_Malloc((n) * sizeof(type)) ) )
#define PyMem_NEW(type, n) \
  ( ((size_t)(n) > PY_SSIZE_T_MAX / sizeof(type)) ? NULL :	\
	( (type *) PyMem_MALLOC((n) * sizeof(type)) ) )

/*
 * The value of (p) is always clobbered by this macro regardless of success.
 * The caller MUST check if (p) is NULL afterwards and deal with the memory
 * error if so.  This means the original value of (p) MUST be saved for the
 * caller's memory error handler to not lose track of it.
 */
#define PyMem_Resize(p, type, n) \
  ( (p) = ((size_t)(n) > PY_SSIZE_T_MAX / sizeof(type)) ? NULL :	\
	(type *) PyMem_Realloc((p), (n) * sizeof(type)) )
#define PyMem_RESIZE(p, type, n) \
  ( (p) = ((size_t)(n) > PY_SSIZE_T_MAX / sizeof(type)) ? NULL :	\
	(type *) PyMem_REALLOC((p), (n) * sizeof(type)) )
```

在 `PyMem_New` 中，只要提供类型和数量，Python 会自动侦测其所需的内存空间大小。

### Layer 2：

以 PyObje_为前缀的函数族，主要提供了创建 Python 对象的接口。这一套函数族又被唤作 Pymalloc 机制。

在第二层内存管理机制之上，对于 Python 中的一些常用对象，比如整数对象、字符串对象等，Python 又构建了更高抽象层次的内存管理策略。

真正在 Python 中发挥巨大作用、同时也是 GC 的藏身之处的内存管理机制所在层次。

### Layer 3:

第三层的内存管理策略，主要就是对象缓冲池机制。（参加书中第一部分，或者下篇博客）。

## 小块空间的内存池

在 Python 中，许多时候申请的内存都是小块的内存，这些小块内存在申请后，很快又会被释放，由于这些内存的申请并不是为了创建对象，所以并没有对象一级的内存池机制。这就意味着 Python 在运行期间会大量地执行 malloc 和 free 操作，导致操作系统频繁地在用户态和核心态之间进行切换，这将严重影响 Python 的执行效率。为了提高 Python 的执行效率，Python 引入了一个内存池机制，用于管理对小块内存的申请和释放。**这也就是之前提到的 Pymalloc 机制**。

在 Python 中，整个小块内存池可以视为一个层次结构，在这个层次结构中，一共分为4层，从下至上分别是：block、pool、arena 和内存池。需要说明的是，block、pool 和 arena 都是 Python 代码中可以找到的实体，而最顶层的『内存池』只是一个概念上的东西，表示 Python 对于整个小块内存分配和释放行为的内存管理机制。

### Block

在最底层，block 是一个确定大小的内存块。在 Python 中，有多种 block，不同种类的 block 都有不同的内存大小，这个内存大小的值被称为 size class。为了在当前主流的平台都能获得最佳性能，所有的 block 的长度都是8字节对齐的。

```c
// Object/obmalloc.c

#define ALIGNMENT               8               /* must be 2^N */
#define ALIGNMENT_SHIFT         3
```

block 上限，当申请的内存大小小于这个上限时，Python 可以使用不同种类的 block 来满足对内存的需求；当神奇的内存大小超过这个上限，Python 就会对内存的请求转交给第一层的内存管理机制，即 PyMem 函数族，来处理。

```C
// Object/obmalloc.c

#define SMALL_REQUEST_THRESHOLD 512
#define NB_SMALL_SIZE_CLASSES   (SMALL_REQUEST_THRESHOLD / ALIGNMENT)
```

现在，需要指出一个相当关键的点，虽然我们这里谈论了很多block，但是在Python中，block只是一个概念，在Python源码中没有与之对应的实体存在。之前我们说对象，对象在Python源码中有对应的PyObject；我们说列表，列表在Python源码中对应PyListObject、PyType_List。这里的block就很奇怪了，它仅仅是概念上的东西，我们知道它是具有一定大小的内存，但它不与Python源码里的某个东西对应。然而，Python却提供了一个管理block的东西，这就是下面要剖析的pool。

### Pool

一组 block 的集合称为一个 pool，换句话说，一个 pool 管理着一堆有固定大小的内存块。事实上，pool 管理着一大块内存，它由一定的策略，将这块大的内存划分为多个小的内存块。在 Python 中，一个 pool 的大小通常为一个系统内存页，由于当前大多数 Python 支持的系统的内存页都是4KB，所以 Python 内部也将一个 pool 的大小定义为4KB。

```C
// Object/obmalloc.c

#define SYSTEM_PAGE_SIZE        (4 * 1024)
#define SYSTEM_PAGE_SIZE_MASK   (SYSTEM_PAGE_SIZE - 1)

/*
 * Size of the pools used for small blocks. Should be a power of 2,
 * between 1K and SYSTEM_PAGE_SIZE, that is: 1k, 2k, 4k.
 */
#define POOL_SIZE               SYSTEM_PAGE_SIZE        /* must be 2^N */
#define POOL_SIZE_MASK          SYSTEM_PAGE_SIZE_MASK
```

```C
// Object/obmalloc.c

/* When you say memory, my mind reasons in terms of (pointers to) blocks */
typedef uint8_t block;

/* Pool for small blocks. */
struct pool_header {
    union { block *_padding;
            uint count; } ref;          /* number of allocated blocks    */
    block *freeblock;                   /* pool's free list head         */
    struct pool_header *nextpool;       /* next pool of this size class  */
    struct pool_header *prevpool;       /* previous pool       ""        */
    uint arenaindex;                    /* index into arenas of base adr */
    uint szidx;                         /* block size class index        */
    uint nextoffset;                    /* bytes to virgin block         */
    uint maxnextoffset;                 /* largest valid nextoffset      */
};
```

4KB 中除去 pool_header，还有一大块内存就是 pool 中维护的 block 的集合占用的内存。

前面提到block 是有固定大小的内存块，因此，pool 也携带了大量这样的信息。一个 pool 管理的所有 block，它们的大小都是一样的。也就是说，一个 pool 可能管理了100个32个字节的 block，也可能管理了100个64个字节的 block，但是绝不会有一个管理了50个32字节的 block 和50个64字节的 block 的 pool 存在。每一个 pool 都和一个 size 联系在一起，更确切地说，都和一个 size class index 联系在一起。这就是 `pool_header` 中的 szindex 的意义。

假设我们手上现在有一块 4KB 的内存，来看看 Python 是如何将这块内存改造为一个管理32字节 block 的 pool，并从 pool 中取出第一块 block 的。

```C
// [obmalloc.c]-[convert 4k raw memory to pool]
#define ROUNDUP(x)    (((x) + ALIGNMENT_MASK) & ~ALIGNMENT_MASK)
#define POOL_OVERHEAD   ROUNDUP(sizeof(struct pool_header))
#define struct pool_header* poolp
#define uchar block

poolp pool;
block* bp;
…… // pool指向了一块4kB的内存
pool->ref.count = 1;
//设置pool的size class index
pool->szidx = size; 
//将size class index转换为size，比如3转换为32字节
size = INDEX2SIZE(size); 
//跳过用于pool_header的内存，并进行对齐
bp = (block *)pool + POOL_OVERHEAD; 
//实际就是pool->nextoffset = POOL_OVERHEAD+size+size
pool->nextoffset = POOL_OVERHEAD + (size << 1); 
pool->maxnextoffset = POOL_SIZE - size;
pool->freeblock = bp + size;
*(block **)(pool->freeblock) = NULL;
// bp 返回的实际是一个地址，这个地址之后有将近 4KB 的内存实际上都是可用的，但是可用肯定申请内存的函数只会使用[bp, bp+size] 这个区间的内存，这是由 size class index 可用保证的。
return (void *)bp; 
```

![改造成 pool 后的 4KB 内存](https://github.com/BingLau7/blog/blob/master/images/blog_18/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-04-04%20%E4%B8%8B%E5%8D%886.54.29.png?raw=true)

注意其中的实线箭头是指针，但是虚线箭头不是代表指针，是偏移位置的形象表示。在nextoffset和maxnextoffset中存储的是相对于poo头部的偏移位置。

```C
// [obmalloc.c]-[allocate block]

if (pool != pool->nextpool) {
            ++pool->ref.count;
            bp = pool->freeblock;
            ……
            if (pool->nextoffset <= pool->maxnextoffset) {
                //有足够的block空间
                pool->freeblock = (block *)pool + pool->nextoffset;
                pool->nextoffset += INDEX2SIZE(size);
                // 设置*freeblock 的动作正是建立离散自由 block 链表的关键所在
                *(block **)(pool->freeblock) = NULL;
                return (void *)bp;
            }
        }
 
```

原来freeblock指向的是下一个可用的block的起始地址，这一点在上图中也可以看得出。当再次申请32字节的block时，只需返回freeblock指向的地址就可以了，很显然，这时freeblock需要向前进，指向下一个可用的block。这时，nextoffset现身了。

在pool header中，nextoffset和maxoffset是两个用于对pool中的block集合进行迭代的变量：从初始化pool的结果及图16-2中可以看到，它所指示的偏移位置正好指向了freeblock之后的下一个可用的block的地址。从这里分配block的动作也可以看到，在分配了block之后，freeblock和nextoffset都会向前移动一个block的距离，如此反复，就可对所有的block进行一次遍历。而maxnextoffset指名了该pool中最后一个可用的block距pool开始位置的便移，它界定了pool的边界，当nextoffset > maxnextoff 时，也就意味着已经遍历完了pool中所有的block了。

可以想像，一旦Python运转起来，内存的释放动作将会导致pool中出现大量的离散的自由block，Python必须建立一种机制，将这些离散的自由block组织起来，再次使用。这个机制就是所谓的自由block链表。这个链表的关键就着落在pool_header中的那个freeblock身上。

```C
// [obmalloc.c]
//基于地址P获得离P最近的pool的边界地址
#define POOL_ADDR(P) ((poolp)((uptr)(P) & ~(uptr)POOL_SIZE_MASK))

void PyObject_Free(void *p)
{
    poolp pool;
    block *lastfree;
    poolp next, prev;
    uint size;

    pool = POOL_ADDR(p);
    //判断p指向的block是否属于pool
    if (Py_ADDRESS_IN_RANGE(p, pool)) {
        // 被释放的第一个字节的值被设置为当前的 freeblock 的值
        *(block **)p = lastfree = pool->freeblock; 
        // pool 的值被更新，指向其首地址，则一个 block 被插入到了离散自由的 block 链表中
        pool->freeblock = (block *)p;             
        ……
    }
}
```

![释放了 block 之后产生的自由 block 链表](https://github.com/BingLau7/blog/blob/master/images/blog_18/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-04-04%20%E4%B8%8B%E5%8D%887.25.44.png?raw=true)

### arena

在 Python 中，多个 pool 聚合的结果就是一个 arena。

```C
// Object/obmalloc.c

// arena 大小的默认值
#define ARENA_SIZE              (256 << 10)     /* 256KB */

//arena_object 是 arena 的一部分

typedef struct pool_header *poolp;

/* Record keeping for arenas. */
struct arena_object {
    /* The address of the arena, as returned by malloc.  Note that 0
     * will never be returned by a successful malloc, and is used
     * here to mark an arena_object that doesn't correspond to an
     * allocated arena.
     */
    uintptr_t address;

    /* Pool-aligned pointer to the next pool to be carved off. */
    block* pool_address;

    /* The number of available pools in the arena:  free pools + never-
     * allocated pools.
     */
    uint nfreepools;

    /* The total number of pools in the arena, whether or not available. */
    uint ntotalpools;

    /* Singly-linked list of available pools. */
    struct pool_header* freepools;

    /* Whenever this arena_object is not associated with an allocated
     * arena, the nextarena member is used to link all unassociated
     * arena_objects in the singly-linked `unused_arena_objects` list.
     * The prevarena member is unused in this case.
     *
     * When this arena_object is associated with an allocated arena
     * with at least one available pool, both members are used in the
     * doubly-linked `usable_arenas` list, which is maintained in
     * increasing order of `nfreepools` values.
     *
     * Else this arena_object is associated with an allocated arena
     * all of whose pools are in use.  `nextarena` and `prevarena`
     * are both meaningless in this case.
     */
    struct arena_object* nextarena;
    struct arena_object* prevarena;
};
```

#### 『未使用』的 arena 和『可用』的 arena

实际上，在Python中，确实会存在多个arena_object构成的集合，但是这个集合并不构成链表，而是构成了一个arena的数组。数组的首地址由arenas维护，这个数组就是Python中的通用小块内存的内存池；另一方面，nextarea和prevarea也确实是用来连接arena_object组成链表的。

pool_header管理的内存与pool_header自身是一块连续的内存，而areana_object与其管理的内存则是分离的。这后面隐藏着这样一个事实：当pool_header被申请时，它所管理的block集合的内存一定也被申请了；但是当aerna_object被申请时，它所管理的pool集合的内存则没有被申请。换句话说，arena_object和pool集合在某一时刻需要建立联系。注意，这个建立联系的时刻是一个关键的时刻，Python从这个时刻一刀切下，将一个arena_object切分为两种状态。

![pool 和 arena 的内存布局区别](https://github.com/BingLau7/blog/blob/master/images/blog_18/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-04-04%20%E4%B8%8B%E5%8D%887.49.39.png?raw=true)

当一个arena的area_object没有与pool集合建立联系时，这时的arena处于“未使用”状态；一旦建立了联系，这时arena就转换到了“可用”状态。对于每一种状态，都有一个arena的链表。“未使用”的arena的链表表头是unused_arena_objects、arena与arena之间通过nextarena连接，是一个单向链表；而“可用”的arena的链表表头是usable_arenas、arena与arena之间通过nextarena和prevarena连接，是一个双向链表。

![某一时刻多个 arena 的一个可能状态](https://github.com/BingLau7/blog/blob/master/images/blog_18/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-04-04%20%E4%B8%8B%E5%8D%887.56.39.png?raw=true)

#### 申请 arena

```C
// [obmalloc.c]

//arenas管理着arena_object的集合
static struct arena_object* arenas = NULL;
//当前arenas中管理的arena_object的个数
static uint maxarenas = 0;
//“未使用的”arena_object链表
static struct arena_object* unused_arena_objects = NULL;
//“可用的”arena_object链表
static struct arena_object* usable_arenas = NULL;
//初始化时需要申请的arena_object的个数
#define INITIAL_ARENA_OBJECTS 16

static struct arena_object* new_arena(void)
{
    struct arena_object* arenaobj; 
    uint excess;  /* number of bytes above pool alignment */
  
    //[1]: 判断是否需要扩充“未使用的”arena_object列表
    if (unused_arena_objects == NULL) { // if one
    uint i;
    uint numarenas;
    size_t nbytes;

    //[2]: 确定本次需要申请的arena_object的个数，并申请内存
    numarenas = maxarenas ? maxarenas << 1 : INITIAL_ARENA_OBJECTS;
    if (numarenas <= maxarenas)
      return NULL;  //overflow（溢出）
    nbytes = numarenas * sizeof(*arenas);
    if (nbytes / sizeof(*arenas) != numarenas)
      return NULL;  //overflow
    arenaobj = (struct arena_object *)realloc(arenas, nbytes);
    if (arenaobj == NULL)
      return NULL;
    arenas = arenaobj;

    //[3]: 初始化新申请的arena_object，并将其放入unused_arena_objects链表中
    for (i = maxarenas; i < numarenas; ++i) {
      arenas[i].address = 0;  /* mark as unassociated */
      arenas[i].nextarena = i < numarenas - 1 ? &arenas[i+1] : NULL;
    }
    /* Update globals. */
    unused_arena_objects = &arenas[maxarenas];
    maxarenas = numarenas;
} // end one

     //[4]: 从unused_arena_objects链表中取出一个“未使用的”arena_object
    arenaobj = unused_arena_objects;
    unused_arena_objects = arenaobj->nextarena;
    assert(arenaobj->address == 0);
  
    //[5]: 申请arena_object管理的内存
    arenaobj->address = (uptr)malloc(ARENA_SIZE);
    ++narenas_currently_allocated;
  
    //[6]: 设置pool集合的相关信息
    arenaobj->freepools = NULL;
    arenaobj->pool_address = (block*)arenaobj->address;
    arenaobj->nfreepools = ARENA_SIZE / POOL_SIZE;
    //将pool的起始地址调整为系统页的边界
    excess = (uint)(arenaobj->address & POOL_SIZE_MASK);
    if (excess != 0) {
    --arenaobj->nfreepools;
    arenaobj->pool_address += POOL_SIZE - excess;
   }
    arenaobj->ntotalpools = arenaobj->nfreepools;

    return arenaobj;
}
```

### 内存池

#### 可用 pool 缓冲池——usedpools

Python 内部默认的小块内存与大块内存的分界点定在512个字节，这个分界点由前面我们看到的名为`SMALL_REQUEST_THRESHOLD` 的符号控制。也就是说，当申请的内存小于512字节时，`PyObject_Malloc` 会在内存池中申请内存；当申请的内存大于512字节时，`PyObject_Malloc` 的行为将退化为 malloc 的行为。

在Python中，pool是一个有size概念的内存管理抽象体，一个pool中的block总是有确定的大小，这个pool总是和某个size class index对应，还记得pool_head中的那个szidx么？而arena是没有size概念的内存管理抽象体，这就意味着，同一个arena，在某个时刻，其内的pool集合可能都是管理的32字节的block；而到了另一时刻，由于系统需要，这个arena可能被重新划分，其中的pool集合可能改为管理64字节的block了，甚至pool集合中一半管理32字节，一半管理64字节。这就决定了在进行内存分配和销毁时，所有的动作都是在pool上完成的。

内存池中的pool，不仅是一个有size概念的内存管理抽象体，而且，更进一步的，它还是一个有状态的内存管理抽象体。一个pool在Python运行的任何一个时刻，总是处于以下三种状态的一种：

-  used状态：pool中至少有一个block已经被使用，并且至少有一个block还未被使用。这种状态的pool受控于Python内部维护的usedpools数组；
-  full状态：pool中所有的block都已经被使用，这种状态的pool在arena中，但不在arena的freepools链表中；
-  empty状态：pool中所有的block都未被使用，处于这个状态的pool的集合通过其pool_header中的nextpool构成一个链表，这个链表的表头就是arena_object中的freepools；

![某个时刻 aerna 中 pool 集合的可能状态](https://github.com/BingLau7/blog/blob/master/images/blog_18/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-04-04%20%E4%B8%8B%E5%8D%888.25.34.png?raw=true)

Python内部维护的usedpools数组是一个非常巧妙的实现，维护着所有的处于used状态的pool。当申请内存时，Python就会通过usedpools寻找到一块可用的（处于used状态的）pool，从中分配一个block。一定有一个与usedpools相关联的机制，完成从申请的内存的大小到size class index之间的转换，否则Python也就无法寻找到最合适的pool了。这种机制与usedpools的结构有密切的关系，我们来看一看usedpools的结构。

一定有一个与usedpools相关联的机制，完成从申请的内存的大小到size class index之间的转换，否则Python也就无法寻找到最合适的pool了。这种机制与usedpools的结构有密切的关系，我们来看一看usedpools的结构。

```C
// [obmalloc.c]
#define NB_SMALL_SIZE_CLASSES   (SMALL_REQUEST_THRESHOLD / ALIGNMENT)

typedef uchar block;

#define PTA(x)  ((poolp )((uchar *)&(usedpools[2*(x)]) - 2*sizeof(block *)))
#define PT(x)   PTA(x), PTA(x)

// 这里的 poolp 指的就是 pool_head
static poolp usedpools[2 * ((NB_SMALL_SIZE_CLASSES + 7) / 8) * 8] = {
    PT(0), PT(1), PT(2), PT(3), PT(4), PT(5), PT(6), PT(7)
#if NB_SMALL_SIZE_CLASSES > 8 //指明了在当前的配置之下，一共有多少个 size class
    , PT(8), PT(9), PT(10), PT(11), PT(12), PT(13), PT(14), PT(15)
    …… 
#endif
}
 
```

其中的`NB_SMALL_SIZE_CLASSES`指明了在当前的配置之下，一共有多少个size class。

```C
// [obmalloc.c]
#define NB_SMALL_SIZE_CLASSES   (SMALL_REQUEST_THRESHOLD / ALIGNMENT)
```



![usedpools 数组](https://github.com/BingLau7/blog/blob/master/images/blog_18/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-04-04%20%E4%B8%8B%E5%8D%888.29.02.png?raw=true)

Python会首先获得 size class index，通过 size = (uint )(nbytes - 1) >> ALIGNMENT_SHIFT，得到 size class index 为3。在usedpools 中，寻找第3+3=6个元素，发现 usedpools[6] 的值是指向 usedpools[4] 的地址。有些迷惑了，对吧？好了，现在对照 pool_header 的定义来看一看 usedpools[6] -> nextpool 这个指针指向哪里了呢？是从 usedpools[6]（即usedpools+4）开始向后偏移8个字节（一个ref的大小加上一个freeblock的大小）后的内存，不正是 usedpools[6] 的地址（即usedpools+6）吗？这是Python内部使用的一个 trick。

想象一下，当我们手中有一个size class为32字节的pool，想要将其放入这个usedpools中时，需要怎么做呢？从上面的描述可以看到，只需要进行usedpools[i+i]->nextpool = pool即可，其中i为size class index，对应于32字节，这个i为3。当下次需要访问size class为32字节（size class index为3）的pool时，只需要简单地访问usedpool[3+3]就可以得到了。Python正是使用这个usedpools快速地从众多的pool中快速地寻找到一个最适合当前内存需求的pool，从中分配一块block。

```C
// [obmalloc.c]
void* PyObject_Malloc(size_t nbytes)
{
    block *bp;
    poolp pool;
    poolp next;
    uint size;
    if ((nbytes - 1) < SMALL_REQUEST_THRESHOLD) {
        LOCK();
        //获得size class index
        size = (uint )(nbytes - 1) >> ALIGNMENT_SHIFT;
        pool = usedpools[size + size];
        //usedpools中有可用的pool
        if (pool != pool->nextpool) {
            ……//usedpools中有可用的pool
        }
        …… //usedpools中无可用pool，尝试获取empty状态pool
}
```

#### Pool 的初始化

当 Python 启动之后，在 usedpools 这个小块空间内存池中，并不存在任何可用的内存，准确地说，不存在任何可用的pool。在这里，Python 采用了延迟分配的策略，即当我们确实开始申请小块内存时，Python 才开始建立这个内存池。

考虑一下这样的情况，当申请32字节内存时，从“可用的” arena 中取出其中一个 pool 用作32字节的 pool 。当下一次内存分配请求分配64字节的内存时，Python 可以直接使用当前“可用的” arena 的另一个 pool 即可。这正如我们前面所说， arena 没有 size class 的属性，而 pool 才有（见下面代码）。

```C
// [obmalloc.c]
void * PyObject_Malloc(size_t nbytes)
{
    block *bp; 
    poolp pool;
    poolp next;
    uint size;

  if ((nbytes - 1) < SMALL_REQUEST_THRESHOLD) {
    LOCK();
    size = (uint)(nbytes - 1) >> ALIGNMENT_SHIFT;
    pool = usedpools[size + size];
    if (pool != pool->nextpool) {
      …… //usedpools中有可用的pool
    }
    //usedpools中无可用pool，尝试获取empty状态pool
    //[1]: 如果usable_arenas链表为空，则创建链表
    if (usable_arenas == NULL) {
      //申请新的arena_object，并放入usable_arenas链表
      usable_arenas = new_arena();
      usable_arenas->nextarena = usable_arenas->prevarena = NULL;
    }

    //[2]: 从usable_arenas链表中第一个arena的freepools中抽取一个可用的pool
    pool = usable_arenas->freepools;
    if (pool != NULL) {
      usable_arenas->freepools = pool->nextpool;
      //[3]: 调整usable_arenas链表中第一个arena中的可用pool数量
      //如果调整后数量为0，则将该arena从usable_arenas链表中摘除
      --usable_arenas->nfreepools;
      if (usable_arenas->nfreepools == 0) {
        usable_arenas = usable_arenas->nextarena;
        if (usable_arenas != NULL) {
          usable_arenas->prevarena = NULL;
        }
      }
    init pool:
            ……
}
```

##### 初始化之一

好了，现在我们手里有了一块用于32字节内存分配的pool，为了以后提高内存分配的效率，我们需要将这个pool放入到usedpools中。这一步，叫做init pool。

```C
// [obmalloc.c]

#define ROUNDUP(x)    (((x) + ALIGNMENT_MASK) & ~ALIGNMENT_MASK)
#define POOL_OVERHEAD   ROUNDUP(sizeof(struct pool_header))
void * PyObject_Malloc(size_t nbytes) {
……
init_pool:
            //[1]: 将pool放入usedpools中
            next = usedpools[size + size]; /* == prev */
            pool->nextpool = next;
            pool->prevpool = next;
            next->nextpool = pool;
            next->prevpool = pool;
            pool->ref.count = 1;
  
            //[2]：pool在之前就具有正确的size结构，直接返回pool中的一个block
            if (pool->szidx == size) {
                bp = pool->freeblock;
                pool->freeblock = *(block **)bp;
                UNLOCK();
                return (void *)bp;
            }
            
            //[3]： 初始化pool header，将freeblock指向第二个block，返回第一个block
            pool->szidx = size;
            size = INDEX2SIZE(size);
            bp = (block *)pool + POOL_OVERHEAD;
            pool->nextoffset = POOL_OVERHEAD + (size << 1);
            pool->maxnextoffset = POOL_SIZE - size;
            pool->freeblock = bp + size;
            *(block **)(pool->freeblock) = NULL;
            UNLOCK();
            return (void *)bp;
……
}
```

在什么样的情况下才会发生一个 pool 从 empty 状态转换为 used 状态呢？假设申请的内存的 size class index 为 i，且 usedpools[i+i] 处没有处于 used 状态的 pool ，同时在 Python 维护的全局变量 freepools 中还有处于 empty 的 pool ，那么位于 freepools 所维护的 pool 链表头部的 pool 将被取出来，放入 usedpools 中，并从其内部分配一块 block 。同时，这个 pool 也就从 empty 状态转换到了 used 状态。下面我们看一看这个行为在代码中是如何体现的。

```C
// [obmalloc.c]
    ……
        pool = usable_arenas->freepools;
        if (pool != NULL) {
            usable_arenas->freepools = pool->nextpool;
            …… //调整usable_arenas->nfreepools和usable_arenas自身
            [init_pool] 
        }
 
```

##### 初始化之二

我们现在可以来看看，当PyObject_Malloc从new_arena中得到一个新的arena后，是怎么样来初始化其中的pool集合，并最终完成PyObject_Malloc函数的分配一个block这个终极任务的。

```C
// [obmalloc.c]
#define DUMMY_SIZE_IDX    0xffff  /* size class of newly cached pools */
void * PyObject_Malloc(size_t nbytes)
{
    block *bp; 
    poolp pool;
    poolp next;
    uint size;
    ……
    //[1]：从arena中取出一个新的pool
    pool = (poolp)usable_arenas->pool_address;
    // 设置 pool 中的 arenaindex，这个 index 实际上就是 pool 所在的 arena 位于 arenas 所指的数组的序号。用于判断一个 block 是否在某个 pool 中。
    pool->arenaindex = usable_arenas - arenas;
    // 随后 Python 将新得到的 pool 的 szidx 设置为 0xffff，表示从没管理过 block 集合。
    pool->szidx = DUMMY_SIZE_IDX;
    // 调整刚获得的 arena 中的 pools 集合，甚至可能调整 usable_arenas
    usable_arenas->pool_address += POOL_SIZE;
    --usable_arenas->nfreepools;

    if (usable_arenas->nfreepools == 0) {
    /* Unlink the arena:  it is completely allocated. */
    usable_arenas = usable_arenas->nextarena;
    if (usable_arenas != NULL) {
      usable_arenas->prevarena = NULL;
    }
    }
    goto init_pool;
    ……
}
```

#### block 的释放


pool 的状态变更最为常见的还是之前是 used 之后也是 used：

在pool的状态保持used状态这种情况下，Python仅仅将block重新放入到自由block链表中，并调整了pool中的ref.count这个引用计数。

```C
// [obmalloc.c]
void PyObject_Free(void *p)
{
    poolp pool;
    block *lastfree;
    poolp next, prev;
    uint size;

    pool = POOL_ADDR(p);
    if (Py_ADDRESS_IN_RANGE(p, pool)) {
        //设置离散自由block链表
        *(block **)p = lastfree = pool->freeblock;
        pool->freeblock = (block *)p;
        if (lastfree) { //lastfree有效，表明当前pool不是处于full状态
             if (--pool->ref.count != 0) { //pool不需要转换为empty状态
                return;
            }
            ……
        }
        ……
    }

    //待释放的内存在PyObject_Malloc中是通过malloc获得的
    //所以要归还给系统
    free(p);
}
```

当我们释放一个 block 后，可能会引起 pool 的状态的转变，这种转变可分为两种情况：

- full状态转变为used状态

   仅仅是将 pool 重新链回到 usedpools 中即可

   ```C
   [obmalloc.c]
   void PyObject_Free(void *p)
   {
       poolp pool;
       block *lastfree;
       poolp next, prev;
       uint size;

       pool = POOL_ADDR(p);
       if (Py_ADDRESS_IN_RANGE(p, pool)) {
           ……
           //当前pool处于full状态，在释放一块block后，需将其转换为used状态，并重新
           //链入usedpools的头部
           --pool->ref.count;
           size = pool->szidx;
           next = usedpools[size + size];
           prev = next->prevpool;
           /* insert pool before next:   prev <-> pool <-> next */
           pool->nextpool = next;
           pool->prevpool = prev;
           next->prevpool = pool;
           prev->nextpool = pool;
           return;
       }
       ……
   }
   ```

- used状态转变为empty状态

   首先 Python 要做的是将empty状态的 pool 链入到 freepools 中去

   ```C
   // [obmalloc.c]
   void PyObject_Free(void *p)
   {
       poolp pool;
       block *lastfree;
       poolp next, prev;
       uint size;

       pool = POOL_ADDR(p);
       if (Py_ADDRESS_IN_RANGE(p, pool)) {
           *(block **)p = lastfree = pool->freeblock;
           pool->freeblock = (block *)p;
           if (lastfree) { 
                struct arena_object* ao; 
                uint nf;  //ao->nfreepools 
                if (--pool->ref.count != 0) { 
                   return;
               }
               // 将pool放入freepools维护的链表中
               // 这里隐藏着一个类似于内存泄露的问题：arena 从来不释放 pool
               ao = &arenas[pool->arenaindex];
               pool->nextpool = ao->freepools;
               ao->freepools = pool;
               nf = ++ao->nfreepools;
               ……
           }
           ……
   }
   ……
   }
   ```

   现在开始处理 arena，分为四种情况：

   -  如果arena中所有的pool都是empty的，释放pool集合占用的内存。

      ```C
      // [obmalloc.c]
      void PyObject_Free(void *p)
      {
          poolp pool;
          block *lastfree;
          poolp next, prev;
          uint size;

          pool = POOL_ADDR(p);
          struct arena_object* ao;  
          uint nf;  //ao->nfreepools 
          ……
          //将pool放入freepools维护的链表中
          ao = &arenas[pool->arenaindex];
          pool->nextpool = ao->freepools;
          ao->freepools = pool;
          nf = ++ao->nfreepools;
          if (nf == ao->ntotalpools) {
              //调整usable_arenas链表
              if (ao->prevarena == NULL) {
                  usable_arenas = ao->nextarena;
              }
              else {
                  ao->prevarena->nextarena = ao->nextarena;
              }
        
              if (ao->nextarena != NULL) {
                  ao->nextarena->prevarena = ao->prevarena;
              }
              //调整unused_arena_objects链表
              ao->nextarena = unused_arena_objects;
              unused_arena_objects = ao;
              //释放内存
              free((void *)ao->address);
              //设置address，将arena的状态转为“未使用”
              ao->address = 0;
              --narenas_currently_allocated;
          }
          ……
      }
      ```

   - 如果之前arena中没有了empty的pool，那么在usable_arenas链表中就找不到该arena，由于现在arena中有了一个pool，所以需要将这个aerna链入到usable_arenas链表的表头。

   - 若arena中的empty的pool个数为n，则从usable_arenas开始寻找arena可以插入的位置，将arena插入到usable_arenas。这个操作的原因是由于usable_arenas实际上是一个有序的链表，从表头开始往后，每一个arena中的empty的pool的个数，即nfreepools，都不能大于前面的arena，也不能小于前面的arena。保持这种有序性的原因是分配block时，是从usable_arenas的表头开始寻找可用的arena的，这样，就能保证如果一个arena的empty pool数量越多，它被使用的机会就越少。因此，它最终释放其维护的pool集合的内存的机会就越大，这样就能保证多余的内存会被归还给系统。

   - 其他情况，不进行任何对arena的处理。

   ​

#### 内存池全景

![Python 的小块内存的内存池全景](https://github.com/BingLau7/blog/blob/master/images/blog_18/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-04-04%20%E4%B8%8B%E5%8D%889.59.26.png?raw=true)

## 参考资料

《Python 源码解析》
