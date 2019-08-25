.. meta::
    :description lang=en: Collect useful snippets of Python typing
    :keywords: Python3, Static Typing, Python Type hints, Type hints Cheat Sheet

======
类型
======

PEP `484 <https://www.python.org/dev/peps/pep-0484/>`_，在Python3中提供一个关于类型系统是什么样子的规范，
引入了类型注解的概念。此外，为了更好的理解类型设计的原则，阅读PEP `483 <https://www.python.org/dev/peps/pep-0483/>`_ 是至关重要的，
它有助于帮助pythoneer理解Python引入类型系统的原因。
这章的主要目的是展示在Python3中类型注解的常用用法。


.. contents:: Table of Contents
    :backlinks: none

没有类型检查
-------------------

.. code-block:: python

    def fib(n):
        a, b = 0, 1
        for _ in range(n):
            yield a
            b, a = a + b, b

    print([n for n in fib(3.6)])


输出:

.. code-block:: bash

    # 直到运行时错误才会检测到

    $ python fib.py
    Traceback (most recent call last):
      File "fib.py", line 8, in <module>
        print([n for n in fib(3.5)])
      File "fib.py", line 8, in <listcomp>
        print([n for n in fib(3.5)])
      File "fib.py", line 3, in fib
        for _ in range(n):
    TypeError: 'float' object cannot be interpreted as an integer


有类型检查
----------------

.. code-block:: python

    # 给一个类型提示
    from typing import Generator

    def fib(n: int) -> Generator:
        a: int = 0
        b: int = 1
        for _ in range(n):
            yield a
            b, a = a + b, b

    print([n for n in fib(3.6)])

输出:

.. code-block:: bash

    # 在运行之前错误就能被发现

    $ mypy --strict fib.py
    fib.py:12: error: Argument 1 to "fib" has incompatible type "float"; expected "int"

基本类型
-----------

.. code-block:: python

    import io
    import re

    from collections import deque, namedtuple
    from typing import (
        Dict,
        List,
        Tuple,
        Set,
        Deque,
        NamedTuple,
        IO,
        Pattern,
        Match,
        Text,
        Optional,
        Sequence,
        Iterable,
        Mapping,
        MutableMapping,
        Any,
    )

    # 没有初始化
    x: int

    # 任意类型
    y: Any
    y = 1
    y = "1"

    # 内建
    var_int: int = 1
    var_str: str = "Hello Typing"
    var_byte: bytes = b"Hello Typing"
    var_bool: bool = True
    var_float: float = 1.
    var_unicode: Text = u'\u2713'

    # 可以是None
    var_could_be_none: Optional[int] = None
    var_could_be_none = 1

    # 集合
    var_set: Set[int] = {i for i in range(3)}
    var_dict: Dict[str, str] = {"foo": "Foo"}
    var_list: List[int] = [i for i in range(3)]
    var_Tuple: Tuple = (1, 2, 3)
    var_deque: Deque = deque([1, 2, 3])
    var_nametuple: NamedTuple = namedtuple('P', ['x', 'y'])

    # io
    var_io_str: IO[str] = io.StringIO("Hello String")
    var_io_byte: IO[bytes] = io.BytesIO(b"Hello Bytes")
    var_io_file_str: IO[str] = open(__file__)
    var_io_file_byte: IO[bytes] = open(__file__, 'rb')

    # re
    p: Pattern = re.compile("(https?)://([^/\r\n]+)(/[^\r\n]*)?")
    m: Optional[Match] = p.match("https://www.python.org/")

    # 鸭子类型：类似列表
    var_seq_list: Sequence[int] = [1, 2, 3]
    var_seq_tuple: Sequence[int] = (1, 2, 3)
    var_iter_list: Iterable[int] = [1, 2, 3]
    var_iter_tuple: Iterable[int] = (1, 2, 3)

    # 鸭子类型：类似字典
    var_map_dict: Mapping[str, str] = {"foo": "Foo"}
    var_mutable_dict: MutableMapping[str, str] = {"bar": "Bar"}

函数
----------

.. code-block:: python

    from typing import Generator, Callable

    # function
    def gcd(a: int, b: int) -> int:
        while b:
            a, b = b, a % b
        return a

    # 回调
    def fun(cb: Callable[[int, int], int]) -> int:
        return cb(55, 66)

    # lambda
    f: Callable[[int], int] = lambda x: x * 2

类
--------

