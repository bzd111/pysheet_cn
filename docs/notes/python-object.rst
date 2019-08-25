===================
类和对象
===================

列举属性
---------------

.. code-block:: python

    >>> dir(list)  # 检查列表的所有属性
    ['__add__', '__class__', ...]

获取实例类型
-----------------

.. code-block:: python

    >>> ex = 10
    >>> isinstance(ex, int)
    True

定义一个类
---------------

.. code-block:: python

    >>> def fib(self, n):
    ...     if n <= 2:
    ...         return 1
    ...     return fib(self, n-1) + fib(self, n-2)
    ...
    >>> Fib = type('Fib', (object,), {'val': 10,
    ...                               'fib': fib})
    >>> f = Fib()
    >>> f.val
    10
    >>> f.fib(f.val)
    55

等同于

.. code-block:: python

    >>> class Fib(object):
    ...     val = 10
    ...     def fib(self, n):
    ...         if n <=2:
    ...             return 1
    ...         return self.fib(n-1)+self.fib(n-2)
    ...
    >>> f = Fib()
    >>> f.val
    10
    >>> f.fib(f.val)
    55

具有 / 获取 / 设置属性
--------------------------

.. code-block:: python

    >>> class Example(object):
    ...   def __init__(self):
    ...     self.name = "ex"
    ...   def printex(self):
    ...     print("This is an example")
    ...
    >>> ex = Example()
    >>> hasattr(ex,"name")
    True
    >>> hasattr(ex,"printex")
    True
    >>> hasattr(ex,"print")
    False
    >>> getattr(ex,'name')
    'ex'
    >>> setattr(ex,'name','example')
    >>> ex.name
    'example'

检查继承
-----------------

.. code-block:: python

    >>> class Example(object):
    ...   def __init__(self):
    ...     self.name = "ex"
    ...   def printex(self):
    ...     print("This is an Example")
    ...
    >>> issubclass(Example, object)
    True

获取类名
--------------

.. code-block:: python

    >>> class ExampleClass(object):
    ...   pass
    ...
    >>> ex = ExampleClass()
    >>> ex.__class__.__name__
    'ExampleClass'

创建和初始化
------------

``__init__`` 将会被调用

.. code-block:: python

    >>> class ClassA(object):
    ...     def __new__(cls, arg):
    ...         print('__new__ ' + arg)
    ...         return object.__new__(cls, arg)
    ...     def __init__(self, arg):
    ...         print('__init__ ' + arg)
    ...
    >>> o = ClassA("Hello")
    __new__ Hello
    __init__ Hello

``__init__`` 不会被调用

.. code-block:: python

    >>> class ClassB(object):
    ...     def __new__(cls, arg):
    ...         print('__new__ ' + arg)
    ...         return object
    ...     def __init__(self, arg):
    ...         print('__init__ ' + arg)
    ...
    >>> o = ClassB("Hello")
    __new__ Hello


菱形继承问题
-------------------

这是多继承时，寻找一个方法顺序的问题。

.. code-block:: python

    >>> def foo_a(self):
    ...     print("This is ClsA")
    ...
    >>> def foo_b(self):
    ...     print("This is ClsB")
    ...
    >>> def foo_c(self):
    ...     print("This is ClsC")
    ...
    >>> class Type(type):
    ...     def __repr__(cls):
    ...         return cls.__name__
    ...
    >>> ClsA = Type("ClsA", (object,), {'foo': foo_a})
    >>> ClsB = Type("ClsB", (ClsA,), {'foo': foo_b})
    >>> ClsC = Type("ClsC", (ClsA,), {'foo': foo_c})
    >>> ClsD = Type("ClsD", (ClsB, ClsC), {})
    >>> ClsD.mro()
    [ClsD, ClsB, ClsC, ClsA, <type 'object'>]
    >>> ClsD().foo()
    This is ClsB

表示一个类
-------------------------

.. code-block:: python

    >>> class Example(object):
    ...    def __str__(self):
    ...       return "Example __str__"
    ...    def __repr__(self):
    ...       return "Example __repr__"
    ...
    >>> print(str(Example()))
    Example __str__
    >>> Example()
    Example __repr__

可调用对象
---------------

