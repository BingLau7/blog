---
title: Python整型对象创建
date: 2017-02-01 16:50:44
tags:
    - Python
categories: 源码分析
---

### 整数类型

Python3 中无论整数还是长整数统统使用 PyLongObject 类型来代替

```C
typedef struct _longobject PyLongObject; /* Revealed in longintrepr.h */

/* Long integer representation.
   The absolute value of a number is equal to
   	SUM(for i=0 through abs(ob_size)-1) ob_digit[i] * 2**(SHIFT*i)
   Negative numbers are represented with ob_size < 0;
   zero is represented by ob_size == 0.
   In a normalized number, ob_digit[abs(ob_size)-1] (the most significant
   digit) is never zero.  Also, in all cases, for all valid i,
   	0 <= ob_digit[i] <= MASK.
   The allocation function takes care of allocating extra memory
   so that ob_digit[0] ... ob_digit[abs(ob_size)-1] are actually available.

   CAUTION:  Generic code manipulating subtypes of PyVarObject has to
   aware that ints abuse  ob_size's sign bit.
*/

struct _longobject {
	PyObject_VAR_HEAD
	digit ob_digit[1];
};

#define PyObject_VAR_HEAD      PyVarObject ob_base;

typedef struct {
    PyObject ob_base;
    Py_ssize_t ob_size; /* Number of items in variable part */
} PyVarObject;

/* Nothing is actually declared to be a PyObject, but every pointer to
 * a Python object can be cast to a PyObject*.  This is inheritance built
 * by hand.  Similarly every pointer to a variable-size Python object can,
 * in addition, be cast to PyVarObject*.
 */
typedef struct _object {
    _PyObject_HEAD_EXTRA
    Py_ssize_t ob_refcnt;
    struct _typeobject *ob_type;
} PyObject;

#ifdef Py_TRACE_REFS
/* Define pointers to support a doubly-linked list of all live heap objects. */
#define _PyObject_HEAD_EXTRA            \
    struct _object *_ob_next;           \
    struct _object *_ob_prev;

#define _PyObject_EXTRA_INIT 0, 0,

#else
#define _PyObject_HEAD_EXTRA
#define _PyObject_EXTRA_INIT
#endif
```

由于`Py_TRACE_REFS`在release状态下是不会定义的，所以整个数据结构就会特别简单:

```C
typedef struct _object {
    Py_ssize_t ob_refcnt;
    struct _typeobject *ob_type;
} PyObject;
```

小数优化

```C
#ifndef NSMALLPOSINTS
#define NSMALLPOSINTS           257
#endif
#ifndef NSMALLNEGINTS
#define NSMALLNEGINTS           5
#endif

#if NSMALLNEGINTS + NSMALLPOSINTS > 0
/* Small integers are preallocated in this array so that they
   can be shared.
   The integers that are preallocated are those in the range
   -NSMALLNEGINTS (inclusive) to NSMALLPOSINTS (not inclusive).
*/
static PyLongObject small_ints[NSMALLNEGINTS + NSMALLPOSINTS];
#ifdef COUNT_ALLOCS
Py_ssize_t quick_int_allocs, quick_neg_int_allocs;
#endif

static PyObject *
get_small_int(sdigit ival)
{
    PyObject *v;
    assert(-NSMALLNEGINTS <= ival && ival < NSMALLPOSINTS);
    v = (PyObject *)&small_ints[ival + NSMALLNEGINTS];
    Py_INCREF(v);
#ifdef COUNT_ALLOCS
    if (ival >= 0)
        quick_int_allocs++;
    else
        quick_neg_int_allocs++;
#endif
    return v;
}

#define CHECK_SMALL_INT(ival) \
    do if (-NSMALLNEGINTS <= ival && ival < NSMALLPOSINTS) { \
        return get_small_int((sdigit)ival); \
    } while(0)
```

创建函数