.. code-block:: python

    from typing import ClassVar, Dict, List

    class Foo:

        x: int = 1  # 实例变量，默认等于1
        y: ClassVar[str] = "class var"  # 类变量

        def __init__(self) -> None:
            self.i: List[int] = [0]

        def foo(self, a: int, b: str) -> Dict[int, str]:
            return {a: b}

    foo = Foo()
    foo.x = 123

    print(foo.x)
    print(foo.i)
    print(Foo.y)
    print(foo.foo(1, "abc"))

生成器
----------

.. code-block:: python

    from typing import Generator

    # Generator[YieldType, SendType, ReturnType]
    def fib(n: int) -> Generator[int, None, None]:
        a: int = 0
        b: int = 1
        while n > 0:
            yield a
            b, a = a + b, b
            n -= 1

    g: Generator = fib(10)
    i: Iterator[int] = (x for x in range(3))

异步生成器
-----------------------

.. code-block:: python

    import asyncio

    from typing import AsyncGenerator, AsyncIterator

    async def fib(n: int) -> AsyncGenerator:
        a: int = 0
        b: int = 1
        while n > 0:
            await asyncio.sleep(0.1)
            yield a

            b, a = a + b, b
            n -= 1

    async def main() -> None:
        async for f in fib(10):
            print(f)

        ag: AsyncIterator = (f async for f in fib(10))

    loop = asyncio.get_event_loop()
    loop.run_until_complete(main())

上下文管理器
---------------

.. code-block:: python

    from typing import ContextManager, Generator, IO
    from contextlib import contextmanager

    @contextmanager
    def open_file(name: str) -> Generator:
        f = open(name)
        yield f
        f.close()

    cm: ContextManager[IO] = open_file(__file__)
    with cm as f:
        print(f.read())

异步上下文管理器
-----------------------------

.. code-block:: python

    import asyncio

    from typing import AsyncContextManager, AsyncGenerator, IO
    from contextlib import asynccontextmanager

    # 需要python-3.7或更高版本
    @asynccontextmanager
    async def open_file(name: str) -> AsyncGenerator:
        await asyncio.sleep(0.1)
        f = open(name)
        yield f
        await asyncio.sleep(0.1)
        f.close()

    async def main() -> None:
        acm: AsyncContextManager[IO] = open_file(__file__)
        async with acm as f:
            print(f.read())

    loop = asyncio.get_event_loop()
    loop.run_until_complete(main())

避免 ``None`` 访问
----------------------

.. code-block:: python

    import re

    from typing import Pattern, Dict, Optional

    # like c++
    # std::regex url("(https?)://([^/\r\n]+)(/[^\r\n]*)?");
    # std::regex color("^#?([a-f0-9]{6}|[a-f0-9]{3})$");

    url: Pattern = re.compile("(https?)://([^/\r\n]+)(/[^\r\n]*)?")
    color: Pattern = re.compile("^#?([a-f0-9]{6}|[a-f0-9]{3})$")

    x: Dict[str, Pattern] = {"url": url, "color": color}
    y: Optional[Pattern] = x.get("baz", None)

    print(y.match("https://www.python.org/"))

输出:

.. code-block:: bash

    $ mypy --strict foo.py
    foo.py:15: error: Item "None" of "Optional[Pattern[Any]]" has no attribute "match"

只限位置参数
--------------------------

.. code-block:: python

    # 定义名称以__开头的参数

    def fib(__n: int) -> int:  # positional only arg
        a, b = 0, 1
        for _ in range(__n):
            b, a = a + b, b
        return a


    def gcd(*, a: int, b: int) -> int:  # keyword only arg
        while b:
            a, b = b, a % b
        return a


    print(fib(__n=10))  # error
    print(gcd(10, 5))   # error

输出:

.. code-block:: bash

    mypy --strict foo.py
    foo.py:1: note: "fib" defined here
    foo.py:14: error: Unexpected keyword argument "__n" for "fib"
    foo.py:15: error: Too many positional arguments for "gcd"

多个返回值
-----------------------

.. code-block:: python

    from typing import Tuple, Iterable, Union

    def foo(x: int, y: int) -> Tuple[int, int]:
        return x, y

    # or

    def bar(x: int, y: str) -> Iterable[Union[int, str]]:
        # XXX: not recommend declaring in this way
        return x, y

    a: int
    b: int
    a, b = foo(1, 2)      # ok
    c, d = bar(3, "bar")  # ok

