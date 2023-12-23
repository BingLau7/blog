---
title: yield与yield from备忘录
date: 2017-03-21 16:49:05
tags:
    - Python
categories: 工具
---

对于 `yield`与`yield from`的语法，官方给出的解释[Link](https://docs.python.org/3.6/reference/simple_stmts.html#yield)

<!-- more -->

```python
yield <expr>
yield from <expr>
```

等于

```python
(yield <expr>)
(yield from <expr>)
```



要理解`yield`需要先介绍几个概念

### `Iterables`（可迭代对象）

>  An object capable of returning its members one at a time. Examples of iterables include all sequence types (such as [`list`](https://docs.python.org/3.6/library/stdtypes.html#list), [`str`](https://docs.python.org/3.6/library/stdtypes.html#str), and [`tuple`](https://docs.python.org/3.6/library/stdtypes.html#tuple)) and some non-sequence types like [`dict`](https://docs.python.org/3.6/library/stdtypes.html#dict), [file objects](https://docs.python.org/3.6/glossary.html#term-file-object), and objects of any classes you define with an [`__iter__()`](https://docs.python.org/3.6/reference/datamodel.html#object.__iter__) or [`__getitem__()`](https://docs.python.org/3.6/reference/datamodel.html#object.__getitem__) method. Iterables can be used in a [`for`](https://docs.python.org/3.6/reference/compound_stmts.html#for) loop and in many other places where a sequence is needed ([`zip()`](https://docs.python.org/3.6/library/functions.html#zip), [`map()`](https://docs.python.org/3.6/library/functions.html#map), ...).
>
>   When an iterable object is passed as an argument to the built-in function [`iter()`](https://docs.python.org/3.6/library/functions.html#iter), it returns an iterator for the object. This iterator is good for one pass over the set of values. When using iterables, it is usually not necessary to call [`iter()`](https://docs.python.org/3.6/library/functions.html#iter) or deal with iterator objects yourself. The `for` statement does that automatically for you, creating a temporary unnamed variable to hold the iterator for the duration of the loop. See also [iterator](https://docs.python.org/3.6/glossary.html#term-iterator), [sequence](https://docs.python.org/3.6/glossary.html#term-sequence), and [generator](https://docs.python.org/3.6/glossary.html#term-generator).

```Python
>>> mylist = [x*x for x in range(3)]
>>> for i in mylist :
...    print(i)
0
1
4
```

所有你可以使用 `for .. in ..` 语法的叫做一个迭代器：列表，字符串，文件……你经常使用它们是因为你可以如你所愿的读取其中的元素，但是你把所有的值都存储到了内存中，如果你有大量数据的话这个方式并不是你想要的。

### `Generators`（生成器）

>  A function which returns a [generator iterator](https://docs.python.org/3.6/glossary.html#term-generator-iterator). It looks like a normal function except that it contains **[`yield`](https://docs.python.org/3.6/reference/simple_stmts.html#yield) expressions** for producing a series of values usable in a for-loop or that can be retrieved one at a time with the [`next()`](https://docs.python.org/3.6/library/functions.html#next) function.
>
>  Usually refers to a generator function, but may refer to a *generator iterator* in some contexts. In cases where the intended meaning isn’t clear, using the full terms avoids ambiguity.

生成器是可以迭代的，但是你 **只可以读取它一次** ，因为它并不把所有的值放在内存中，它是实时地生成数据:

```Python
>>> mygenerator = (x*x for x in range(3))
>>> for i in mygenerator :
...    print(i)
0
1
4
```

看起来除了把 `[]` 换成 `()` 外没什么不同。但是，你不可以再次使用 `for i inmygenerator` , 因为生成器只能被迭代一次：先计算出0，然后继续计算1，然后计算4，一个跟一个的…

其实这里跟**流的概念**（计算机程序的构造与解释）是一样的，所谓流就是一个语法糖而已，如果用Python来实现，应该可以这么做:

```Python
class Generator:
    def __init__(self, l):
        self.l = l

    def __iter__(self):
        return self

    def __next__(self):
        if len(self.l) > 0:
            return self.l.pop(0)
        raise StopIteration()
        

In [1]: for i in Generator([1, 2, 3, 4]):
   ...:     print(i)
   ...:
1
2
3
4
```

当然，我这只是简单范例，以后我会再结合Python源码来说明生成器是如何实现的（挖了个大坑）。

### `yield`语法

先给出官方文档连接（[Link](https://docs.python.org/3.6/reference/expressions.html#yield-expressions)）

`yield` 是一个类似 `return` 的关键字，只是这个函数返回的是个生成器。

```Python
>>> def createGenerator() :
...    mylist = range(3)
...    for i in mylist :
...        yield i*i
...
>>> mygenerator = createGenerator() # create a generator
>>> print(mygenerator) # mygenerator is an object!
<generator object createGenerator at 0xb7555c34>
>>> for i in mygenerator:
...     print(i)
0
1
4
```

这个例子没什么用途，但是它让你知道，这个函数会返回一大批你只需要读一次的值.

为了精通 `yield` ,你必须要理解：**当你调用这个函数的时候，函数内部的代码并不立马执行** ，这个函数只是返回一个生成器对象，这有点蹊跷不是吗。

那么，函数内的代码什么时候执行呢？当你使用for进行迭代的时候.

现在到了关键点了！

第一次迭代中你的函数会执行，从开始到达 `yield` 关键字，然后返回 `yield` 后的值作为第一次迭代的返回值. 然后，每次执行这个函数都会继续执行你在函数内部定义的那个循环的下一次，再返回那个值，直到没有可以返回的。

如果生成器内部没有定义 `yield` 关键字，那么这个生成器被认为成空的。这种情况可能因为是循环进行没了，或者是没有满足 `if/else` 条件。

看到上述解释，你可以这样认为，`yield`是实现一个语法糖，其语法是将这个函数本身传了出去，并生成一个对象，这个对象是一个生成器，其会执行一次`yield`封装的函数然后将结果作为下一次的返回值（第一次的返回值是直接返回，或者这样更好实现，就是如果检测到函数有`yield`关键字则直接封装为一个生成器对象，然后调用就是先执行一次`yield`封装的函数，然后再返回数据）。

### `yield from`

该语法是在[PEP380](https://www.python.org/dev/peps/pep-0380/)中被定义的。总之大意是原本的yield语句只能将CPU控制权 还给直接调用者，当你想要将一个generator或者coroutine里带有 yield语句的逻辑重构到另一个generator（原文是subgenerator） 里的时候，会非常麻烦，因为外面的generator要负责为里面的 generator做消息传递；所以某人有个想法是让python把消息传递封装起来，使其对开发者透明，于是就有了`yield from`。

`yield from x()`其中的x应该返回一个可迭代的对象(`Iterables`)，这样整个表达式会返回一个生成器(`Generators`)，我们将整个表达式称为y(就是说`y=lambda:yield from x()`)。

>  The full semantics of the yield from expression can be described in terms of the generator protocol as follows:
>
>  -  Any values that the iterator yields（x） are passed directly to the caller（y）.
>  -  Any values sent to the delegating generator using send() are passed directly to the iterator. If the sent value is None, the iterator's \_\_next\_\_() method is called. If the sent value is not None, the iterator's send() method is called. If the call raises StopIteration, the delegating generator is resumed. Any other exception is propagated to the delegating generator.
>  -  Exceptions other than GeneratorExit thrown into the delegating generator are passed to the throw() method of the iterator. If the call raises StopIteration, the delegating generator is resumed. Any other exception is propagated to the delegating generator.
>  -  If a GeneratorExit exception is thrown into the delegating generator, or the close() method of the delegating generator is called, then the close() method of the iterator is called if it has one. If this call results in an exception, it is propagated to the delegating generator. Otherwise, GeneratorExit is raised in the delegating generator.
>  -  The value of the yield from expression is the first argument to the StopIteration exception raised by the iterator when it terminates.
>  -  return expr in a generator causes StopIteration(expr) to be raised upon exit from the generator.

```Python
In [1]: def reader():
   ...:     """A generator that fakes a read from a file, socket, etc."""
   ...:     for i in range(4):
   ...:         yield '<< %s' % i
   ...:

In [2]: def reader_wrapper(g):
   ...:     # Manually iterate over data produced by reader
   ...:     for v in g:
   ...:         yield v
   ...:

In [3]: wrap = reader_wrapper(reader())
   ...: for i in wrap:
   ...:     print(i)
   ...:
<< 0
<< 1
<< 2
<< 3
```

```Python
In [4]: def reader_wrapper(g):
   ...:     yield from g
   ...:

In [5]: wrap = reader_wrapper(reader())
   ...: for i in wrap:
   ...:     print(i)
   ...:
<< 0
<< 1
<< 2
<< 3
```

这篇文章作为介绍Python的`asyncio`的基础文章。



参考文章：

-  Python官方文档
-  PEP380
-  [In practice, what are the main uses for the new “yield from” syntax in Python 3.3?](http://stackoverflow.com/questions/9708902/in-practice-what-are-the-main-uses-for-the-new-yield-from-syntax-in-python-3)
-  [What does the “yield” keyword do?](http://stackoverflow.com/questions/231767/what-does-the-yield-keyword-do)