```C
PyLongObject *
_PyLong_New(Py_ssize_t size)
{
    PyLongObject *result;
    /* Number of bytes needed is: offsetof(PyLongObject, ob_digit) +
       sizeof(digit)*size.  Previous incarnations of this code used
       sizeof(PyVarObject) instead of the offsetof, but this risks being
       incorrect in the presence of padding between the PyVarObject header
       and the digits. */
    if (size > (Py_ssize_t)MAX_LONG_DIGITS) {
        PyErr_SetString(PyExc_OverflowError,
                        "too many digits in integer");
        return NULL;
    }
    // offsetof(type, member-designator) 会生成一个类型为 size_t 的整型常量，它是一个结构成员相对于结构开头的字节偏移量。成员是由 member-designator 给定的，结构的名称是在 type 中给定的。
    result = PyObject_MALLOC(offsetof(PyLongObject, ob_digit) +
                             size*sizeof(digit));
    if (!result) {
        PyErr_NoMemory();
        return NULL;
    }
    return (PyLongObject*)PyObject_INIT_VAR(result, &PyLong_Type, size);
}
```

其中 `PyObject_MALLOC`是与 `WITH_PYMALLOC`定义相关

>  Pymalloc, a specialized object allocator written by Vladimir Marangozov, was a feature added to Python 2.1. Pymalloc is intended to be faster than the system malloc() and to have less memory overhead for allocation patterns typical of Python programs. The allocator uses C's malloc() function to get large pools of memory and then fulfills smaller memory requests from these pools.
>
>  In 2.1 and 2.2, pymalloc was an experimental feature and wasn't enabled by default; you had to explicitly enable it when compiling Python by providing the **--with-pymalloc** option to the **configure** script. In 2.3, pymalloc has had further enhancements and is now enabled by default; you'll have to supply**--without-pymalloc** to disable it.

可以看出这东西是默认开启的，所以从代码可知调用的是`_PyObject_Malloc`方法，再追踪源码发现其最终调用的是如下方法：