.. code-block:: python

    >>> class CallableObject(object):
    ...   def example(self, *args, **kwargs):
    ...     print("I am callable!")
    ...   def __call__(self, *args, **kwargs):
    ...     self.example(*args, **kwargs)
    ...
    >>> ex = CallableObject()
    >>> ex()
    I am callable!

上下文管理
---------------

.. code-block:: python

    # 替换 try: ... finally: ...
    # 更多信息可以看: PEP343
    # 通常在打开、关闭时使用

    import socket

    class Socket(object):
        def __init__(self, host, port):
            self.host = host
            self.port = port

        def __enter__(self):
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            sock.bind((self.host,self.port))
            sock.listen(5)
            self.sock = sock
            return self.sock

        def __exit__(self,*exc_info):
            if exc_info[0] is not None:
                import traceback
                traceback.print_exception(*exc_info)
            self.sock.close()

    if __name__=="__main__":
        host = 'localhost'
        port = 5566
        with Socket(host, port) as s:
            while True:
                conn, addr = s.accept()
                msg = conn.recv(1024)
                print(msg)
                conn.send(msg)
                conn.close()

使用contextlib
------------------

.. code-block:: python

    from contextlib import contextmanager

    @contextmanager
    def opening(filename, mode='r'):
       f = open(filename, mode)
       try:
          yield f
       finally:
          f.close()

    with opening('example.txt') as fd:
       fd.read()

Property属性
---------------

.. code-block:: python

    >>> class Example(object):
    ...     def __init__(self, value):
    ...        self._val = value
    ...     @property
    ...     def val(self):
    ...         return self._val
    ...     @val.setter
    ...     def val(self, value):
    ...         if not isinstance(value, int):
    ...             raise TypeError("Expected int")
    ...         self._val = value
    ...     @val.deleter
    ...     def val(self):
    ...         del self._val
    ...
    >>> ex = Example(123)
    >>> ex.val = "str"
    Traceback (most recent call last):
      File "", line 1, in
      File "test.py", line 12, in val
        raise TypeError("Expected int")
    TypeError: Expected int

等同于

.. code-block:: python

    >>> class Example(object):
    ...     def __init__(self, value):
    ...        self._val = value
    ...
    ...     def _val_getter(self):
    ...         return self._val
    ...
    ...     def _val_setter(self, value):
    ...         if not isinstance(value, int):
    ...             raise TypeError("Expected int")
    ...         self._val = value
    ...
    ...     def _val_deleter(self):
    ...         del self._val
    ...
    ...     val = property(fget=_val_getter, fset=_val_setter, fdel=_val_deleter, doc=None)
    ...

计算属性
-------------------

``@property`` 只有当我们需要时，才会去计算一个属性的值。
而不是预先存储在内存中。

.. code-block:: python

    >>> class Example(object):
    ...   @property
    ...   def square3(self):
    ...     return 2**3
    ...
    >>> ex = Example()
    >>> ex.square3
    8

描述符
----------

.. code-block:: python

    >>> class Integer(object):
    ...   def __init__(self, name):
    ...     self._name = name
    ...   def __get__(self, inst, cls):
    ...     if inst is None:
    ...       return self
    ...     else:
    ...       return inst.__dict__[self._name]
    ...   def __set__(self, inst, value):
    ...     if not isinstance(value, int):
    ...       raise TypeError("Expected int")
    ...     inst.__dict__[self._name] = value
    ...   def __delete__(self,inst):
    ...     del inst.__dict__[self._name]
    ...
    >>> class Example(object):
    ...   x = Integer('x')
    ...   def __init__(self, val):
    ...     self.x = val
    ...
    >>> ex1 = Example(1)
    >>> ex1.x
    1
    >>> ex2 = Example("str")
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
      File "<stdin>", line 4, in __init__
      File "<stdin>", line 11, in __set__
    TypeError: Expected an int
    >>> ex3 = Example(3)
    >>> hasattr(ex3, 'x')
    True
    >>> del ex3.x
    >>> hasattr(ex3, 'x')
    False

静态方法和类方法
------------------------

``@classmethod`` 于class绑定。 ``@staticmethod`` 和python的函数类似，只是定义在一个class中。

