====
列表
====

列表是我们用来存储对象的常用数据结构。大多数时候，
程序员关心获取、设置、搜索、过滤、排序。
此外，有时候，我们会把自己跳入内存管理的常见陷阱。
因此，这章备忘录的主要的目标是收集一些常用的操作和陷阱。

.. contents:: Table of Contents
    :backlinks: none

从头开始
------------

在Python中，我们有很多方法操作列表。
在我们开始学习这些多功能操作之前，下面的代码片段显示了列表中最常见的操作。

.. code-block:: python

    >>> a = [1, 2, 3, 4, 5]
    >>> # 包含
    >>> 2 in a
    True
    >>> # 正索引
    >>> a[0]
    1
    >>> # 负索引
    >>> a[-1]
    5
    >>> # 切片 list[start:end:step]
    >>> a[1:]
    [2, 3, 4, 5]
    >>> a[1:-1]
    [2, 3, 4]
    >>> a[1:-1:2]
    [2, 4]
    >>> # 反序
    >>> a[::-1]
    [5, 4, 3, 2, 1]
    >>> a[:0:-1]
    [5, 4, 3, 2]
    >>> # 赋值
    >>> a[0] = 0
    >>> a
    [0, 2, 3, 4, 5]
    >>> # 添加元素
    >>> a.append(6)
    >>> a
    [0, 2, 3, 4, 5, 6]
    >>> a.extend([7, 8, 9])
    >>> a
    [0, 2, 3, 4, 5, 6, 7, 8, 9]
    >>> # 删除一个元素
    >>> del a[-1]
    >>> a
    [0, 2, 3, 4, 5, 6, 7, 8]
    >>> # 列表推导式
    >>> b = [x for x in range(3)]
    >>> b
    [0, 1, 2]
    >>> # 两个列表添加
    >>> a + b
    [0, 2, 3, 4, 5, 6, 7, 8, 0, 1, 2]

初始化
----------

通常来说，如果一个列表中的元素是一个不可变对象，我们可以创建列表通过 ``*`` 操作符。

.. code-block:: python

    >>> a = [None] * 3
    >>> a
    [None, None, None]
    >>> a[0] = "foo"
    >>> a
    ['foo', None, None]

然而，如果列表表达式中的元素是可变对象时，使用 ``*`` 操作符将会拷贝元素的索引N次。
为了避免这个陷阱，我们应该使用列表推导式初始化一个列表。

.. code-block:: python

    >>> a = [[]] * 3
    >>> b = [[] for _ in range(3)]
    >>> a[0].append("Hello")
    >>> a
    [['Hello'], ['Hello'], ['Hello']]
    >>> b[0].append("Python")
    >>> b
    [['Python'], [], []]

拷贝
----

把一个列表赋值给一个变量是一个常见的陷阱。这个赋值不会拷贝一个列表到这个变量。
这个变量只是引用这个列表和增加了列表的引用计数。

.. code-block:: python

    import sys
    >>> a = [1, 2, 3]
    >>> sys.getrefcount(a)
    2
    >>> b = a
    >>> sys.getrefcount(a)
    3
    >>> b[2] = 123456  # a[2] = 123456
    >>> b
    [1, 2, 123456]
    >>> a
    [1, 2, 123456]

这里有两种类型的拷贝。第一种叫 *浅拷贝* (不会递归拷贝)，第二种叫 *深拷贝* (递归拷贝)。
大多数情况下，我们通过浅拷贝复制列表就足够了。但是，如果列表是嵌套的话，我们必须使用深拷贝。

.. code-block:: python

    >>> # shallow copy
    >>> a = [1, 2]
    >>> b = list(a)
    >>> b[0] = 123
    >>> a
    [1, 2]
    >>> b
    [123, 2]
    >>> a = [[1], [2]]
    >>> b = list(a)
    >>> b[0][0] = 123
    >>> a
    [[123], [2]]
    >>> b
    [[123], [2]]
    >>> # deep copy
    >>> import copy
    >>> a = [[1], [2]]
    >>> b = copy.deepcopy(a)
    >>> b[0][0] = 123
    >>> a
    [[1], [2]]
    >>> b
    [[123], [2]]

使用 ``slice``
---------------

有时，我们的数据也许是一大串连续的，比如数据包。
在这个情况下，我们将使用 ``slice`` 对象来解释变量用来表示数据的范围，而不是使用 *slicing表达式*。

