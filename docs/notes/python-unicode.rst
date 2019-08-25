.. meta::
    :description lang=en: Collect useful snippets of unicode
    :keywords: Python, Python3, Python Unicode, Python Unicode Cheat Sheet

=======
Unicode
=======

这个备忘录的目的是收集一些关于Unicode常用的片段。
在Python3中，字符串有Unicode表示，而不是bytes了。
更多的信息可以看PEP `3100 <https://www.python.org/dev/peps/pep-3100>`_。

**ASCII** 码是最有名的标准，把数字符号定义为字符。
最初定了128个字符，所以ASCII只包含了控制字符、数字、小写字母、大写字母等。
然而，这对我们来说，用来表示存在于世的字符是不够的，比如重音字符、汉字、表情等。
因此，**Unicode** 被开发出来，解决这个问题。
它定义 *代码点* 来表示各种字符，如ASCII，但字符数最多为1,111,998。它定义代码点去表示不同的字符，如ASCII，
但是字符的数量高达1111998。

.. contents:: Table of Contents
    :backlinks: none

字符串
------

在Python2中, 字符串用 *bytes* 来表示，而不是 *Unicode*。
Python提供了不同类型的字符串，例如Unicode字符串，原始字符串等。
在这种情况下，如果我们想要定义一个Unicode字符串，我们需要在字符串的前面加上 ``u`` 。

.. code-block:: python

    >>> s = 'Café'  # byte string
    >>> s
    'Caf\xc3\xa9'
    >>> type(s)
    <type 'str'>
    >>> u = u'Café' # unicode string
    >>> u
    u'Caf\xe9'
    >>> type(u)
    <type 'unicode'>

在Python3中, 字符串用 *Unicode* 来表示。如果我们想要定义一个byte字符串，我们需要在字符串前面加上 ``b`` 前缀。
注意在Python3.0-3.2版本不支持添加 ``u`` 前缀。换句话说，
为了减轻从Python2的应用迁移Unicode字符串，所以从Python3.3开始支持了 ``u`` 前缀。
更多的信息可以看PEP `414 <https://www.python.org/dev/peps/pep-0414>`_。

.. code-block:: python

    >>> s = 'Café'
    >>> type(s)
    <class 'str'>
    >>> s
    'Café'
    >>> s.encode('utf-8')
    b'Caf\xc3\xa9'
    >>> s.encode('utf-8').decode('utf-8')
    'Café'

Characters
----------

Python2所有字符串的字符都是bytes。 在这种情况下，字符串的长度可能不等于字符的数量。
举个🌰，``Café`` 的长度是5, 不是4，因为 ``é`` 被编码成2个bytes字符。

.. code-block:: python

    >>> s= 'Café'
    >>> print([_c for _c in s])
    ['C', 'a', 'f', '\xc3', '\xa9']
    >>> len(s)
    5
    >>> s = u'Café'
    >>> print([_c for _c in s])
    [u'C', u'a', u'f', u'\xe9']
    >>> len(s)
    4

Python3所有的字符串的字符都是Unicode的代码点。
字符串的长度总是等于字符的数量。

.. code-block:: python

    >>> s = 'Café'
    >>> print([_c for _c in s])
    ['C', 'a', 'f', 'é']
    >>> len(s)
    4
    >>> bs = bytes(s, encoding='utf-8')
    >>> print(bs)
    b'Caf\xc3\xa9'
    >>> len(bs)
    5

移植unicode(s, 'utf-8')
---------------------------

在Python3中，`unicode() <https://docs.python.org/2.7/library/functions.html#unicode>`_的内建方法被移除，
最好的的方法转换表达式 ``unicode(s, 'utf-8')`` 是什么，它能在Python2、3上兼容吗?

在Python2中:

.. code-block:: python

    >>> s = 'Café'
    >>> unicode(s, 'utf-8')
    u'Caf\xe9'
    >>> s.decode('utf-8')
    u'Caf\xe9'
    >>> unicode(s, 'utf-8') == s.decode('utf-8')
    True

在Python3中:

.. code-block:: python

    >>> s = 'Café'
    >>> s.decode('utf-8')
    AttributeError: 'str' object has no attribute 'decode'

所以，真正的答案是......

