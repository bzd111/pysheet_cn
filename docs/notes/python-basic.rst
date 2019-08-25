
.. meta::
    :description lang=en: Collect useful snippets of Python
    :keywords: Python, Python Cheat Sheet

============
从头开始
============
这个备忘录的主要目标是收集一些常用的、简单的语义或片段。这节包括了一些我们已经知道但还有点模凌两可的语义或者我们google了一次又一次
的代码片段。此外，因为 **Python2在2020不再维护了**，大部分的代码片段都是基于 **Python3** 的语法。

.. contents:: Table of Contents
    :backlinks: none

Hello world!
------------

当我们学习一门语言的时候，我们通常学习输出
**Hello world!**。在Python中, 我们可以使用其他的方法通过导入 ``__hello__`` 模块。源码可以在这里看到
`frozen.c <https://github.com/python/cpython/blob/master/Python/frozen.c>`_。

.. code-block:: python

    >>> print("Hello world!")
    Hello world!
    >>> import __hello__
    Hello world!
    >>> import __phello__
    Hello world!
    >>> import __phello__.spam
    Hello world!


Python版本
--------------

对于程序员来说，知道当前Python的版本是很重要的。因为并不是每一种语法,都适用于当前版本。
在这种情况下，我们可以得到Python版本通过 ``python -V`` 或者使用 ``sys`` 模块。

.. code-block:: python

    >>> import sys
    >>> print(sys.version)
    3.7.1 (default, Nov  6 2018, 18:46:03)
    [Clang 10.0.0 (clang-1000.11.45.5)]

我们也可以使用 ``platform.python_version`` 获取Python版本。

.. code-block:: python

    >>> import platform
    >>> platform.python_version()
    '3.7.1'

有时, 检查当前的Python版本是重要的，因为我们也许想要使用一些特性在某些特殊的版本。``sys.version_info`` 提供了更多详细的信息关于解释器。
我们可以用它来与我们想要的版本进行比较。

.. code-block:: python

    >>> import sys
    >>> sys.version_info >= (3, 6)
    True
    >>> sys.version_info >= (3, 7)
    False

省略号
--------

`Ellipsis <https://docs.python.org/3/library/constants.html#Ellipsis>`_ 是一个内建的常量。
在Python 3.0之后，我们可以使用 ``...`` 作为 ``Ellipsis``。它也许是最神秘的常量在Python中。基于官方文档,
我们可以用它来扩展切片语法。不过，还有其他一些类型提示，存根文件或函数表达式中的约定。

.. code-block:: python

    >>> ...
    Ellipsis
    >>> ... == Ellipsis
    True
    >>> type(...)
    <class 'ellipsis'>

以下代码段展示了我们可以使用省略号来表示尚未实现的函数或类。

.. code-block:: python

    >>> class Foo: ...
    ...
    >>> def foo(): ...
    ...

if ... elif ... else
--------------------

**if 语句** 用于控制代码流。而不是使用
``switch`` 或者 ``case`` 语句来控制代码的逻辑，Python使用
``if ... elif ... else`` 序列。虽然有人建议我们可以使用
``dict`` 实现 ``switch`` 语句, 这个解决方法可能会引入一些额外的开销比如创建一次性的字典和
破坏代码的可读性。因此，不推荐这个方法。

.. code-block:: python

    >>> import random
    >>> num = random.randint(0, 10)
    >>> if num < 3:
    ...     print("less than 3")
    ... elif num < 5:
    ...     print("less than 5")
    ... else:
    ...     print(num)
    ...
    less than 3

for循环
--------

在Python中, 我们可以直接访问可迭代的对象通过
**for 语句**. 如果我们需要同时获取索引和可迭代对象的元素，例如列表和元组, 使用 ``enumerate`` 会更好比
``range(len(iterable))``。更多的信息可以在
`Looping Techniques <https://docs.python.org/3/tutorial/datastructures.html#looping-techniques>`_ 看到。