.. code-block:: python

    >>> icmp = (
    ...     b"080062988e2100005bff49c20005767c"
    ...     b"08090a0b0c0d0e0f1011121314151617"
    ...     b"18191a1b1c1d1e1f2021222324252627"
    ...     b"28292a2b2c2d2e2f3031323334353637"
    ... )
    >>> head = slice(0, 32)
    >>> data = slice(32, len(icmp))
    >>> icmp[head]
    b'080062988e2100005bff49c20005767c'

List推导式
-------------------

`List推导式 <https://docs.python.org/3/tutorial/datastructures.html#list-comprehensions>`_
被提出在PEP `202 <https://www.python.org/dev/peps/pep-0202/>`_。
它提供了一个优雅的方式基于另一个列表、序列或者可迭代对象创建一个新的列表。
此外，有时候，我们可以用这个表达式中式替换 ``map`` 和 ``filter`` 。

.. code-block:: python

    >>> [x for x in range(10)]
    [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
    >>> [(lambda x: x**2)(i) for i in range(10)]
    [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]
    >>> [x for x in range(10) if x > 5]
    [6, 7, 8, 9]
    >>> [x if x > 5 else 0 for x in range(10)]
    [0, 0, 0, 0, 0, 0, 6, 7, 8, 9]
    >>> [x + 1 if x < 5 else x + 2 if x > 5 else x + 5 for x in range(10)]
    [1, 2, 3, 4, 5, 10, 8, 9, 10, 11]
    >>> [(x, y) for x in range(3) for y in range(2)]
    [(0, 0), (0, 1), (1, 0), (1, 1), (2, 0), (2, 1)]

解包
---------

有时，我们想要解包我们的列表到变量，为了我们的代码更有可读性。
在这个情况下，我们分配N个元素到N个变量，如下面的🌰展示的一样。

.. code-block:: python

    >>> arr = [1, 2, 3]
    >>> a, b, c = arr
    >>> a, b, c
    (1, 2, 3)

基于PEP `3132 <https://www.python.org/dev/peps/pep-3132>`_，在Python3中，我们可以使用单个星号来解包N个元素，
分配的变量数可以小于元素N。

.. code-block:: python

    >>> arr = [1, 2, 3, 4, 5]
    >>> a, b, *c, d = arr
    >>> a, b, d
    (1, 2, 5)
    >>> c
    [3, 4]

使用 ``enumerate``
-------------------

``enumerate`` 是一个内建的方法。它帮助我们同时获取索引(或者一个数量)和元素，不需要使用 ``range(len(list))``。
更多信息可以看 `Looping Techniques <https://docs.python.org/3/tutorial/datastructures.html#looping-techniques>`_。

.. code-block:: python

    >>> for i, v in enumerate(range(3)):
    ...     print(i, v)
    ...
    0 0
    1 1
    2 2
    >>> for i, v in enumerate(range(3), 1): # start = 1
    ...     print(i, v)
    ...
    1 0
    2 1
    3 2

合并列表
---------

`zip <https://docs.python.org/3/library/functions.html#zip>`_ 使我们一次能够迭代多个列表中的包含的元素。
当其中之一的列表元素被用完，迭代结束。作为结果，迭代的长度和最短的列表长度相同。
如果不希望出现这个行为，我们在 **Python3** 中可以使用 ``itertools.zip_longest``，或者在 **Python2** 中可以使用 ``itertools.izip_longest``。

.. code-block:: python

    >>> a = [1, 2, 3]
    >>> b = [4, 5, 6]
    >>> list(zip(a, b))
    [(1, 4), (2, 5), (3, 6)]
    >>> c = [1]
    >>> list(zip(a, b, c))
    [(1, 4, 1)]
    >>> from itertools import zip_longest
    >>> list(zip_longest(a, b, c))
    [(1, 4, 1), (2, 5, None), (3, 6, None)]


过滤元素
------------

`filter <https://docs.python.org/3/library/functions.html#filter>`_ 是一个内建的函数用来帮助我们移除不需要的元素。
在 **Python 2** 中，``filter`` 返回一个列表。 然而，在 **Python 3** 中， ``filter`` 返回一个
*可迭代对象*。请注意 *列表推导式* 或者 *生成器表达式* 提供一种更简洁的方法来删除元素。

.. code-block:: python

    >>> [x for x in range(5) if x > 1]
    [2, 3, 4]
    >>> l = ['1', '2', 3, 'Hello', 4]
    >>> f = lambda x: isinstance(x, int)
    >>> filter(f, l)
    <filter object at 0x10bee2198>
    >>> list(filter(f, l))
    [3, 4]
    >>> list((i for i in l if f(i)))
    [3, 4]

栈
------

这里不需要一个额外的数据结构，栈，因为在Python中，``list`` 提供了 ``append`` 和 ``pop`` 方法，
使我们能够将列表用作栈。

.. code-block:: python

    >>> stack = []
    >>> stack.append(1)
    >>> stack.append(2)
    >>> stack.append(3)
    >>> stack
    [1, 2, 3]
    >>> stack.pop()
    3
    >>> stack.pop()
    2
    >>> stack
    [1]

``in`` 操作
----------------

我们可以实现 ``__contains__`` 方法，使一个类可以使用 ``in`` 操作。
这是一个常用的方式对程序员来说，实现一个自定义类进行模拟会员测试操作。

.. code-block:: python

    class Stack:

        def __init__(self):
            self.__list = []

        def push(self, val):
            self.__list.append(val)

        def pop(self):
            return self.__list.pop()

        def __contains__(self, item):
            return True if item in self.__list else False

    stack = Stack()
    stack.push(1)
    print(1 in stack)
    print(0 in stack)

Example

.. code-block:: bash

    python stack.py
    True
    False

访问元素
---------------

使自定义类执行get和set操作就像列表一样简单。
我们可以实现 ``__getitem__`` 和 ``__setitem__`` 方法，使一个类可以通过索引获取和重写数据。
此外，如果我们想要这个函数 ``len``，去计算元素的数量，我们可以实现一个 ``__len__`` 方法。

.. code-block:: python

    class Stack:

        def __init__(self):
            self.__list = []

        def push(self, val):
            self.__list.append(val)

        def pop(self):
            return self.__list.pop()

        def __repr__(self):
            return "{}".format(self.__list)

        def __len__(self):
            return len(self.__list)

        def __getitem__(self, idx):
            return self.__list[idx]

        def __setitem__(self, idx, val):
            self.__list[idx] = val


    stack = Stack()
    stack.push(1)
    stack.push(2)
    print("stack:", stack)

    stack[0] = 3
    print("stack:", stack)
    print("num items:", len(stack))

Example

.. code-block:: bash

    $ python stack.py
    stack: [1, 2]
    stack: [3, 2]
    num items: 2

委托迭代
---------------------

如果一个自定义容器类包含一个列表，我们想要迭代在容器上工作，
我们可以实现 ``__iter__`` 方法将迭代委托给列表。
注意这个方法，``__iter__``，应该返回一个 *可迭代对象*，
因此我们不能直接返回这个列表，否则，Python会抛出 ``TypeError``。

.. code-block:: python

    class Stack:

        def __init__(self):
            self.__list = []

        def push(self, val):
            self.__list.append(val)

        def pop(self):
            return self.__list.pop()

        def __iter__(self):
            return iter(self.__list)

    stack = Stack()
    stack.push(1)
    stack.push(2)
    for s in stack:
        print(s)

Example

.. code-block:: bash

    $ python stack.py
    1
    2

排序
-------

Python的列表提供一个内建的方法 ``list.sort``，它在不使用额外内存的情况下 `就地 <https://en.wikipedia.org/wiki/In-place_algorithm>`_ 对列表进行排序。
此外，``list.sort`` 的返回值式 ``None``，以避免与 ``sorted`` 混淆，这个函数只能用于列表。

.. code-block:: python

    >>> l = [5, 4, 3, 2, 1]
    >>> l.sort()
    >>> l
    [1, 2, 3, 4, 5]
    >>> l.sort(reverse=True)
    >>> l
    [5, 4, 3, 2, 1]

这个 ``sorted`` 函数不会就地修改任何可迭代的对象。
相反，它返回一个新的排序的列表。
使用 ``sorted`` 比 ``list.sort`` 更安全，如果列表的元素是只读的或者是不可变类型。
除此之外，``list.sort`` 和 ``sorted`` 的另一个区别是，``sorted`` 可以用在任何 **可迭代对象** 上。

.. code-block:: python

    >>> l = [5, 4, 3, 2, 1]
    >>> new = sorted(l)
    >>> new
    [1, 2, 3, 4, 5]
    >>> l
    [5, 4, 3, 2, 1]
    >>> d = {3: 'andy', 2: 'david', 1: 'amy'}
    >>> sorted(d)  # sort iterable
    [1, 2, 3]

为了排序列表中的元素是元组，使用 ``operator.itemgetter`` 是有帮助的，因为它可以作为 ``sorted`` 函数的key参数。
注意，这个key应该有可比性，否则，它会抛出一个 ``TypeError``。

.. code-block:: python

    >>> from operator import itemgetter
    >>> l = [('andy', 10), ('david', 8), ('amy', 3)]
    >>> l.sort(key=itemgetter(1))
    >>> l
    [('amy', 3), ('david', 8), ('andy', 10)]

``operator.itemgetter`` 是有用的，因为这个函数返回一个getter方法，它可以通过 ``__getitem__`` 应用于其他对象。
举个🌰，排序一个列表而且它的元素都是字典，可以通过使用 ``operator.itemgetter`` 实现，由于它所有的元素都支持  ``__getitem__``。

.. code-block:: python

    >>> from pprint import pprint
    >>> from operator import itemgetter
    >>> l = [
    ...     {'name': 'andy', 'age': 10},
    ...     {'name': 'david', 'age': 8},
    ...     {'name': 'amy', 'age': 3},
    ... ]
    >>> l.sort(key=itemgetter('age'))
    >>> pprint(l)
    [{'age': 3, 'name': 'amy'},
     {'age': 8, 'name': 'david'},
     {'age': 10, 'name': 'andy'}]

如果必须要对一个列表元素即没有可比性也没有 ``__getitem__`` 方法进行排序，指定一个自定义的key函数是可能的。

.. code-block:: python

    >>> class Node(object):
    ...     def __init__(self, val):
    ...         self.val = val
    ...     def __repr__(self):
    ...         return f"Node({self.val})"
    ...
    >>> nodes = [Node(3), Node(2), Node(1)]
    >>> nodes.sort(key=lambda x: x.val)
    >>> nodes
    [Node(1), Node(2), Node(3)]
    >>> nodes.sort(key=lambda x: x.val, reverse=True)
    >>> nodes
    [Node(3), Node(2), Node(1)]

以上的代码片段可以简单的使用 ``operator.attrgetter`` 实现。
这个函数返回一个获取属性的方法，基于属性名。
注意，这个属性应该是有可比性的，否则 ``sorted`` 或者 ``list.sort`` 将会抛出 ``TypeError``。

.. code-block:: python

    >>> from operator import attrgetter
    >>> class Node(object):
    ...     def __init__(self, val):
    ...         self.val = val
    ...     def __repr__(self):
    ...         return f"Node({self.val})"
    ...
    >>> nodes = [Node(3), Node(2), Node(1)]
    >>> nodes.sort(key=attrgetter('val'))
    >>> nodes
    [Node(1), Node(2), Node(3)]

如果一个对象有 ``__lt__`` 方法，它意味着这个对象是可比的，
``sorted`` 或者 ``list.sort`` 就不必要传入key函数了到它的key参数了。
一个列表或者一个可迭代对象可以直接排序了。

.. code-block:: python

    >>> class Node(object):
    ...     def __init__(self, val):
    ...         self.val = val
    ...     def __repr__(self):
    ...         return f"Node({self.val})"
    ...     def __lt__(self, other):
    ...         return self.val - other.val < 0
    ...
    >>> nodes = [Node(3), Node(2), Node(1)]
    >>> nodes.sort()
    >>> nodes
    [Node(1), Node(2), Node(3)]

如果一个对象没有 ``__lt__`` 方法，可以在声明对象的类之后，修补这个方法。
换句话说，在这个布丁之后，这个对象就是可比的了。

.. code-block:: python

    >>> class Node(object):
    ...     def __init__(self, val):
    ...         self.val = val
    ...     def __repr__(self):
    ...         return f"Node({self.val})"
    ...
    >>> Node.__lt__ = lambda s, o: s.val < o.val
    >>> nodes = [Node(3), Node(2), Node(1)]
    >>> nodes.sort()
    >>> nodes
    [Node(1), Node(2), Node(3)]

注意，在Python3中，``sorted`` 或者 ``list.sort`` 不支持 ``cmp`` 参数，这是在Python2 **唯一** 有效的参数。
如果非要使用一个老的比较函数，例如一些遗留代码，``functools.cmp_to_key`` 是有用的，因为它将cmp函数转换为key函数。

.. code-block:: python

    >>> from functools import cmp_to_key
    >>> class Node(object):
    ...     def __init__(self, val):
    ...         self.val = val
    ...     def __repr__(self):
    ...         return f"Node({self.val})"
    ...
    >>> nodes = [Node(3), Node(2), Node(1)]
    >>> nodes.sort(key=cmp_to_key(lambda x,y: x.val - y.val))
    >>> nodes
    [Node(1), Node(2), Node(3)]