.. code-block:: python

    >>> class example(object):
    ...   @classmethod
    ...   def clsmethod(cls):
    ...     print("I am classmethod")
    ...   @staticmethod
    ...   def stmethod():
    ...     print("I am staticmethod")
    ...   def instmethod(self):
    ...     print("I am instancemethod")
    ...
    >>> ex = example()
    >>> ex.clsmethod()
    I am classmethod
    >>> ex.stmethod()
    I am staticmethod
    >>> ex.instmethod()
    I am instancemethod
    >>> example.clsmethod()
    I am classmethod
    >>> example.stmethod()
    I am staticmethod
    >>> example.instmethod()
    Traceback (most recent call last):
      File "", line 1, in
    TypeError: unbound method instmethod() ...

抽象方法
---------------

``abc`` 常被用来定义方法，但是没有具体实现。

.. code-block:: python

    >>> from abc import ABCMeta, abstractmethod
    >>> class base(object):
    ...   __metaclass__ = ABCMeta
    ...   @abstractmethod
    ...   def absmethod(self):
    ...     """ Abstract method """
    ...
    >>> class example(base):
    ...   def absmethod(self):
    ...     print("abstract")
    ...
    >>> ex = example()
    >>> ex.absmethod()
    abstract

另一个常用的方法是 ``raise NotImplementedError``

.. code-block:: python

    >>> class base(object):
    ...   def absmethod(self):
    ...     raise NotImplementedError
    ...
    >>> class example(base):
    ...   def absmethod(self):
    ...     print("abstract")
    ...
    >>> ex = example()
    >>> ex.absmethod()
    abstract

使用 slot 去节省内存
-------------------------

.. code-block:: python

    #!/usr/bin/env python3

    import resource
    import platform
    import functools


    def profile_mem(func):
        @functools.wraps(func)
        def wrapper(*a, **k):
            s = resource.getrusage(resource.RUSAGE_SELF).ru_maxrss
            ret = func(*a, **k)
            e = resource.getrusage(resource.RUSAGE_SELF).ru_maxrss

            uname = platform.system()
            if uname == "Linux":
                print(f"mem usage: {e - s} kByte")
            elif uname == "Darwin":
                print(f"mem usage: {e - s} Byte")
            else:
                raise Exception("not support")
            return ret
        return wrapper


    class S(object):
        __slots__ = ['attr1', 'attr2', 'attr3']

        def __init__(self):
            self.attr1 = "Foo"
            self.attr2 = "Bar"
            self.attr3 = "Baz"


    class D(object):

        def __init__(self):
            self.attr1 = "Foo"
            self.attr2 = "Bar"
            self.attr3 = "Baz"


    @profile_mem
    def alloc(cls):
        _ = [cls() for _ in range(1000000)]


    alloc(S)
    alloc(D)

输出:

.. code-block:: console

    $ python3.6 s.py
    mem usage: 70922240 Byte
    mem usage: 100659200 Byte

常用的魔法函数
-------------------

.. code-block:: python

    # 看python文档：数据模型
    # 对于命令类
    __main__
    __name__
    __file__
    __module__
    __all__
    __dict__
    __class__
    __doc__
    __init__(self, [...)
    __str__(self)
    __repr__(self)
    __del__(self)

    # 对于描述符
    __get__(self, instance, owner)
    __set__(self, instance, value)
    __delete__(self, instance)

    #  上下文管理器
    __enter__(self)
    __exit__(self, exc_ty, exc_val, tb)

    # 模拟容器类型
    __len__(self)
    __getitem__(self, key)
    __setitem__(self, key, value)
    __delitem__(self, key)
    __iter__(self)
    __contains__(self, value)

    # 控制属性访问
    __getattr__(self, name)
    __setattr__(self, name, value)
    __delattr__(self, name)
    __getattribute__(self, name)

    # 可调用对象
    __call__(self, [args...])

    # 比较相关
    __cmp__(self, other)
    __eq__(self, other)
    __ne__(self, other)
    __lt__(self, other)
    __gt__(self, other)
    __le__(self, other)
    __ge__(self, other)

    # 算术运算相关
    __add__(self, other)
    __sub__(self, other)
    __mul__(self, other)
    __div__(self, other)
    __mod__(self, other)
    __and__(self, other)
    __or__(self, other)
    __xor__(self, other)
