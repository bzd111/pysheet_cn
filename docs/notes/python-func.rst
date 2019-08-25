.. meta::
    :description lang=en: Collect useful snippets of Python Function
    :keywords: Python, Python Function, Python Cheat Sheet

========
函数
========

函数可以帮助程序员封装他们的逻辑变成一个任务以避免重复的任务。
在Python中，函数的定义是非常的通用，我们可以使用许多特性，比如装饰器、注解、文档字符串、
默认参数等。在这章备忘录中，收集了很多定义函数的方法和揭开函数中的神秘面纱。


.. contents:: Table of Contents
    :backlinks: none

文档功能
------------------

文档为程序员提供了如何使用这个函数的信息。
文档字符串提供了一种权宜之计去写关于函数可读性高的文档。
PEP `257 <https://www.python.org/dev/peps/pep-0257>`_ 定义许多约定关于文档字符串。
为了避免违反约定，这里有几个工具，比如 `doctest <https://docs.python.org/3/library/doctest.html>`_，
或者 `pydocstyle <https://github.com/PyCQA/pydocstyle>`_ 可以帮助我们检查文档字符串的格式。

.. code-block:: python

    >>> def example():
    ...   """This is an example function."""
    ...   print("Example function")
    ...
    >>> example.__doc__
    'This is an example function.'
    >>> help(example)

默认参数
-----------------

在Python中，定义一个参数是可选的而且具有默认值的函数是十分简单的。
我们可以在定义时分配值，并确保默认参数出现在最后。
We can just assign values in the definition and make
sure the default arguments appear in the end.

.. code-block:: python

    >>> def add(a, b=0):
    ...     return a + b
    ...
    >>> add(1)
    1
    >>> add(1, 2)
    3
    >>> add(1, b=2)
    3

可选参数
----------------

.. code-block:: python

    >>> def example(a, b=None, *args, **kwargs):
    ...     print(a, b)
    ...     print(args)
    ...     print(kwargs)
    ...
    >>> example(1, "var", 2, 3, word="hello")
    1 var
    (2, 3)
    {'word': 'hello'}

解包参数
----------------

.. code-block:: python

    >>> def foo(a, b, c='BAZ'):
    ...     print(a, b, c)
    ...
    >>> foo(*("FOO", "BAR"), **{"c": "baz"})
    FOO BAR baz

关键字参数
----------------------

**Python 3.0中的新功能**

.. code-block:: python

    >>> def f(a, b, *, kw):
    ...     print(a, b, kw)
    ...
    >>> f(1, 2, kw=3)
    1 2 3
    >>> f(1, 2, 3)
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
    TypeError: f() takes 2 positional arguments but 3 were given

注解
-----------

**Python 3.0中的新功能**

注解是有用的方式可以提示程序员参数的类型。
这个特性的规范在PEP `3107 <https://www.python.org/dev/peps/pep-3107/>`_。
Python3.5介绍 ``typing`` 模块来拓展类型提示的概念。
此外, 从3.6版本开始，Python开始提供一个通用的方式去定义一个变量伴随着一个注解。 to offer a general way to define a
variable with an annotation.
更多的信息可以看PEP `483 <https://www.python.org/dev/peps/pep-0483>`_， PEP
`484 <https://www.python.org/dev/peps/pep-0484>`_，和 PEP
`526 <https://www.python.org/dev/peps/pep-0526>`_。

.. code-block:: python

    >>> def fib(n: int) -> int:
    ...     a, b = 0, 1
    ...     for _ in range(n):
    ...         b, a = a + b, b
    ...     return a
    ...
    >>> fib(10)
    55
    >>> fib.__annotations__
    {'n': <class 'int'>, 'return': <class 'int'>}

可调用
--------

在某些情况下，例如传递一个回调函数，我们需要检查对象是否可调用。
内建函数 ``callable``, 可以帮助我们避免，当一个对象是不可调用的抛出 ``TypeError``。

.. code-block:: python

    >>> a = 10
    >>> def fun():
    ...   print("I am callable")
    ...
    >>> callable(a)
    False
    >>> callable(fun)
    True

获取函数名称
-----------------

.. code-block:: python

    >>> def example_function():
    ...   pass
    ...
    >>> example_function.__name__
    'example_function'

Lambda表达式
----------------

有时，我们不想要使用 *def* 语句定义一个简短的回调函数。
我们可以使用一个 ``lambda`` 表达式，作为快捷方式来定义匿名函数和单行函数。
但是，在 ``lambda`` 中只能指定一个表达式。
也就是说，不能包含其他特性，比如多行语句、条件或异常处理。

.. code-block:: python

    >>> fn = lambda x: x**2
    >>> fn(3)
    9
    >>> (lambda x: x**2)(3)
    9
    >>> (lambda x: [x*_ for _ in range(5)])(2)
    [0, 2, 4, 6, 8]
    >>> (lambda x: x if x>3 else 3)(5)
    5

生成器
---------

.. code-block:: python

    >>> def fib(n):
    ...     a, b = 0, 1
    ...     for _ in range(n):
    ...         yield a
    ...         b, a = a + b, b
    ...
    >>> [f for f in fib(10)]
    [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

装饰器
---------

**Python 2.4中的新功能**

- PEP `318 <https://www.python.org/dev/peps/pep-0318/>`_ - 函数和方法的装饰器

.. code-block:: python

    >>> from functools import wraps
    >>> def decorator(func):
    ...     @wraps(func)
    ...     def wrapper(*args, **kwargs):
    ...         print("Before calling {}.".format(func.__name__))
    ...         ret = func(*args, **kwargs)
    ...         print("After calling {}.".format(func.__name__))
    ...         return ret
    ...     return wrapper
    ...
    >>> @decorator
    ... def example():
    ...     print("Inside example function.")
    ...
    >>> example()
    Before calling example.
    Inside example function.
    After calling example.

等同于

.. code-block:: python

    ... def example():
    ...     print("Inside example function.")
    ...
    >>> example = decorator(example)
    >>> example()
    Before calling example.
    Inside example function.
    After calling example.

有参数的装饰器
------------------------

.. code-block:: python

    >>> from functools import wraps
    >>> def decorator_with_argument(val):
    ...     def decorator(func):
    ...         @wraps(func)
    ...         def wrapper(*args, **kwargs):
    ...             print("Val is {0}".format(val))
    ...             return func(*args, **kwargs)
    ...         return wrapper
    ...     return decorator
    ...
    >>> @decorator_with_argument(10)
    ... def example():
    ...     print("This is example function.")
    ...
    >>> example()
    Val is 10
    This is example function.

等同于

.. code-block:: python

    >>> def example():
    ...     print("This is example function.")
    ...
    >>> example = decorator_with_argument(10)(example)
    >>> example()
    Val is 10
    This is example function.

缓存
-----

**Python 3.2中的新功能**

没有缓存

.. code-block:: python

    >>> import time
    >>> def fib(n):
    ...     if n < 2:
    ...         return n
    ...     return fib(n - 1) + fib(n - 2)
    ...
    >>> s = time.time(); _ = fib(32); e = time.time(); e - s
    1.1562161445617676

有缓存 (动态编程)

.. code-block:: python

    >>> from functools import lru_cache
    >>> @lru_cache(maxsize=None)
    ... def fib(n):
    ...     if n < 2:
    ...         return n
    ...     return fib(n - 1) + fib(n - 2)
    ...
    >>> s = time.time(); _ = fib(32); e = time.time(); e - s
    2.9087066650390625e-05
    >>> fib.cache_info()
    CacheInfo(hits=30, misses=33, maxsize=None, currsize=33)
