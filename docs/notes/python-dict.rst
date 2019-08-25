========
字典
========

获取所有键
------------

.. code-block:: python

    >>> a = {"1":1, "2":2, "3":3}
    >>> b = {"2":2, "3":3, "4":4}
    >>> a.keys()
    ['1', '3', '2']

获取键和值
-----------------

.. code-block:: python

    >>> a = {"1":1, "2":2, "3":3}
    >>> a.items()

发现相同的键
--------------

.. code-block:: python

    >>> a = {"1":1, "2":2, "3":3}
    >>> b = {"2":2, "3":3, "4":4}
    >>> [_ for _ in a.keys() if _ in b.keys()]
    ['3', '2']
    >>> # 更好的方式
    >>> c = set(a).intersection(set(b))
    >>> list(c)
    ['3', '2']
    >>> # 或
    >>> [_ for _ in a if _ in b]
    ['3', '2']
    [('1', 1), ('3', 3), ('2', 2)]

设置默认值
-------------

.. code-block:: python

    >>> # 直观但不推荐
    >>> d = {}
    >>> key = "foo"
    >>> if key not in d:
    ...     d[key] = []
    ...

    >>> # 使用 d.setdefault(key[, default])
    >>> d = {}
    >>> key = "foo"
    >>> d.setdefault(key, [])
    []
    >>> d[key] = 'bar'
    >>> d
    {'foo': 'bar'}

    >>> # 使用 collections.defaultdict
    >>> from collections import defaultdict
    >>> d = defaultdict(list)
    >>> d["key"]
    []
    >>> d["foo"]
    []
    >>> d["foo"].append("bar")
    >>> d
    defaultdict(<class 'list'>, {'key': [], 'foo': ['bar']})

``dict.setdefault(key[, default])`` 返回它的默认值，如果 *key* 不存在字典中。
然而，如果它的key存在字典中，这个函数将会返回它的值。

.. code-block:: python

    >>> d = {}
    >>> d.setdefault("key", [])
    []
    >>> d["key"] = "bar"
    >>> d.setdefault("key", [])
    'bar'

更新字典
-----------------

.. code-block:: python

    >>> a = {"1":1, "2":2, "3":3}
    >>> b = {"2":2, "3":3, "4":4}
    >>> a.update(b)
    >>> a
    {'1': 1, '3': 3, '2': 2, '4': 4}

合并两个字典
----------------------

Python 3.4或更低版本

.. code-block:: python

    >>> a = {"x": 55, "y": 66}
    >>> b = {"a": "foo", "b": "bar"}
    >>> c = a.copy()
    >>> c.update(b)
    >>> c
    {'y': 66, 'x': 55, 'b': 'bar', 'a': 'foo'}


Python 3.5或更低版本

.. code-block:: python

    >>> a = {"x": 55, "y": 66}
    >>> b = {"a": "foo", "b": "bar"}
    >>> c = {**a, **b}
    >>> c
    {'x': 55, 'y': 66, 'a': 'foo', 'b': 'bar'}

模拟一个字典
----------------------

.. code-block:: python

    >>> class EmuDict(object):
    ...   def __init__(self, dict_):
    ...     self._dict = dict_
    ...   def __repr__(self):
    ...     return "EmuDict: " + repr(self._dict)
    ...   def __getitem__(self, key):
    ...     return self._dict[key]
    ...   def __setitem__(self, key, val):
    ...     self._dict[key] = val
    ...   def __delitem__(self, key):
    ...     del self._dict[key]
    ...   def __contains__(self, key):
    ...     return key in self._dict
    ...   def __iter__(self):
    ...     return iter(self._dict.keys())
    ...
    >>> _ = {"1":1, "2":2, "3":3}
    >>> emud = EmuDict(_)
    >>> emud  # __repr__
    EmuDict: {'1': 1, '2': 2, '3': 3}
    >>> emud['1']  # __getitem__
    1
    >>> emud['5'] = 5  # __setitem__
    >>> emud
    EmuDict: {'1': 1, '2': 2, '3': 3, '5': 5}
    >>> del emud['2']  # __delitem__
    >>> emud
    EmuDict: {'1': 1, '3': 3, '5': 5}
    >>> for _ in emud:
    ...     print(emud[_], end=' ')  # __iter__
    ... else:
    ...     print()
    ...
    1 3 5
    >>> '1' in emud  # __contains__
    True