.. code-block:: python

    >>> for val in ["foo", "bar"]:
    ...     print(val)
    ...
    foo
    bar
    >>> for idx, val in enumerate(["foo", "bar", "baz"]):
    ...     print(idx, val)
    ...
    (0, 'foo')
    (1, 'bar')
    (2, 'baz')

for ... else ...
----------------

第一次看到 ``else`` 属于一个 ``for`` 循环这种语法，可能会有点奇怪。这个 ``else`` 子句可以帮助我们避免在循环中使用 ``flag`` 变量。
当没有 ``break`` 发生时，循环的 ``else`` 子句才会运行。

.. code-block:: python

    >>> for _ in range(5):
    ...     pass
    ... else:
    ...     print("no break")
    ...
    no break

下面的代码片段会展示使用 ``flag`` 变量和 ``else`` 子句来控制循环的不同。
当循环中发生了 ``break``，我们可以看到 ``else`` 没有运行。

.. code-block:: python

    >>> is_break = False
    >>> for x in range(5):
    ...     if x % 2 == 0:
    ...         is_break = True
    ...         break
    ...
    >>> if is_break:
    ...     print("break")
    ...
    break

    >>> for x in range(5):
    ...     if x % 2 == 0:
    ...         print("break")
    ...         break
    ... else:
    ...     print("no break")
    ...
    break

使用 ``range``
---------------
Python2中 ``range`` 的问题在于，可能会占用大量的缓存，如果我们需要迭代一个循环多次。
所以，在Python2中推荐使用 ``xrange`` 。

.. code-block:: python

    >>> import platform
    >>> import sys
    >>> platform.python_version()
    '2.7.15'
    >>> sys.getsizeof(range(100000000))
    800000072
    >>> sys.getsizeof(xrange(100000000))
    40

在Python3中，内建函数 ``range`` 返回一个可迭代的 **range 对象** 而不是列表。
``range`` 的行为和Python2的 ``xrange`` 是一样的。
因此，如果我们想要在循环中运行一个代码块多次，使用 ``range`` 不会占用太多的内存。
更多的信息可以在这里看到PEP `3100 <https://www.python.org/dev/peps/pep-3100>`_。

.. code-block:: python

    >>> import platform
    >>> import sys
    >>> platform.python_version()
    '3.7.1'
    >>> sys.getsizeof(range(100000000))
    48

while ... else ...
------------------

``else`` 子句属于一个while循环和属于一个for循环的作用是一样的。
我们可以看到，当这个while循坏发生 ``break`` 时，这个 ``else`` 子句没有被运行。

.. code-block:: python

    >>> n = 0
    >>> while n < 5:
    ...     if n == 3:
    ...         break
    ...     n += 1
    ... else:
    ...     print("no break")
    ...

``do while`` 语法
--------------------------

有很多编程语言像C/C++、Ruby或者Javascript,
都提供 ``do while`` 语法。在Python中，没有 ``do while``
语法。然而，我们可以设置条件，然后把 ``break`` 的 ``while`` 循环的最后达到相同的效果。

.. code-block:: python

    >>> n = 0
    >>> while True:
    ...     n += 1
    ...     if n == 5:
    ...         break
    ...
    >>> n
    5

try ... except ... else ...
---------------------------

大多数时间，我们处理错误在 ``except`` 子句，在 ``finally`` 子句处理资源。有趣的是，``try`` 语法也为我们提供了一个
``else`` 子句，避免捕获不受 ``try ... except`` 保护的代码抛出的错误。当 ``try`` 和 ``except`` 之间没有异常发生，
``else`` 子句才会运行。

.. code-block:: python

    >>> try:
    ...     print("No exception")
    ... except:
    ...     pass
    ... else:
    ...     print("Success")
    ...
    No exception
    Success

字符串
------

不想其他的编程语言，Python不支持直接对字符串的元素进行赋值。
因此, 如果非要操作字符串的元素，例如交互元素。我们不得不把字符串转化成列表，
然后在完成一系列元素赋值后，通过 **join** 操作转化为字符串。

