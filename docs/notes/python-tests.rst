====
Test
====

.. contents:: Table of Contents
    :backlinks: none


一个简单的Python单元测试
------------------------

.. code-block:: python

    # python单元测试只会运行已"tset"开头的函数

    >>> from __future__ import print_function
    >>> import unittest
    >>> class TestFoo(unittest.TestCase):
    ...     def test_foo(self):
    ...             self.assertTrue(True)
    ...     def fun_not_run(self):
    ...             print("no run")
    ...
    >>> unittest.main()
    .
    ----------------------------------------------------------------------
    Ran 1 test in 0.000s

    OK
    >>> import unittest
    >>> class TestFail(unittest.TestCase):
    ...     def test_false(self):
    ...             self.assertTrue(False)
    ...
    >>> unittest.main()
    F
    ======================================================================
    FAIL: test_false (__main__.TestFail)
    ----------------------------------------------------------------------
    Traceback (most recent call last):
      File "<stdin>", line 3, in test_false
    AssertionError: False is not true

    ----------------------------------------------------------------------
    Ran 1 test in 0.000s

    FAILED (failures=1)


Python单元测试setup和teardown层次结构
------------------------------------------

.. code-block:: python

    from __future__ import print_function

    import unittest

    def fib(n):
        return 1 if n<=2 else fib(n-1) + fib(n-2)

    def setUpModule():
            print("setup module")

    def tearDownModule():
            print("teardown module")

    class TestFib(unittest.TestCase):

        def setUp(self):
            print("setUp")
            self.n = 10
        def tearDown(self):
            print("tearDown")
            del self.n

        @classmethod
        def setUpClass(cls):
            print("setUpClass")

        @classmethod
        def tearDownClass(cls):
            print("tearDownClass")

        def test_fib_assert_equal(self):
            self.assertEqual(fib(self.n), 55)

        def test_fib_assert_true(self):
            self.assertTrue(fib(self.n) == 55)

    if __name__ == "__main__":
        unittest.main()

输出:

.. code-block:: console

    $ python test.py
    setup module
    setUpClass
    setUp
    tearDown
    .setUp
    tearDown
    .tearDownClass
    teardown module

    ----------------------------------------------------------------------
    Ran 2 tests in 0.000s

    OK

不同模块的setUp和tearDown层次结构
----------------------------------------------

.. code-block:: python

    # test_module.py
    from __future__ import print_function

    import unittest

    class TestFoo(unittest.TestCase):
        @classmethod
        def setUpClass(self):
            print("foo setUpClass")

        @classmethod
        def tearDownClass(self):
            print("foo tearDownClass")

        def setUp(self):
            print("foo setUp")

        def tearDown(self):
            print("foo tearDown")

        def test_foo(self):
            self.assertTrue(True)

    class TestBar(unittest.TestCase):
        def setUp(self):
            print("bar setUp")

        def tearDown(self):
            print("bar tearDown")

        def test_bar(self):
            self.assertTrue(True)

    # test.py
    from __future__ import print_function

    from test_module import TestFoo
    from test_module import TestBar
    import test_module
    import unittest

    def setUpModule():
        print("setUpModule")

    def tearDownModule():
        print("tearDownModule")


    if __name__ == "__main__":
        test_module.setUpModule = setUpModule
        test_module.tearDownModule = tearDownModule
        suite1 = unittest.TestLoader().loadTestsFromTestCase(TestFoo)
        suite2 = unittest.TestLoader().loadTestsFromTestCase(TestBar)
        suite = unittest.TestSuite([suite1,suite2])
        unittest.TextTestRunner().run(suite)


输出:

.. code-block:: console

    $ python test.py
    setUpModule
    foo setUpClass
    foo setUp
    foo tearDown
    .foo tearDownClass
    bar setUp
    bar tearDown
    .tearDownModule

    ----------------------------------------------------------------------
    Ran 2 tests in 0.000s

    OK

通过unittest.TextTestRunner运行测试
-------------------------------------

.. code-block:: python

    >>> import unittest
    >>> class TestFoo(unittest.TestCase):
    ...     def test_foo(self):
    ...         self.assertTrue(True)
    ...     def test_bar(self):
    ...         self.assertFalse(False)

    >>> suite = unittest.TestLoader().loadTestsFromTestCase(TestFoo)
    >>> unittest.TextTestRunner(verbosity=2).run(suite)
    test_bar (__main__.TestFoo) ... ok
    test_foo (__main__.TestFoo) ... ok

    ----------------------------------------------------------------------
    Ran 2 tests in 0.000s

    OK