Unicode代码点
------------------

`ord <https://docs.python.org/3/library/functions.html#ord>`_ 是一个强大的内建方法，
从给定的字符获取Unicode代码点。
因此，如果我们想要检查一个字符的Unicode的代码点，我们可以使用 ``ord``。

.. code-block:: python

    >>> s = u'Café'
    >>> for _c in s: print('U+%04x' % ord(_c))
    ...
    U+0043
    U+0061
    U+0066
    U+00e9
    >>> u = '中文'
    >>> for _c in u: print('U+%04x' % ord(_c))
    ...
    U+4e2d
    U+6587


编码
--------

一个 *Unicode代码点* 转化成一个 *字节字符* 被称为编码。

.. code-block:: python

    >>> s = u'Café'
    >>> type(s.encode('utf-8'))
    <class 'bytes'>

解码
---------

一个 *字节字符* 转化成一个 *Unicode代码点*  被称为解码。

.. code-block:: python

    >>> s = bytes('Café', encoding='utf-8')
    >>> s.decode('utf-8')
    'Café'

Unicode规范化
---------------------

某些字符可以用两种相似的形式来表示。
举个🌰，字符 ``é`` 可以被写成 ``e ́`` (规范分解) 或者 ``é``
(典型组合)。在这种情况下，我们也许会获得不期待的结果，当我们在比较这两个字符串时，即使他们看起来很像。
因此，我们可以规划Unicode的形式，来解决这个问题。

.. code-block:: python

    # python 3
    >>> u1 = 'Café'       # unicode string
    >>> u2 = 'Cafe\u0301'
    >>> u1, u2
    ('Café', 'Café')
    >>> len(u1), len(u2)
    (4, 5)
    >>> u1 == u2
    False
    >>> u1.encode('utf-8') # get u1 byte string
    b'Caf\xc3\xa9'
    >>> u2.encode('utf-8') # get u2 byte string
    b'Cafe\xcc\x81'
    >>> from unicodedata import normalize
    >>> s1 = normalize('NFC', u1)  # get u1 NFC format
    >>> s2 = normalize('NFC', u2)  # get u2 NFC format
    >>> s1 == s2
    True
    >>> s1.encode('utf-8'), s2.encode('utf-8')
    (b'Caf\xc3\xa9', b'Caf\xc3\xa9')
    >>> s1 = normalize('NFD', u1)  # get u1 NFD format
    >>> s2 = normalize('NFD', u2)  # get u2 NFD format
    >>> s1, s2
    ('Café', 'Café')
    >>> s1 == s2
    True
    >>> s1.encode('utf-8'), s2.encode('utf-8')
    (b'Cafe\xcc\x81', b'Cafe\xcc\x81')


避免 ``UnicodeDecodeError``
---------------------------------

Python会抛出 `UnicodeDecodeError` 当字节字符串不能被解码成Unicode代码点。
如果我们想要避免这个异常，我们可以在 `decode <https://docs.python.org/3/library/stdtypes.html#bytes.decode>`_ 时，
传递errors参数为 *replace* 、*backslashreplace*、*ignore*。

.. code-block:: python

    >>> u = b"\xff"
    >>> u.decode('utf-8', 'strict')
        Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
    UnicodeDecodeError: 'utf-8' codec can't decode byte 0xff in position 0: invalid start byte
    >>> # use U+FFFD, REPLACEMENT CHARACTER
    >>> u.decode('utf-8', "replace")
    '\ufffd'
    >>> # inserts a \xNN escape sequence
    >>> u.decode('utf-8', "backslashreplace")
    '\\xff'
    >>> # 在结果中去除这个字符
    >>> u.decode('utf-8', "ignore")
    ''

长字符串
-----------

下面的代码片段展示了在Python中几种常见的定义多行字符串的方法。

.. code-block:: python

    # 原始长字符串
    s = 'This is a very very very long python string'

    # 单引号和一个反斜杠
    s = "This is a very very very " \
        "long python string"

    # 使用括号
    s = (
        "This is a very very very "
        "long python string"
    )

    # 使用 ``+``
    s = (
        "This is a very very very " +
        "long python string"
    )

    # 使用三引号和一个反斜杠
    s = '''This is a very very very \
    long python string'''