```C
void *
PyObject_Malloc(size_t size)
{
    /* see PyMem_RawMalloc() */
    if (size > (size_t)PY_SSIZE_T_MAX)
        return NULL;
    return _PyObject.malloc(_PyObject.ctx, size);
}

static void *
_PyObject_Malloc(void *ctx, size_t nbytes)
{
    return _PyObject_Alloc(0, ctx, 1, nbytes);
}

/* malloc.  Note that nbytes==0 tries to return a non-NULL pointer, distinct
 * from all other currently live pointers.  This may not be possible.
 */

/*
 * The basic blocks are ordered by decreasing execution frequency,
 * which minimizes the number of jumps in the most common cases,
 * improves branching prediction and instruction scheduling (small
 * block allocations typically result in a couple of instructions).
 * Unless the optimizer reorders everything, being too smart...
 */

static void *
_PyObject_Alloc(int use_calloc, void *ctx, size_t nelem, size_t elsize)
{
    /*
    解释一下 ctx, 这是 PyMemAllocatorEx 的 ctx，
   static PyMemAllocatorEx _PyObject = {
#ifdef Py_DEBUG
    &_PyMem_Debug.obj, PYDBG_FUNCS
#else
    NULL, PYOBJ_FUNCS
#endif
    };
     来定义，可以看出，这个是为了 Debug 来操作的
    */

    size_t nbytes;
    block *bp;
    poolp pool;
    poolp next;
    uint size;

    // 记录分配了一块 block
    _Py_AllocatedBlocks++;

    assert(nelem <= PY_SSIZE_T_MAX / elsize);
    // 创建对象个数乘以创建每个对象所需要的字节数
    nbytes = nelem * elsize;

    // 是否开启 valgrind 来调试程序
#ifdef WITH_VALGRIND
    if (UNLIKELY(running_on_valgrind == -1))
        running_on_valgrind = RUNNING_ON_VALGRIND;
    if (UNLIKELY(running_on_valgrind))
        goto redirect;
#endif

    if (nelem == 0 || elsize == 0)
        goto redirect;

    // 小块内存与大块内存分配分界处
    if ((nbytes - 1) < SMALL_REQUEST_THRESHOLD) {
        // 这里有些不能理解，看注释是
        /*
         * Locking
         *
         * To reduce lock contention, it would probably be better to refine the
         * crude function locking with per size class locking. I'm not positive
         * however, whether it's worth switching to such locking policy because
         * of the performance penalty it might introduce.
         *
         * The following macros describe the simplest (should also be the fastest)
         * lock object on a particular platform and the init/fini/lock/unlock
         * operations on it. The locks defined here are not expected to be recursive
         * because it is assumed that they will always be called in the order:
         * INIT, [LOCK, UNLOCK]*, FINI.
         */

        /*
         * Python's threads are serialized, so object malloc locking is disabled.
         */
        // 但是不能理解其加锁方式
        LOCK();
        /*
         * Most frequent paths first
         */
        // 得到 size_class_index
        size = (uint)(nbytes - 1) >> ALIGNMENT_SHIFT;
        // 从 userdpool 中找到匹配的pool然后准备存到里面去
        pool = usedpools[size + size];
        if (pool != pool->nextpool) {
            // 这个是最终都要进入的地方
            // 如果 usedpool  中有可用的 pool
            /*
             * There is a used pool for this size class.
             * Pick up the head block of its free list.
             */
            ++pool->ref.count;
            bp = pool->freeblock;
            assert(bp != NULL);
            if ((pool->freeblock = *(block **)bp) != NULL) {
                UNLOCK();
                if (use_calloc)
                    memset(bp, 0, nbytes);
                return (void *)bp;
            }
            /*
             * Reached the end of the free list, try to extend it.
             */
            if (pool->nextoffset <= pool->maxnextoffset) {
                /* There is room for another block. */
                pool->freeblock = (block*)pool +
                                  pool->nextoffset;
                pool->nextoffset += INDEX2SIZE(size);
                *(block **)(pool->freeblock) = NULL;
                UNLOCK();
                if (use_calloc)
                    memset(bp, 0, nbytes);
                return (void *)bp;
            }
            /* Pool is full, unlink from used pools. */
            next = pool->nextpool;
            pool = pool->prevpool;
            next->prevpool = pool;
            pool->nextpool = next;
            UNLOCK();
            if (use_calloc)
                memset(bp, 0, nbytes);
            // bp 返回的实际是一个地址，这个地址之后有将近 4KB 的内存实际上都是可用的，
            // 但是可用肯定申请内存的函数只会使用[bp, bp+size] 这个区间的内存，
            // 这是由 size class index 可用保证的。
            return (void *)bp;
        }

        //usedpools中无可用pool，尝试获取empty状态pool
        /* There isn't a pool of the right size class immediately
         * available:  use a free pool.
         */
        if (usable_arenas == NULL) {
            // 如果usable_arenas链表为空，则创建链表
            /* No arena has a free pool:  allocate a new arena. */
        // WITH_MEMORY_LIMITS 编译时候打开会激活 SMALL_MEMORY_LIMIT 符号，该符号限制了 arena 的个数
#ifdef WITH_MEMORY_LIMITS
            if (narenas_currently_allocated >= MAX_ARENAS) {
                UNLOCK();
                goto redirect;
            }
#endif
            // 申请新的arena_object，并放入usable_arenas链表
            usable_arenas = new_arena();
            if (usable_arenas == NULL) {
                UNLOCK();
                goto redirect;
            }
            usable_arenas->nextarena =
                usable_arenas->prevarena = NULL;
        }
        assert(usable_arenas->address != 0);

        /* Try to get a cached free pool. */
        // 从usable_arenas链表中第一个arena的freepools中抽取一个可用的pool
        pool = usable_arenas->freepools;
        if (pool != NULL) {
            /* Unlink from cached pools. */
            usable_arenas->freepools = pool->nextpool;

            /* This arena already had the smallest nfreepools
             * value, so decreasing nfreepools doesn't change
             * that, and we don't need to rearrange the
             * usable_arenas list.  However, if the arena has
             * become wholly allocated, we need to remove its
             * arena_object from usable_arenas.
             */
            //调整usable_arenas链表中第一个arena中的可用pool数量
            //如果调整后数量为0，则将该arena从usable_arenas链表中摘除
            --usable_arenas->nfreepools;
            if (usable_arenas->nfreepools == 0) {
                /* Wholly allocated:  remove. */
                assert(usable_arenas->freepools == NULL);
                assert(usable_arenas->nextarena == NULL ||
                       usable_arenas->nextarena->prevarena ==
                       usable_arenas);

                usable_arenas = usable_arenas->nextarena;
                if (usable_arenas != NULL) {
                    usable_arenas->prevarena = NULL;
                    assert(usable_arenas->address != 0);
                }
            }
            else {
                /* nfreepools > 0:  it must be that freepools
                 * isn't NULL, or that we haven't yet carved
                 * off all the arena's pools for the first
                 * time.
                 */
                assert(usable_arenas->freepools != NULL ||
                       usable_arenas->pool_address <=
                       (block*)usable_arenas->address +
                           ARENA_SIZE - POOL_SIZE);
            }
        init_pool:
            /* Frontlink to used pools. */
            // 将 pool 放入 usedpool 中
            next = usedpools[size + size]; /* == prev */
            pool->nextpool = next;
            pool->prevpool = next;
            next->nextpool = pool;
            next->prevpool = pool;
            pool->ref.count = 1;
            if (pool->szidx == size) {
                /* Luckily, this pool last contained blocks
                 * of the same size class, so its header
                 * and free list are already initialized.
                 */
                bp = pool->freeblock;
                assert(bp != NULL);
                pool->freeblock = *(block **)bp;
                UNLOCK();
                if (use_calloc)
                    memset(bp, 0, nbytes);
                return (void *)bp;
            }
            /*
             * Initialize the pool header, set up the free list to
             * contain just the second block, and return the first
             * block.
             */
            // 初始化pool header，将freeblock指向第二个block，返回第一个block
            pool->szidx = size;
            size = INDEX2SIZE(size);
            bp = (block *)pool + POOL_OVERHEAD;
            pool->nextoffset = POOL_OVERHEAD + (size << 1);
            pool->maxnextoffset = POOL_SIZE - size;
            pool->freeblock = bp + size;
            *(block **)(pool->freeblock) = NULL;
            UNLOCK();
            if (use_calloc)
                memset(bp, 0, nbytes);
            return (void *)bp;
        }

        /* Carve off a new pool. */
        assert(usable_arenas->nfreepools > 0);
        assert(usable_arenas->freepools == NULL);
        // 从arena中取出一个新的pool
        pool = (poolp)usable_arenas->pool_address;
        assert((block*)pool <= (block*)usable_arenas->address +
                               ARENA_SIZE - POOL_SIZE);
        // 设置 pool 中的 arenaindex，这个 index 实际上就是 pool 所在的 arena
        // 位于 arenas 所指的数组的序号。用于判断一个 block 是否在某个 pool 中。
        pool->arenaindex = (uint)(usable_arenas - arenas);
        assert(&arenas[pool->arenaindex] == usable_arenas);
        // 随后 Python 将新得到的 pool 的 szidx 设置为 0xffff，表示从没管理过 block 集合。
        pool->szidx = DUMMY_SIZE_IDX;
        // 调整刚获得的 arena 中的 pools 集合，甚至可能调整 usable_arenas
        usable_arenas->pool_address += POOL_SIZE;
        --usable_arenas->nfreepools;

        if (usable_arenas->nfreepools == 0) {
            assert(usable_arenas->nextarena == NULL ||
                   usable_arenas->nextarena->prevarena ==
                   usable_arenas);
            /* Unlink the arena:  it is completely allocated. */
            usable_arenas = usable_arenas->nextarena;
            if (usable_arenas != NULL) {
                usable_arenas->prevarena = NULL;
                assert(usable_arenas->address != 0);
            }
        }

        goto init_pool;
    }

    /* The small block allocator ends here. */

redirect:
    /* Redirect the original request to the underlying (libc) allocator.
     * We jump here on bigger requests, on error in the code above (as a
     * last chance to serve the request) or when the max memory limit
     * has been reached.
     */
    {
        void *result;
        if (use_calloc)
            result = PyMem_RawCalloc(nelem, elsize);
        else
            result = PyMem_RawMalloc(nbytes);
        if (!result)
            _Py_AllocatedBlocks--;
        return result;
    }
}
```