测试抛出异常
--------------------

.. code-block:: python

    >>> import unittest

    >>> class TestRaiseException(unittest.TestCase):
    ...     def test_raise_except(self):
    ...         with self.assertRaises(SystemError):
    ...             raise SystemError
    >>> suite_loader = unittest.TestLoader()
    >>> suite = suite_loader.loadTestsFromTestCase(TestRaiseException)
    >>> unittest.TextTestRunner().run(suite)
    .
    ----------------------------------------------------------------------
    Ran 1 test in 0.000s

    OK
    >>> class TestRaiseFail(unittest.TestCase):
    ...     def test_raise_fail(self):
    ...         with self.assertRaises(SystemError):
    ...             pass
    >>> suite = unittest.TestLoader().loadTestsFromTestCase(TestRaiseFail)
    >>> unittest.TextTestRunner(verbosity=2).run(suite)
    test_raise_fail (__main__.TestRaiseFail) ... FAIL

    ======================================================================
    FAIL: test_raise_fail (__main__.TestRaiseFail)
    ----------------------------------------------------------------------
    Traceback (most recent call last):
      File "<stdin>", line 4, in test_raise_fail
    AssertionError: SystemError not raised

    ----------------------------------------------------------------------
    Ran 1 test in 0.000s

    FAILED (failures=1)


传递参数到TestCase
------------------------------

.. code-block:: python

    >>> from __future__ import print_function
    >>> import unittest
    >>> class TestArg(unittest.TestCase):
    ...     def __init__(self, testname, arg):
    ...         super(TestArg, self).__init__(testname)
    ...         self._arg = arg
    ...     def setUp(self):
    ...         print("setUp:", self._arg)
    ...     def test_arg(self):
    ...         print("test_arg:", self._arg)
    ...         self.assertTrue(True)
    ...
    >>> suite = unittest.TestSuite()
    >>> suite.addTest(TestArg('test_arg', 'foo'))
    >>> unittest.TextTestRunner(verbosity=2).run(suite)
    test_arg (__main__.TestArg) ... setUp: foo
    test_arg: foo
    ok

    ----------------------------------------------------------------------
    Ran 1 test in 0.000s

    OK

将多个测试用例分组到一个suite中
-------------------------------------

.. code-block:: python

    >>> import unittest
    >>> class TestFooBar(unittest.TestCase):
    ...     def test_foo(self):
    ...         self.assertTrue(True)
    ...     def test_bar(self):
    ...         self.assertTrue(True)
    ...
    >>> class TestHelloWorld(unittest.TestCase):
    ...     def test_hello(self):
    ...         self.assertEqual("Hello", "Hello")
    ...     def test_world(self):
    ...         self.assertEqual("World", "World")
    ...
    >>> suite_loader = unittest.TestLoader()
    >>> suite1 = suite_loader.loadTestsFromTestCase(TestFooBar)
    >>> suite2 = suite_loader.loadTestsFromTestCase(TestHelloWorld)
    >>> suite = unittest.TestSuite([suite1, suite2])
    >>> unittest.TextTestRunner(verbosity=2).run(suite)
    test_bar (__main__.TestFooBar) ... ok
    test_foo (__main__.TestFooBar) ... ok
    test_hello (__main__.TestHelloWorld) ... ok
    test_world (__main__.TestHelloWorld) ... ok

    ----------------------------------------------------------------------
    Ran 4 tests in 0.000s

    OK

将多个测试用例分组到不同的TestCase
--------------------------------------------

.. code-block:: python

    >>> import unittest
    >>> class TestFoo(unittest.TestCase):
    ...     def test_foo(self):
    ...         assert "foo" == "foo"
    ...
    >>> class TestBar(unittest.TestCase):
    ...     def test_bar(self):
    ...         assert "bar" == "bar"
    ...
    >>> suite = unittest.TestSuite()
    >>> suite.addTest(TestFoo('test_foo'))
    >>> suite.addTest(TestBar('test_bar'))
    >>> unittest.TextTestRunner(verbosity=2).run(suite)
    test_foo (__main__.TestFoo) ... ok
    test_bar (__main__.TestBar) ... ok

    ----------------------------------------------------------------------
    Ran 2 tests in 0.001s

    OK

在TestCase跳过一些测试
-------------------------------