Union[Any, None] 等同于 Optional[Any]
---------------------------------------------

.. code-block:: python

    from typing import List, Union

    def first(l: List[Union[int, None]]) -> Union[int, None]:
        return None if len(l) == 0 else l[0]

    first([None])

    # 等同于

    from typing import List, Optional

    def first(l: List[Optional[int]]) -> Optional[int]:
        return None if len(l) == 0 else l[0]

    first([None])

小心 ``Optional``
---------------------------

.. code-block:: python

    from typing import cast, Optional

    def fib(n):
        a, b = 0, 1
        for _ in range(n):
            b, a = a + b, b
        return a

    def cal(n: Optional[int]) -> None:
        print(fib(n))

    cal(None)

输出:

.. code-block:: bash

    # mypy不会检测到错误
    $ mypy foo.py

准确声明

.. code-block:: python

    from typing import Optional

    def fib(n: int) -> int:  # 定义n是一个int类型
        a, b = 0, 1
        for _ in range(n):
            b, a = a + b, b
        return a

    def cal(n: Optional[int]) -> None:
        print(fib(n))

output:

.. code-block:: bash

    # mypy可以检测到错误，即使我们没有检验None
    $ mypy --strict foo.py
    foo.py:11: error: Argument 1 to "fib" has incompatible type "Optional[int]"; expected "int"

小心casting
----------------------

.. code-block:: python

    from typing import cast, Optional

    def gcd(a: int, b: int) -> int:
        while b:
            a, b = b, a % b
        return a

    def cal(a: Optional[int], b: Optional[int]) -> None:
        # XXX: 避免 casting
        ca, cb = cast(int, a), cast(int, b)
        print(gcd(ca, cb))

    cal(None, None)

输出:

.. code-block:: bash

    # mypy不会检测到错误
    $ mypy --strict foo.py


向前引用
-------------------

基于PEP 484, 如果我们想在声明之前引用一个类型，我们
不得不使用 **字符串** 去表明，稍后会在文件中会出现这种类型的名称。

.. code-block:: python

    from typing import Optional


    class Tree:
        def __init__(
            self, data: int,
            left: Optional["Tree"],  # 向前引用
            right: Optional["Tree"]
        ) -> None:
            self.data = data
            self.left = left
            self.right = right

.. note::

    有一些问题，mypy不会输出关于向前引用的检测信息。
    更多消息可以看 `Issue#948`_。

.. _Issue\#948: https://github.com/python/mypy/issues/948

.. code-block:: python

    class A:
        def __init__(self, a: A) -> None:  # 应该是失败的
            self.a = a

输出:

.. code-block:: bash

    $ mypy --strict type.py
    $ echo $?
    0
    $ python type.py   # 在运行时失败
    Traceback (most recent call last):
      File "type.py", line 1, in <module>
        class A:
      File "type.py", line 2, in A
        def __init__(self, a: A) -> None:  # should fail
    NameError: name 'A' is not defined

推迟执行注解
-----------------------------------

**Python 3.7中的新功能**

- PEP 563_ - 推迟执行注解

.. _563: https://www.python.org/dev/peps/pep-0563/

在Python 3.7之前

.. code-block:: python

    >>> class A:
    ...     def __init__(self, a: A) -> None:
    ...         self._a = a
    ...
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
      File "<stdin>", line 2, in A
    NameError: name 'A' is not defined

在Python 3.7之后(包括3.7)

.. code-block:: python

    >>> from __future__ import annotations
    >>> class A:
    ...     def __init__(self, a: A) -> None:
    ...         self._a = a
    ...

.. note::

    注解只能使用在，变量存在的作用域下。
    因此，**向前引用** 不支持，变量在当前作用域不可信的这种情况。
    **推迟执行注解** 将在Python4.0时，变成默认特性。

类型别名
----------

像 ``typedef`` 或者 ``using`` 在c/c++中的用法

.. code-block:: cpp

    #include <iostream>
    #include <string>
    #include <regex>
    #include <vector>

    typedef std::string Url;
    template<typename T> using Vector = std::vector<T>;

    int main(int argc, char *argv[])
    {
        Url url = "https://python.org";
        std::regex p("(https?)://([^/\r\n]+)(/[^\r\n]*)?");
        bool m = std::regex_match(url, p);
        Vector<int> v = {1, 2};

        std::cout << m << std::endl;
        for (auto it : v) std::cout << it << std::endl;
        return 0;
    }

