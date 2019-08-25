======
Future
======


`Future 语句 <https://docs.python.org/3/reference/simple_stmts.html#future>`_
告诉解释器编译一些语义，这些语义在未来更高Python版本使用。 换句话说，Python使用 ``from __future__ import feature``
去反向移植一些更高版本的Python特性到当前的版本。
在Python3, 许多特性，比如：``print_function`` 已经被启用，但是
为了向后兼容，我们还是会保留future语句。

Future语句  **不是** 导入语句。Future语句会改变
Python解释器解释代码的方式。 因此，它们 **必须** 要放在文件的顶部。否则，
Python解释器会抛出 ``SyntaxError`` 错误。

如果你对future语句很有兴趣，想要获取更多的信息，可以在看一下
`PEP 236 - Back to the __future__  <https://www.python.org/dev/peps/pep-0236>`_ 。

.. contents:: Table of Contents
    :backlinks: none

列举所有新特性
---------------------

`__future__ <https://docs.python.org/3/library/__future__.html>`_ 是一个Python的模块。
我们可以使用它来查看试用于当前Python解释器的的future特性。
有意思的是 ``import __future__`` **不是** 一个future语句, 它是一个导入语句。

.. code-block:: python

    >>> from pprint import pprint
    >>> import __future__
    >>> pprint(__future__.all_feature_names)
    ['nested_scopes',
     'generators',
     'division',
     'absolute_import',
     'with_statement',
     'print_function',
     'unicode_literals',
     'barry_as_FLUFL',
     'generator_stop',
     'annotations']

Future语句不仅会改变Python解释器的行为，也会导入
``__future__._Feature`` 到当前的代码。

.. code-block:: python

    >>> from __future__ import print_function
    >>> print_function
    _Feature((2, 6, 0, 'alpha', 2), (3, 0, 0, 'alpha', 0), 65536)

Print函数
--------------

把 **print语句** 替换成  **print函数** 是Python历史上最臭名昭着的决定之一。
然而, 这种改变使拓展 ``print`` 的能力更加的灵活。
更多的信息可以看PEP `3105 <https://www.python.org/dev/peps/pep-3105>`_。

.. code-block:: python

    >>> print "Hello World"  # print is a statement
    Hello World
    >>> from __future__ import print_function
    >>> print "Hello World"
      File "<stdin>", line 1
        print "Hello World"
                          ^
    SyntaxError: invalid syntax
    >>> print("Hello World") # print become a function
    Hello World

Unicode
-------

正如 **print function**, 使文本变成unicode是另一个臭名昭着的决定。
虽然, 许多现代的编程语言文本都是unicode的。
这个改变使我们早早的decode文本，为了避免在我们运行代码一段时间后发生的运行时错误。
更多的信息可以看PEP `3112 <https://www.python.org/dev/peps/pep-3112>`_。

.. code-block:: python

    >>> type("Guido") # string type is str in python2
    <type 'str'>
    >>> from __future__ import unicode_literals
    >>> type("Guido") # string type become unicode
    <type 'unicode'>

除法
--------
有时，它是有悖常理的，当除法的结果是一个int或者是long。
在这种情况下，Python3默认启用了 **真正的除法** 。
然而，在Python2中，我们不得不反向移植 ``division`` 到当前的解释器。
更多的信息可以看PEP `238 <https://www.python.org/dev/peps/pep-0238>`_。

.. code-block:: python

    >>> 1 / 2
    0
    >>> from __future__ import division
    >>> 1 / 2   # return a float (classic division)
    0.5
    >>> 1 // 2  # return a int (floor division)
    0

注解
-----------

在Python 3.7之前, 如果它在当前作用域不可用的话，我们无法在类或者方法中分配注释。
通用的解决方法是定义一个容器类。

.. code-block:: python

    class Tree(object):

        def insert(self, tree: Tree): ...

举个🌰：

.. code-block:: bash

    $ python3 foo.py
    Traceback (most recent call last):
      File "foo.py", line 1, in <module>
        class Tree(object):
      File "foo.py", line 3, in Tree
        def insert(self, tree: Tree): ...
    NameError: name 'Tree' is not defined

在这种情况下, 这个类的定义是不可信的。
Python解释器在定义时不能解析注释。
为了解决这个问题，Python使用字符串来代替类。

.. code-block:: python

    class Tree(object):

        def insert(self, tree: 'Tree'): ...

在version 3.7及其之后， Python引入了future语句 ``annotations`` 推迟执行。
它在Python4会变成默认特性。
更多的信息可以看PEP `563 <https://www.python.org/dev/peps/pep-0563>`_。


.. code-block:: python

    from __future__ import annotations

    class Tree(object):

        def insert(self, tree: Tree): ...

终身仁慈独裁者退休
----------------------

**在Python 3.1新加入的**

PEP `401 <https://www.python.org/dev/peps/pep-0401/>`_ 是一个复活节彩蛋。
这个特性会把当前的解释器变成老得版本。 它可以让菱行操作符 ``<>`` 可以在Python中使用。

.. code-block:: python

    >>> 1 != 2
    True
    >>> from __future__ import barry_as_FLUFL
    >>> 1 != 2
      File "<stdin>", line 1
        1 != 2
           ^
    SyntaxError: with Barry as BDFL, use '<>' instead of '!='
    >>> 1 <> 2
    True

Braces
------

``braces`` 是一个彩蛋。源码可以在这里
`future.c <https://github.com/python/cpython/blob/master/Python/future.c>`_ 看到。

.. code-block:: python

    >>> from __future__ import braces
      File "<stdin>", line 1
    SyntaxError: not a chance