.. code-block:: python

    >>> import unittest
    >>> RUN_FOO = False
    >>> DONT_RUN_BAR = False
    >>> class TestSkip(unittest.TestCase):
    ...     def test_always_run(self):
    ...         self.assertTrue(True)
    ...     @unittest.skip("always skip this test")
    ...     def test_always_skip(self):
    ...         raise RuntimeError
    ...     @unittest.skipIf(RUN_FOO == False, "demo skipIf")
    ...     def test_skipif(self):
    ...         raise RuntimeError
    ...     @unittest.skipUnless(DONT_RUN_BAR == True, "demo skipUnless")
    ...     def test_skipunless(self):
    ...         raise RuntimeError
    ...
    >>> suite = unittest.TestLoader().loadTestsFromTestCase(TestSkip)
    >>> unittest.TextTestRunner(verbosity=2).run(suite)
    test_always_run (__main__.TestSkip) ... ok
    test_always_skip (__main__.TestSkip) ... skipped 'always skip this test'
    test_skipif (__main__.TestSkip) ... skipped 'demo skipIf'
    test_skipunless (__main__.TestSkip) ... skipped 'demo skipUnless'

    ----------------------------------------------------------------------
    Ran 4 tests in 0.000s

    OK (skipped=3)


整体测试
----------------

.. code-block:: python

    >>> from __future__ import print_function
    >>> import unittest
    >>> class Monolithic(unittest.TestCase):
    ...     def step1(self):
    ...         print('step1')
    ...     def step2(self):
    ...         print('step2')
    ...     def step3(self):
    ...         print('step3')
    ...     def _steps(self):
    ...         for attr in sorted(dir(self)):
    ...             if not attr.startswith('step'):
    ...                 continue
    ...             yield attr
    ...     def test_foo(self):
    ...         for _s in self._steps():
    ...             try:
    ...                 getattr(self, _s)()
    ...             except Exception as e:
    ...                 self.fail('{} failed({})'.format(attr, e))
    ...
    >>> suite = unittest.TestLoader().loadTestsFromTestCase(Monolithic)
    >>> unittest.TextTestRunner().run(suite)
    step1
    step2
    step3
    .
    ----------------------------------------------------------------------
    Ran 1 test in 0.000s

    OK
    <unittest.runner.TextTestResult run=1 errors=0 failures=0>


跨模块变量测试文件
------------------------------------

test_foo.py

.. code-block:: python

    from __future__ import print_function

    import unittest

    print(conf)

    class TestFoo(unittest.TestCase):
        def test_foo(self):
            print(conf)

        @unittest.skipIf(conf.isskip==True, "skip test")
        def test_skip(self):
            raise RuntimeError

test_bar.py

.. code-block:: python

    from __future__ import print_function

    import unittest
    import __builtin__

    if __name__ == "__main__":
        conf = type('TestConf', (object,), {})
        conf.isskip = True

        # make a cross-module variable
        __builtin__.conf = conf
        module = __import__('test_foo')
        loader = unittest.TestLoader()
        suite = loader.loadTestsFromTestCase(module.TestFoo)
        unittest.TextTestRunner(verbosity=2).run(suite)

输出:

.. code-block:: console

    $ python test_bar.py
    <class '__main__.TestConf'>
    test_foo (test_foo.TestFoo) ... <class '__main__.TestConf'>
    ok
    test_skip (test_foo.TestFoo) ... skipped 'skip test'

    ----------------------------------------------------------------------
    Ran 2 tests in 0.000s

    OK (skipped=1)


当测试被跳过时，跳过setup和teardown
-----------------------------------------------

.. code-block:: python

    >>> from __future__ import print_function
    >>> import unittest
    >>> class TestSkip(unittest.TestCase):
    ...     def setUp(self):
    ...         print("setUp")
    ...     def tearDown(self):
    ...         print("tearDown")
    ...     @unittest.skip("skip this test")
    ...     def test_skip(self):
    ...         raise RuntimeError
    ...     def test_not_skip(self):
    ...         self.assertTrue(True)
    ...
    >>> suite = unittest.TestLoader().loadTestsFromTestCase(TestSkip)
    >>> unittest.TextTestRunner(verbosity=2).run(suite)
    test_not_skip (__main__.TestSkip) ... setUp
    tearDown
    ok
    test_skip (__main__.TestSkip) ... skipped 'skip this test'

    ----------------------------------------------------------------------
    Ran 2 tests in 0.000s

    OK (skipped=1)

重用老得测试
----------------------

