.. meta::
    :description lang=en: Collect useful snippets of I/O operations.
    :keywords: Python, Python I/O Operations

=============
文件和I/O
=============

.. contents:: Table of Contents
    :backlinks: none

读一个文件
--------------

在Python2中，从文件系统中读取文件的内容，不会被解码。
也就是说，文件的内容是字节字符串，而不是Unicode字符串。

.. code-block:: python

    >>> with open("/etc/passwd") as f:
    ...    content = f.read()
    >>> print(type(content))
    <type 'str'>
    >>> print(type(content.decode("utf-8")))
    <type 'unicode'>

在Python3中，`open <https://docs.python.org/3/library/functions.html#open>`_
提供了一个 ``encoding`` 选项。如果文件不是有二进制模式打开的话，
编码将由 ``locale.getpreferredencoding(False)`` 或者用户输入决定。

.. code-block:: python

    >>> with open("/etc/hosts", encoding="utf-8") as f:
    ...     content = f.read()
    ...
    >>> print(type(content))
    <class 'str'>

二进制模式

.. code-block:: python

    >>> with open("/etc/hosts", "rb") as f:
    ...     content = f.read()
    ...
    >>> print(type(content))
    <class 'bytes'>

读行
--------

.. code-block:: python

    >>> with open("/etc/hosts") as f:
    ...     for line in f:
    ...         print(line, end='')
    ...
    127.0.0.1       localhost
    255.255.255.255	broadcasthost
    ::1             localhost

读文件块
-------------------

.. code-block:: python


    >>> chunk_size = 16
    >>> content = ''
    >>> with open('/etc/hosts') as f:
    ...     for c in iter(lambda: f.read(chunk_size), ''):
    ...         content += c
    ...
    >>> print(content)
    127.0.0.1       localhost
    255.255.255.255 broadcasthost
    ::1             localhost

写文件
---------------

.. code-block:: python

    >>> content = "Awesome Python!"
    >>> with open("foo.txt", "w") as f:
    ...     f.write(content)

创建符号链接
----------------------

.. code-block:: python

    >>> import os
    >>> os.symlink("foo", "bar")
    >>> os.readlink("bar")
    'foo'

拷贝文件
---------------

.. code-block:: python

    >>> from distutils.file_util import copy_file
    >>> copy_file("foo", "bar")
    ('bar', 1)

移动文件
---------------

.. code-block:: python

    >>> from distutils.file_util import move_file
    >>> move_file("./foo", "./bar")
    './bar'

列出目录
----------------

.. code-block:: python

    >>> >>> import os
    >>> dirs = os.listdir(".")

在Python3.6之后，我们可以使用 ``os.scandir`` 去列出目录。它是更加方便，因为 ``os.scandir``
返回一个 ``os.DirEntry`` 对象的迭代器。
在这个情况下，我们可以通过访问 ``os.DirEntry`` 的属性，获取文件信息。
更多信息请看 `document <https://docs.python.org/3/library/os.html#os.scandir>`_.

.. code-block:: python

    >>> with os.scandir("foo") as it:
    ...     for entry in it:
    ...         st = entry.stat()
    ...

创建目录
------------------

类似于 ``mkdir -p /path/to/dest``

.. code-block:: python

    >>> from distutils.dir_util import mkpath
    >>> mkpath("foo/bar/baz")
    ['foo', 'foo/bar', 'foo/bar/baz']

拷贝目录
----------------

.. code-block:: python

    >>> from distutils.dir_util import copy_tree
    >>> copy_tree("foo", "bar")
    ['bar/baz']

删除目录
------------------

.. code-block:: python

    >>> from distutils.dir_util import remove_tree
    >>> remove_tree("dir")

路径加入
-------------

.. code-block:: python

    >>> from pathlib import Path
    >>> p = Path("/Users")
    >>> p = p / "Guido" / "pysheeet"
    >>> p
    PosixPath('/Users/Guido/pysheeet')

获取绝对路径
-----------------

.. code-block:: python

    >>> from pathlib import Path
    >>> p = Path("README.rst")
    PosixPath('/Users/Guido/pysheeet/README.rst')

获取家目录
------------------

.. code-block:: python

    >>> from pathlib import Path
    >>> Path.home()
    PosixPath('/Users/Guido')

获取当前目录
---------------------

.. code-block:: python

    >>> from pathlib import Path
    >>> p = Path("README.rst")
    >>> p.cwd()
    PosixPath('/Users/Guido/pysheeet')

获取路径属性
-------------------

.. code-block:: python

    >>> from pathlib import Path
    >>> p = Path("README.rst").absolute()
    >>> p.root
    '/'
    >>> p.anchor
    '/'
    >>> p.parent
    PosixPath('/Users/Guido/pysheeet')
    >>> p.parent.parent
    PosixPath('/Users/Guido')
    >>> p.name
    'README.rst'
    >>> p.suffix
    '.rst'
    >>> p.stem
    'README'
    >>> p.as_uri()
    'file:///Users/Guido/pysheeet/README.rst'