.. code-block:: python

    >>> a = "Hello Python"
    >>> l = list(a)
    >>> l[0], l[6] = 'h', 'p'
    >>> ''.join(l)
    'hello python'

列表
----

列表是多功能的容器。Python提供了很多方法，例如：
**负索引**, **切片语法**, or **列表推导** 去操作列表。
下面的代码片段会展示一些常用的列表操作。

.. code-block:: python

    >>> a = [1, 2, 3, 4, 5]
    >>> a[-1]                     # negative index
    5
    >>> a[1:]                     # slicing
    [2, 3, 4, 5]
    >>> a[1:-1]
    [2, 3, 4]
    >>> a[1:-1:2]
    [2, 4]
    >>> a[::-1]                   # reverse
    [5, 4, 3, 2, 1]
    >>> a[0] = 0                  # set an item
    >>> a
    [0, 2, 3, 4, 5]
    >>> a.append(6)               # append an item
    >>> a
    [0, 2, 3, 4, 5, 6]
    >>> del a[-1]                 # del an item
    >>> a
    [0, 2, 3, 4, 5]
    >>> b = [x for x in range(3)] # list comprehension
    >>> b
    [0, 1, 2]
    >>> a + b                     # add two lists
    [0, 2, 3, 4, 5, 0, 1, 2]

字典
----

字典是包含键值对的容器。像列表, Python提供了很多方法，例如：
**字典推导** 去操作字典。在Python 3.6之后，字典保留了健的插入顺序。
下面的代码片段会展示一些常用的字典操作。

.. code-block:: python

    >>> d = {'timmy': 'red', 'barry': 'green', 'guido': 'blue'}
    >>> d
    {'timmy': 'red', 'barry': 'green', 'guido': 'blue'}
    >>> d['timmy'] = "yellow"        # set data
    >>> d
    {'timmy': 'yellow', 'barry': 'green', 'guido': 'blue'}
    >>> del d['guido']               # del data
    >>> d
    >>> 'guido' in d                 # contain data
    False
    {'timmy': 'yellow', 'barry': 'green'}
    >>> {k: v for k ,v in d.items()} # dict comprehension
    {'timmy': 'yellow', 'barry': 'green'}
    >>> d.keys()                     # list all keys
    dict_keys(['timmy', 'barry'])
    >>> d.values()                   # list all values
    dict_values(['yellow', 'green'])

函数
--------

在Python中定义函数是灵活的。我们可以使用
**函数文档**, **默认值**, **可变参数**,
**关键字参数**, **强制关键字参数** 来定义函数。
下面的代码片段展示了一些常用的定义函数的表达式。

.. code-block:: python

    def foo_with_doc():
        """Documentation String."""

    def foo_with_arg(arg): ...
    def foo_with_args(*arg): ...
    def foo_with_kwarg(a, b="foo"): ...
    def foo_with_args_kwargs(*args, **kwargs): ...
    def foo_with_kwonly(a, b, *, k): ...           # python3
    def foo_with_annotations(a: int) -> int: ...   # python3

函数注解
--------------------

我们应该使用 **函数注解** 来表示类型，而不是在函数中写文档来说明入参和返回值的类型。
关于函数注解是在Python3中引入的，更详细的信息可以看
PEP `3017 <https://www.python.org/dev/peps/pep-3107>`_
和 PEP `484 <https://www.python.org/dev/peps/pep-0484/>`_ 。
他们是 **Python3** 里的 **可选** 功能。使用了函数注解的代码，在 **Python2** 中是不兼容的。
我们可以通过存根文件来解决这个问题。另外，我们可以通过 `mypy <http://mypy-lang.org/>`_ 来进行静态类型检查。

.. code-block:: python

    >>> def fib(n: int) -> int:
    ...     a, b = 0, 1
    ...     for _ in range(n):
    ...         b, a = a + b, b
    ...     return a
    ...
    >>> fib(10)
    55

生成器
----------

Python使用 ``yield`` 语法，定义一个 **生成器函数**。
换句话说，当我们调用一个生成器函数时，生成器函数会返回一个 **generator** 取代创建一个迭代的返回值。