.. code-block:: python

    >>> from __future__ import print_function
    >>> import unittest
    >>> def old_func_test():
    ...     assert "Hello" == "Hello"
    ...
    >>> def old_func_setup():
    ...     print("setup")
    ...
    >>> def old_func_teardown():
    ...     print("teardown")
    ...
    >>> testcase = unittest.FunctionTestCase(old_func_test,
    ...                                      setUp=old_func_setup,
    ...                                      tearDown=old_func_teardown)
    >>> suite = unittest.TestSuite([testcase])
    >>> unittest.TextTestRunner().run(suite)
    setup
    teardown
    .
    ----------------------------------------------------------------------
    Ran 1 test in 0.000s

    OK
    <unittest.runner.TextTestResult run=1 errors=0 failures=0>

正确的测试你的文档
------------------------------

.. code-block:: python

    """
    This is an example of doctest

    >>> fib(10)
    55
    """

    def fib(n):
    """ This function calculate fib number.

    Example:

        >>> fib(10)
        55
        >>> fib(-1)
        Traceback (most recent call last):
        ...
        ValueError
    """
    if n < 0:
        raise ValueError('')
    return 1 if n<=2 else fib(n-1) + fib(n-2)

    if __name__ == "__main__":
        import doctest
        doctest.testmod()

输出:

.. code-block:: console

    $ python demo_doctest.py -v
    Trying:
    fib(10)
    Expecting:
    55
    ok
    Trying:
    fib(10)
    Expecting:
    55
    ok
    Trying:
    fib(-1)
    Expecting:
    Traceback (most recent call last):
    ...
    ValueError
    ok
    2 items passed all tests:
    1 tests in __main__
    2 tests in __main__.fib
    3 tests in 2 items.
    3 passed and 0 failed.
    Test passed.

重用doctest到unittest
----------------------------

.. code-block:: python

    import unittest
    import doctest

    """
    This is an example of doctest

    >>> fib(10)
    55
    """

    def fib(n):
        """ This function calculate fib number.

        Example:

            >>> fib(10)
            55
            >>> fib(-1)
            Traceback (most recent call last):
                ...
            ValueError
        """
        if n < 0:
            raise ValueError('')
        return 1 if n<=2 else fib(n-1) + fib(n-2)

    if __name__ == "__main__":
        finder = doctest.DocTestFinder()
        suite = doctest.DocTestSuite(test_finder=finder)
        unittest.TextTestRunner(verbosity=2).run(suite)

输出:

.. code-block:: console

    fib (__main__)
    Doctest: __main__.fib ... ok

    ----------------------------------------------------------------------
    Ran 1 test in 0.023s

    OK


自定义测试报告
----------------------

.. code-block:: python

    from unittest import (
            TestCase,
            TestLoader,
            TextTestResult,
            TextTestRunner)

    from pprint import pprint
    import unittest
    import os

    OK = 'ok'
    FAIL = 'fail'
    ERROR = 'error'
    SKIP = 'skip'

    class JsonTestResult(TextTestResult):

        def __init__(self, stream, descriptions, verbosity):
            super_class = super(JsonTestResult, self)
            super_class.__init__(stream, descriptions, verbosity)

            # TextTestResult has no successes attr
            self.successes = []

        def addSuccess(self, test):
            # addSuccess do nothing, so we need to overwrite it.
            super(JsonTestResult, self).addSuccess(test)
            self.successes.append(test)

        def json_append(self, test, result, out):
            suite = test.__class__.__name__
            if suite not in out:
                out[suite] = {OK: [], FAIL: [], ERROR:[], SKIP: []}
            if result is OK:
                out[suite][OK].append(test._testMethodName)
            elif result is FAIL:
                out[suite][FAIL].append(test._testMethodName)
            elif result is ERROR:
                out[suite][ERROR].append(test._testMethodName)
            elif result is SKIP:
                out[suite][SKIP].append(test._testMethodName)
            else:
                raise KeyError("No such result: {}".format(result))
            return out

        def jsonify(self):
            json_out = dict()
            for t in self.successes:
                json_out = self.json_append(t, OK, json_out)

            for t, _ in self.failures:
                json_out = self.json_append(t, FAIL, json_out)

            for t, _ in self.errors:
                json_out = self.json_append(t, ERROR, json_out)

            for t, _ in self.skipped:
                json_out = self.json_append(t, SKIP, json_out)

            return json_out

    class TestSimple(TestCase):

        def test_ok_1(self):
            foo = True
            self.assertTrue(foo)

        def test_ok_2(self):
            bar = True
            self.assertTrue(bar)

        def test_fail(self):
            baz = False
            self.assertTrue(baz)

        def test_raise(self):
            raise RuntimeError

        @unittest.skip("Test skip")
        def test_skip(self):
            raise NotImplementedError

    if __name__ == '__main__':
        # redirector default 输出 of unittest to /dev/null
        with open(os.devnull, 'w') as null_stream:
            # new a runner and overwrite resultclass of runner
            runner = TextTestRunner(stream=null_stream)
            runner.resultclass = JsonTestResult

            # create a testsuite
            suite = TestLoader().loadTestsFromTestCase(TestSimple)

            # run the testsuite
            result = runner.run(suite)

            # print json 输出
            pprint(result.jsonify())

