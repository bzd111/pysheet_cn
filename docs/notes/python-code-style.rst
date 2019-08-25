===========
代码风格
===========

.. contents:: Table of Contents
    :backlinks: none

命名
------

类命名
^^^^^^^^

差

.. code-block:: python

    class fooClass: ...
    class foo_class: ...

好

.. code-block:: python

    class FooClass: ...


函数命名
^^^^^^^^

差

.. code-block:: python

    def CapCamelCase(*a): ...
    def mixCamelCase(*a): ...

好

.. code-block:: python

    def func_separated_by_underscores(*a): ...

变量命名
^^^^^^^^

差

.. code-block:: python

    FooVar = "CapWords"
    fooVar = "mixedCase"
    Foo_Var = "CapWords_With_Underscore"

好

.. code-block:: python

    # local variable
    var = "lowercase"

    # internal use
    _var = "_single_leading_underscore"

    # avoid conflicts with Python keyword
    var_ = "single_trailing_underscore_"

    # a class attribute (private use in class)
    __var = " __double_leading_underscore"

    # "magic" objects or attributes, ex: __init__
    __name__

    # throwaway variable, ex: _, v = (1, 2)
    _ = "throwaway"