.. code-block:: python

    >>> def fib(n):
    ...     a, b = 0, 1
    ...     for _ in range(n):
    ...         yield a
    ...         b, a = a + b, b
    ...
    >>> g = fib(10)
    >>> g
    <generator object fib at 0x10b240c78>
    >>> for f in fib(5):
    ...     print(f)
    ...
    0
    1
    1
    2
    3

生成器派生
--------------------

在Python3.3中采用了 ``yield from`` 表达式。它允许生成器将操作的一部分委托给另一个生成器r。
换句话说, 我们可以在当前的 **生成器函数** 中，**yield** 一个序列 **from** 其他的  **generators**。
更多信息可以看PEP `380 <https://www.python.org/dev/peps/pep-0380>`_。

.. code-block:: python

    >>> def fib(n):
    ...     a, b = 0, 1
    ...     for _ in range(n):
    ...         yield a
    ...         b, a = a + b, b
    ...
    >>> def fibonacci(n):
    ...     yield from fib(n)
    ...
    >>> [f for f in fibonacci(5)]
    [0, 1, 1, 2, 3]

类
-----

Python的类支持很多常见的特性，例如：**类文档**、 **多继承**、
**累变量**、 **实力变量**、 **静态方法**、 **类方法** 。
除此之外, Python还提供了一些特殊的方法来实现 **迭代器**、**上下文管理器** 。
下面的片段会展示一些类的常用定义。

.. code-block:: python

    class A: ...
    class B: ...
    class Foo(A, B):
        """A class document."""

        foo = "class variable"

        def __init__(self, v):
            self.attr = v
            self.__private = "private var"

        @staticmethod
        def bar_static_method(): ...

        @classmethod
        def bar_class_method(cls): ...

        def bar(self):
            """A method document."""

        def bar_with_arg(self, arg): ...
        def bar_with_args(self, *args): ...
        def bar_with_kwarg(self, kwarg="bar"): ...
        def bar_with_args_kwargs(self, *args, **kwargs): ...
        def bar_with_kwonly(self, *, k): ...
        def bar_with_annotations(self, a: int): ...

``async`` / ``await``
---------------------

Python 3.5引入 ``async`` 和 ``await`` 语法。
它们被设计为与事件循环一起使用。
其他一些功能比如 **异步生成器** 在最新的版本有实现。

一个 **协程 函数** (``async def``) 被用于为事件循环创建 **协程** 。
Python提供一个内建的模块 **asyncio**, 用来写并发的代码通过 ``async``/``await`` 语法。
下面的代码片段展示了一些使用 **asyncio** 模块的小例子。
代码必须在Python 3.7或者更高的版本运行。

.. code-block:: python

    import asyncio

    async def http_ok(r, w):
        head = b"HTTP/1.1 200 OK\r\n"
        head += b"Content-Type: text/html\r\n"
        head += b"\r\n"

        body = b"<html>"
        body += b"<body><h1>Hello world!</h1></body>"
        body += b"</html>"

        _ = await r.read(1024)
        w.write(head + body)
        await w.drain()
        w.close()

    async def main():
        server = await asyncio.start_server(
            http_ok, "127.0.0.1", 8888
        )

        async with server:
            await server.serve_forever()

    asyncio.run(main())

避免使用 ``exec`` 和 ``eval``
----------------------------------

下面的代码片段展示了如何使用内建函数 ``exec`` 。
然而，由于一些安全问题和代码可读性的原因，不推荐使用 ``exec`` 和 ``eval`` 。
更多的信息可以看
`Be careful with exec and eval in Python <http://lucumr.pocoo.org/2011/2/1/exec-in-python/>`_
和 `Eval really is dangerous <Eval really is dangerous>`_


.. code-block:: python

    >>> py = '''
    ... def fib(n):
    ...     a, b = 0, 1
    ...     for _ in range(n):
    ...         b, a = b + a, b
    ...     return a
    ... print(fib(10))
    ... '''
    >>> exec(py, globals(), locals())
    55