类型别名可以有一个简单的变量来定义

.. code-block:: python

    import re

    from typing import Pattern, List

    # 像typedef或者using在c/c++中的用法

    # PEP 484提出大写别名
    Url = str

    url: Url = "https://www.python.org/"

    p: Pattern = re.compile("(https?)://([^/\r\n]+)(/[^\r\n]*)?")
    m = p.match(url)

    Vector = List[int]
    v: Vector = [1., 2.]

定义一个 ``NewType``
---------------------

不像别名，``NewType`` returns a separate type but is identical to the original type at runtime.

.. code-block:: python

    from sqlalchemy import Column, String, Integer
    from sqlalchemy.ext.declarative import declarative_base
    from typing import NewType, Any

    # check mypy #2477
    Base: Any = declarative_base()

    # 创建一个新类型
    Id = NewType('Id', int) # 不等同于别名，它是一个'新类型'

    class User(Base):
        __tablename__ = 'User'
        id = Column(Integer, primary_key=True)
        age = Column(Integer, nullable=False)
        name = Column(String, nullable=False)

        def __init__(self, id: Id, age: int, name: str) -> None:
            self.id = id
            self.age = age
            self.name = name

    # 创建用户
    user1 = User(Id(1), 62, "Guido van Rossum") # ok
    user2 = User(2, 48, "David M. Beazley")     # error

输出:

.. code-block:: bash

    $ python foo.py
    $ mypy --ignore-missing-imports foo.py
    foo.py:24: error: Argument 1 to "User" has incompatible type "int"; expected "Id"

进一步阅读:

- `Issue\#1284`_

.. _`Issue\#1284`: https://github.com/python/mypy/issues/1284


使用 ``TypeVar`` 作为模版
------------------------------

像c++的 ``template <typename T>``

.. code-block:: cpp

    #include <iostream>

    template <typename T>
    T add(T x, T y) {
        return x + y;
    }

    int main(int argc, char *argv[])
    {
        std::cout << add(1, 2) << std::endl;
        std::cout << add(1., 2.) << std::endl;
        return 0;
    }

在Python中使用 ``TypeVar``

.. code-block:: python

    from typing import TypeVar

    T = TypeVar("T")

    def add(x: T, y: T) -> T:
        return x + y

    add(1, 2)
    add(1., 2.)

使用 ``TypeVar`` 和 ``Generic`` 作为一个类模版
----------------------------------------------------

像 c++ ``template <typename T> class``

.. code-block:: cpp

    #include <iostream>

    template<typename T>
    class Foo {
    public:
        Foo(T foo) {
            foo_ = foo;
        }
        T Get() {
            return foo_;
        }
    private:
        T foo_;
    };

    int main(int argc, char *argv[])
    {
        Foo<int> f(123);
        std::cout << f.Get() << std::endl;
        return 0;
    }

在Python中定义一个通用类

.. code-block:: python

    from typing import Generic, TypeVar

    T = TypeVar("T")

    class Foo(Generic[T]):
        def __init__(self, foo: T) -> None:
            self.foo = foo

        def get(self) -> T:
            return self.foo

    f: Foo[str] = Foo("Foo")
    v: int = f.get()

输出:

.. code-block:: bash

    $ mypy --strict foo.py
    foo.py:13: error: Incompatible types in assignment (expression has type "str", variable has type "int")

``TypeVar`` 的作用域规则
------------------------------

- ``TypeVar`` 使用不同的通用函数，将会推断为不同的类型。

.. code-block:: python

    from typing import TypeVar

    T = TypeVar("T")

    def foo(x: T) -> T:
        return x

    def bar(y: T) -> T:
        return y

    a: int = foo(1)    # ok: T被推断为int
    b: int = bar("2")  # error: T被推断为str

输出:

.. code-block:: bash

    $ mypy --strict foo.py
    foo.py:12: error: Incompatible types in assignment (expression has type "str", variable has type "int")

- ``TypeVar`` 使用不同的通用函数，将会推断为相同的类型。

.. code-block:: python

    from typing import TypeVar, Generic

    T = TypeVar("T")

    class Foo(Generic[T]):

        def foo(self, x: T) -> T:
            return x

        def bar(self, y: T) -> T:
            return y

    f: Foo[int] = Foo()
    a: int = f.foo(1)    # ok: T被推断为int
    b: str = f.bar("2")  # error: T被期待为int

输出:

.. code-block:: bash

    $ mypy --strict foo.py
    foo.py:15: error: Incompatible types in assignment (expression has type "int", variable has type "str")
    foo.py:15: error: Argument 1 to "bar" of "Foo" has incompatible type "str"; expected "int"

- ``TypeVar`` 在一个方法中使用，但是没有匹配任何参数，这些参数使用 ``Generic`` 可以被推断成不同的类型。

.. code-block:: python

    from typing import TypeVar, Generic

    T = TypeVar("T")
    S = TypeVar("S")

    class Foo(Generic[T]):    # S不能匹配到参数

        def foo(self, x: T, y: S) -> S:
            return y

        def bar(self, z: S) -> S:
            return z

    f: Foo[int] = Foo()
    a: str = f.foo(1, "foo")  # S被推断为str
    b: int = f.bar(12345678)  # S被推断为int

输出:

.. code-block:: bash

    $  mypy --strict foo.py

- 如果它没有绑定类型，``TypeVar`` 不应该出现在函数或方法中。

.. code-block:: python

    from typing import TypeVar, Generic

    T = TypeVar("T")
    S = TypeVar("S")

    def foo(x: T) -> None:
        a: T = x    # ok
        b: S = 123  # error: 非法的类型

输出:

.. code-block:: bash

    $ mypy --strict foo.py
    foo.py:8: error: Invalid type "foo.S"

限制为一组固定的可能类型
----------------------------------------------

``T = TypeVar('T', ClassA, ...)`` 意味着我们创建一个 **type变量同时伴随一个约束**。

.. code-block:: python

    from typing import TypeVar

    # 约束 T = int 或者 T = float
    T = TypeVar("T", int, float)

    def add(x: T, y: T) -> T:
        return x + y

    add(1, 2)
    add(1., 2.)
    add("1", 2)
    add("hello", "world")

output:

.. code-block:: bash

    # mypy can detect wrong type
    $ mypy --strict foo.py
    foo.py:10: error: Value of type variable "T" of "add" cannot be "object"
    foo.py:11: error: Value of type variable "T" of "add" cannot be "str"

具有上限的``TypeVar``
--------------------------------

``T = TypeVar('T', bound=BaseClass)`` 意味着我们创建了一个 **type变量同事伴随一个上限**.
这个概念和c++的 **polymorphism** 是相似的。

.. code-block:: cpp

    #include <iostream>

    class Shape {
    public:
        Shape(double width, double height) {
            width_ = width;
            height_ = height;
        };
        virtual double Area() = 0;
    protected:
        double width_;
        double height_;
    };

    class Rectangle: public Shape {
    public:
        Rectangle(double width, double height)
        :Shape(width, height)
        {};

        double Area() {
            return width_ * height_;
        };
    };

    class Triangle: public Shape {
    public:
        Triangle(double width, double height)
        :Shape(width, height)
        {};

        double Area() {
            return width_ * height_ / 2;
        };
    };

    double Area(Shape &s) {
        return s.Area();
    }

    int main(int argc, char *argv[])
    {
        Rectangle r(1., 2.);
        Triangle t(3., 4.);

        std::cout << Area(r) << std::endl;
        std::cout << Area(t) << std::endl;
        return 0;
    }

与c++一样，创建一个基类和 ``TypeVar``，把基类和它绑定在一起。
然后，静态类型检查会判断每个子类的类型和基类是不是相同的。

.. code-block:: python

    from typing import TypeVar


    class Shape:
        def __init__(self, width: float, height: float) -> None:
            self.width = width
            self.height = height

        def area(self) -> float:
            return 0


    class Rectangle(Shape):
        def area(self) -> float:
            width: float = self.width
            height: float = self.height
            return width * height


    class Triangle(Shape):
        def area(self) -> float:
            width: float = self.width
            height: float = self.height
            return width * height / 2


    S = TypeVar("S", bound=Shape)


    def area(s: S) -> float:
        return s.area()


    r: Rectangle = Rectangle(1, 2)
    t: Triangle = Triangle(3, 4)
    i: int = 5566

    print(area(r))
    print(area(t))
    print(area(i))

output:

.. code-block:: bash

    $ mypy --strict foo.py
    foo.py:40: error: Value of type variable "S" of "area" cannot be "int"

@overload
----------

有时候，我们使用 ``Union`` 去推断函数的返回值有多个不同的类型。
然而，类型检查不能区分，我们想要哪个类型。
因此，下面的代码片段会展示，类型检查无法确定哪种类型是正确的。