输出:

.. code-block:: bash

    $ python test.py
    {'TestSimple': {'error': ['test_raise'],
                    'fail': ['test_fail'],
                    'ok': ['test_ok_1', 'test_ok_2'],
                    'skip': ['test_skip']}}


Mock - 使用 ``@patch`` 替换原始方法
----------------------------------------------------

.. code-block:: python

    # python-3.3 or above

    >>> from unittest.mock import patch
    >>> import os
    >>> def fake_remove(path, *a, **k):
    ...     print("remove done")
    ...
    >>> @patch('os.remove', fake_remove)
    ... def test():
    ...     try:
    ...         os.remove('%$!?&*') # fake os.remove
    ...     except OSError as e:
    ...         print(e)
    ...     else:
    ...         print('test success')
    ...
    >>> test()
    remove done
    test success

.. note::

    不使用mock，上面的测试会一直失败。

.. code-block:: python

    >>> import os
    >>> def test():
    ...     try:
    ...         os.remove('%$!?&*')
    ...     except OSError as e:
    ...         print(e)
    ...     else:
    ...         print('test success')
    ...
    >>> test()
    [Errno 2] No such file or directory: '%$!?&*'


``with unittest.mock.patch`` 到底做了什么?
----------------------------------------------

.. code-block:: python

    from unittest.mock import patch
    import os

    PATH = '$@!%?&'

    def fake_remove(path):
        print("Fake remove")


    class SimplePatch:

        def __init__(self, target, new):
            self._target = target
            self._new = new

        def get_target(self, target):
            target, attr = target.rsplit('.', 1)
            getter = __import__(target)
            return getter, attr

        def __enter__(self):
            orig, attr = self.get_target(self._target)
            self.orig, self.attr = orig, attr
            self.orig_attr = getattr(orig, attr)
            setattr(orig, attr, self._new)
            return self._new

        def __exit__(self, *exc_info):
            setattr(self.orig, self.attr, self.orig_attr)
            del self.orig_attr


    print('---> inside unittest.mock.patch scope')
    with patch('os.remove', fake_remove):
        os.remove(PATH)

    print('---> inside simple patch scope')
    with SimplePatch('os.remove', fake_remove):
        os.remove(PATH)

    print('---> outside patch scope')
    try:
        os.remove(PATH)
    except OSError as e:
        print(e)

输出:

.. code-block:: bash

    $ python3 simple_patch.py
    ---> inside unittest.mock.patch scope
    Fake remove
    ---> inside simple patch scope
    Fake remove
    ---> outside patch scope
    [Errno 2] No such file or directory: '$@!%?&'


Mock - 替换 ``open``
---------------------------

.. code-block:: python

    >>> import urllib
    >>> from unittest.mock import patch, mock_open
    >>> def send_req(url):
    ...     with urllib.request.urlopen(url) as f:
    ...         if f.status == 200:
    ...             return f.read()
    ...         raise urllib.error.URLError
    ...
    >>> fake_html = b'<html><h1>Mock Content</h1></html>'
    >>> mock_urlopen = mock_open(read_data=fake_html)
    >>> ret = mock_urlopen.return_value
    >>> ret.status = 200
    >>> @patch('urllib.request.urlopen', mock_urlopen)
    ... def test_send_req_success():
    ...     try:
    ...         ret = send_req('http://www.mockurl.com')
    ...         assert ret == fake_html
    ...     except Exception as e:
    ...         print(e)
    ...     else:
    ...         print('test send_req success')
    ...
    >>> test_send_req_success()
    test send_req success
    >>> ret = mock_urlopen.return_value
    >>> ret.status = 404
    >>> @patch('urllib.request.urlopen', mock_urlopen)
    ... def test_send_req_fail():
    ...     try:
    ...         ret = send_req('http://www.mockurl.com')
    ...         assert ret == fake_html
    ...     except Exception as e:
    ...         print('test fail success')
    ...
    >>> test_send_req_fail()
    test fail success