```C
/* Allocate a new arena.  If we run out of memory, return NULL.  Else
 * allocate a new arena, and return the address of an arena_object
 * describing the new arena.  It's expected that the caller will set
 * `usable_arenas` to the return value.
 */
static struct arena_object*
new_arena(void)
{
    struct arena_object* arenaobj;
    uint excess;        /* number of bytes above pool alignment */
    void *address;
    static int debug_stats = -1;

    if (debug_stats == -1) {
        char *opt = Py_GETENV("PYTHONMALLOCSTATS");
        debug_stats = (opt != NULL && *opt != '\0');
    }
    if (debug_stats)
        _PyObject_DebugMallocStats(stderr);

    // 判断是否需要扩充“未使用的”arena_object列表
    if (unused_arena_objects == NULL) {
        uint i;
        uint numarenas;
        size_t nbytes;

        /* Double the number of arena objects on each allocation.
         * Note that it's possible for `numarenas` to overflow.
         */
        // 确定本次需要申请的arena_object的个数，并申请内存
        numarenas = maxarenas ? maxarenas << 1 : INITIAL_ARENA_OBJECTS;
        if (numarenas <= maxarenas)
            return NULL;                /* overflow */
#if SIZEOF_SIZE_T <= SIZEOF_INT
        if (numarenas > SIZE_MAX / sizeof(*arenas))
            return NULL;                /* overflow */
#endif
        nbytes = numarenas * sizeof(*arenas);
        // realloc() 对 ptr 指向的内存重新分配 size 大小的空间，size 可比原来的大或者小
        arenaobj = (struct arena_object *)PyMem_RawRealloc(arenas, nbytes);
        if (arenaobj == NULL)
            return NULL;
        arenas = arenaobj;

        /* We might need to fix pointers that were copied.  However,
         * new_arena only gets called when all the pages in the
         * previous arenas are full.  Thus, there are *no* pointers
         * into the old array. Thus, we don't have to worry about
         * invalid pointers.  Just to be sure, some asserts:
         */
        assert(usable_arenas == NULL);
        assert(unused_arena_objects == NULL);

        /* Put the new arenas on the unused_arena_objects list. */
        for (i = maxarenas; i < numarenas; ++i) {
            arenas[i].address = 0;              /* mark as unassociated */
            arenas[i].nextarena = i < numarenas - 1 ?
                                   &arenas[i+1] : NULL;
        }

        /* Update globals. */
        unused_arena_objects = &arenas[maxarenas];
        maxarenas = numarenas;
    }

    /* Take the next available arena object off the head of the list. */
    // 从unused_arena_objects链表中取出一个“未使用的”arena_object
    assert(unused_arena_objects != NULL);
    arenaobj = unused_arena_objects;
    unused_arena_objects = arenaobj->nextarena;
    assert(arenaobj->address == 0);
    // 申请arena_object管理的内存
    // 这里的 alloc 对应 linux 下就是 malloc
    address = _PyObject_Arena.alloc(_PyObject_Arena.ctx, ARENA_SIZE);
    if (address == NULL) {
        /* The allocation failed: return NULL after putting the
         * arenaobj back.
         */
        arenaobj->nextarena = unused_arena_objects;
        unused_arena_objects = arenaobj;
        return NULL;
    }
    arenaobj->address = (uintptr_t)address;

    ++narenas_currently_allocated;
    ++ntimes_arena_allocated;
    if (narenas_currently_allocated > narenas_highwater)
        narenas_highwater = narenas_currently_allocated;
    arenaobj->freepools = NULL;
    /* pool_address <- first pool-aligned address in the arena
       nfreepools <- number of whole pools that fit after alignment */
    arenaobj->pool_address = (block*)arenaobj->address;
    arenaobj->nfreepools = ARENA_SIZE / POOL_SIZE;
    assert(POOL_SIZE * arenaobj->nfreepools == ARENA_SIZE);
    excess = (uint)(arenaobj->address & POOL_SIZE_MASK);
    if (excess != 0) {
        --arenaobj->nfreepools;
        arenaobj->pool_address += POOL_SIZE - excess;
    }
    arenaobj->ntotalpools = arenaobj->nfreepools;

    return arenaobj;
}
```



具体可以参考以上注释，以上。



### 参考资料

https://docs.python.org/2.3/whatsnew/section-pymalloc.html

《Python 源码解析》