.. code-block:: python

    from typing import List, Union


    class Array(object):
        def __init__(self, arr: List[int]) -> None:
            self.arr = arr

        def __getitem__(self, i: Union[int, str]) -> Union[int, str]:
            if isinstance(i, int):
                return self.arr[i]
            if isinstance(i, str):
                return str(self.arr[int(i)])


    arr = Array([1, 2, 3, 4, 5])
    x:int = arr[1]
    y:str = arr["2"]

输出:

.. code-block:: bash

    $ mypy --strict foo.py
    foo.py:16: error: Incompatible types in assignment (expression has type "Union[int, str]", variable has type "int")
    foo.py:17: error: Incompatible types in assignment (expression has type "Union[int, str]", variable has type "str")

虽然，我们可以使用 ``cast`` 解决这个问题，但是它不能避免错字和 ``cast`` 是不安全的。

.. code-block:: python

    from typing import  List, Union, cast


    class Array(object):
        def __init__(self, arr: List[int]) -> None:
            self.arr = arr

        def __getitem__(self, i: Union[int, str]) -> Union[int, str]:
            if isinstance(i, int):
                return self.arr[i]
            if isinstance(i, str):
                return str(self.arr[int(i)])


    arr = Array([1, 2, 3, 4, 5])
    x: int = cast(int, arr[1])
    y: str = cast(str, arr[2])  # typo. we want to assign arr["2"]

output:

.. code-block:: bash

    $ mypy --strict foo.py
    $ echo $?
    0

使用 ``@overload`` 可以解决这个问题，我们可以定义返回值的确切类型。

.. code-block:: python

    from typing import Generic, List, Union, overload


    class Array(object):
        def __init__(self, arr: List[int]) -> None:
            self.arr = arr

        @overload
        def __getitem__(self, i: str) -> str:
            ...

        @overload
        def __getitem__(self, i: int) -> int:
            ...

        def __getitem__(self, i: Union[int, str]) -> Union[int, str]:
            if isinstance(i, int):
                return self.arr[i]
            if isinstance(i, str):
                return str(self.arr[int(i)])


    arr = Array([1, 2, 3, 4, 5])
    x: int = arr[1]
    y: str = arr["2"]

输出:

.. code-block:: bash

    $ mypy --strict foo.py
    $ echo $?
    0

.. warning::

    基于PEP 484, 这个 ``@overload`` 装饰器 **只用于类型检查**，它不回实现一个像c++或者java中的overloading。
    因此，我们不得不实现一个完全不相关的 ``@overload`` 函数。
    在运行时，调用 ``@overload`` 会抛出 ``NotImplementedError``。

.. code-block:: python

    from typing import List, Union, overload


    class Array(object):
        def __init__(self, arr: List[int]) -> None:
            self.arr = arr

        @overload
        def __getitem__(self, i: Union[int, str]) -> Union[int, str]:
            if isinstance(i, int):
                return self.arr[i]
            if isinstance(i, str):
                return str(self.arr[int(i)])


    arr = Array([1, 2, 3, 4, 5])
    try:
        x: int = arr[1]
    except NotImplementedError as e:
        print("NotImplementedError")

输出:

.. code-block:: bash

    $ python foo.py
    NotImplementedError

存根文件
----------

存根文件就像我们通常在c/c++中定义接口的头文件一样。
在python中，我们可以在同一个目录下定义接口或者 ``export MYPYPATH=${stubs}``。

首先，我们需要为模块创建一个存根文件(接口文件)。

.. code-block:: bash

    $ mkdir fib
    $ touch fib/__init__.py fib/__init__.pyi

然后， 定义在 ``__init__.pyi`` 文件中函数的接口并且实现这个模块。

.. code-block:: python

    # fib/__init__.pyi
    def fib(n: int) -> int: ...

    # fib/__init__.py

    def fib(n):
        a, b = 0, 1
        for _ in range(n):
            b, a = a + b, b
        return a

然后, 写一个test.py去测试 ``fib`` 模块。

.. code-block:: python

    # touch test.py
    import sys

    from pathlib import Path

    p = Path(__file__).parent / "fib"
    sys.path.append(str(p))

    from fib import fib

    print(fib(10.0))

输出:

.. code-block:: bash

    $ mypy --strict test.py
    test.py:10: error: Argument 1 to "fib" has incompatible type "float"; expected "int"
