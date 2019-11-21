.. meta::
    :description lang=en: Collect useful snippets of c extensions
    :keywords: Python, Python3, Python C Extensions, Python C Extensions Cheat Sheet

============
C拓展
============

有时候，这是不可避免的，pythoneers要编写C扩展。
比如说，将C库或新的系统调用移植到Python需要通过C扩展实现新的对象类型。
为了简要介绍C扩展的工作原理。 该备忘单主要致力于编写Python C扩展。

注意这里的C拓展是针对与官方的CPython的。拓展模块可能会在其他的Pytho解释器上不工作，比如`PyPy <https://pypy.org/>`_。
即使是官方的CPython解释器，Python的C API也可能不兼容版本，例如，Python2和Python3。
因此，如果拓展模块想要运行在其他的Python解释器上，最好使用 `ctypes <https://docs.python.org/3/library/ctypes.html>`_
模块或 `cffi <https://cffi.readthedocs.io/en/latest/>`_。

.. contents:: Table of Contents
    :backlinks: none


简单的setup.py
----------------

.. code-block:: python

    from distutils.core import setup, Extension

    ext = Extension('foo', sources=['foo.c'])
    setup(name="Foo", version="1.0", ext_modules=[ext])


自定义CFLAGS
-----------------

.. code-block:: python

    import sysconfig
    from distutils.core import setup, Extension

    cflags = sysconfig.get_config_var("CFLAGS")

    extra_compile_args = cflags.split()
    extra_compile_args += ["-Wextra"]

    ext = Extension(
        "foo", ["foo.c"],
        extra_compile_args=extra_compile_args
    )

    setup(name="foo", version="1.0", ext_modules=[ext])

文档字符串
----------

.. code-block:: c

    PyDoc_STRVAR(doc_mod, "Module document\n");
    PyDoc_STRVAR(doc_foo, "foo() -> None\n\nFoo doc");

    static PyMethodDef methods[] = {
        {"foo", (PyCFunction)foo, METH_NOARGS, doc_foo},
        {NULL, NULL, 0, NULL}
    };

    static struct PyModuleDef module = {
        .m_base    = PyModuleDef_HEAD_INIT,
        .m_name    = "Foo",
        .m_doc     = doc_mod,
        .m_size    = -1,
        .m_methods = methods
    };


简单C拓展
-------------------

foo.c

.. code-block:: c

    #include <Python.h>

    PyDoc_STRVAR(doc_mod, "Module document\n");
    PyDoc_STRVAR(doc_foo, "foo() -> None\n\nFoo doc");

    static PyObject* foo(PyObject* self)
    {
        PyObject* s = PyUnicode_FromString("foo");
        PyObject_Print(s, stdout, 0);
        Py_RETURN_NONE;
    }

    static PyMethodDef methods[] = {
        {"foo", (PyCFunction)foo, METH_NOARGS, doc_foo},
        {NULL, NULL, 0, NULL}
    };

    static struct PyModuleDef module = {
        PyModuleDef_HEAD_INIT, "Foo", doc_mod, -1, methods
    };

    PyMODINIT_FUNC PyInit_foo(void)
    {
        return PyModule_Create(&module);
    }

输出:

.. code-block:: bash

    $ python setup.py -q build
    $ python setup.py -q install
    $ python -c "import foo; foo.foo()"
    'foo'

释放GIL锁
---------------

.. code-block:: c

    #include <Python.h>

    static PyObject* foo(PyObject* self)
    {
        Py_BEGIN_ALLOW_THREADS
        sleep(3);
        Py_END_ALLOW_THREADS
        Py_RETURN_NONE;
    }

    static PyMethodDef methods[] = {
        {"foo", (PyCFunction)foo, METH_NOARGS, NULL},
        {NULL, NULL, 0, NULL}
    };

    static struct PyModuleDef module = {
        PyModuleDef_HEAD_INIT, "Foo", NULL, -1, methods
    };

    PyMODINIT_FUNC PyInit_foo(void)
    {
        return PyModule_Create(&module);
    }

输出:

.. code-block:: bash

    $ python setup.py -q build
    $ python setup.py -q install
    $ python -c "
    > import threading
    > import foo
    > from datetime import datetime
    > def f(n):
    >     now = datetime.now()
    >     print(f'{now}: thread {n}')
    >     foo.foo()
    > ts = [threading.Thread(target=f, args=(n,)) for n in range(3)]
    > [t.start() for t in ts]
    > [t.join() for t in ts]"
    2018-11-04 20:15:34.860454: thread 0
    2018-11-04 20:15:34.860592: thread 1
    2018-11-04 20:15:34.860705: thread 2


在C拓展中，阻塞I/O应该被拆入到包含了 ``Py_BEGIN_ALLOW_THREADS`` 和 ``Py_END_ALLOW_THREADS`` 块中，为了暂时释放GIL锁；
否则，一个阻塞I/O操作不得不等待，知道上一个操作完成。
例如

.. code-block:: c

    #include <Python.h>

    static PyObject* foo(PyObject* self)
    {
        sleep(3);
        Py_RETURN_NONE;
    }

    static PyMethodDef methods[] = {
        {"foo", (PyCFunction)foo, METH_NOARGS, NULL},
        {NULL, NULL, 0, NULL}
    };

    static struct PyModuleDef module = {
        PyModuleDef_HEAD_INIT, "Foo", NULL, -1, methods
    };

    PyMODINIT_FUNC PyInit_foo(void)
    {
        return PyModule_Create(&module);
    }

输出:

.. code-block:: bash

    $ python -c "
    > import threading
    > import foo
    > from datetime import datetime
    > def f(n):
    >     now = datetime.now()
    >     print(f'{now}: thread {n}')
    >     foo.foo()
    > ts = [threading.Thread(target=f, args=(n,)) for n in range(3)]
    > [t.start() for t in ts]
    > [t.join() for t in ts]"
    2018-11-04 20:16:44.055932: thread 0
    2018-11-04 20:16:47.059718: thread 1
    2018-11-04 20:16:50.063579: thread 2

.. warning::

    GIL可以安全地被释放，当 ``Py_BEGIN_ALLOW_THREADS`` 和 ``Py_END_ALLOW_THREADS`` 之间，**没有** Python C API的函数。

取得GIL锁
---------------

.. code-block:: c

    #include <pthread.h>
    #include <Python.h>

    typedef struct {
        PyObject *sec;
        PyObject *py_callback;
    } foo_args;

    void *
    foo_thread(void *args)
    {
        long n = -1;
        PyObject *rv = NULL, *sec = NULL,* py_callback = NULL;
        foo_args *a = NULL;

        if (!args)
            return NULL;

        a = (foo_args *)args;
        sec = a->sec;
        py_callback = a->py_callback;

        n = PyLong_AsLong(sec);
        if ((n == -1) && PyErr_Occurred()) {
            return NULL;
        }

        sleep(n);  // slow task

        // acquire the GIL
        PyGILState_STATE state = PyGILState_Ensure();
        rv = PyObject_CallFunction(py_callback, "s", "Awesome Python!");
        // release the GIL
        PyGILState_Release(state);
        Py_XDECREF(rv);
        return NULL;
    }

    static PyObject *
    foo(PyObject *self, PyObject *args)
    {
        long i = 0, n = 0;
        pthread_t *arr = NULL;
        PyObject *py_callback = NULL;
        PyObject *sec = NULL, *num = NULL;
        PyObject *rv = NULL;
        foo_args a = {};

        if (!PyArg_ParseTuple(args, "OOO:callback", &num, &sec, &py_callback))
            return NULL;

        // allow releasing GIL
        Py_BEGIN_ALLOW_THREADS

        if (!PyLong_Check(sec) || !PyLong_Check(num)) {
            PyErr_SetString(PyExc_TypeError, "should be int");
            goto error;
        }

        if (!PyCallable_Check(py_callback)) {
            PyErr_SetString(PyExc_TypeError, "should be callable");
            goto error;
        }

        n = PyLong_AsLong(num);
        if (n == -1 && PyErr_Occurred())
            goto error;

        arr = (pthread_t *)PyMem_RawCalloc(n, sizeof(pthread_t));
        if (!arr)
            goto error;

        a.sec = sec;
        a.py_callback = py_callback;
        for (i = 0; i < n; i++) {
            if (pthread_create(&arr[i], NULL, foo_thread, &a)) {
                PyErr_SetString(PyExc_TypeError, "create a thread failed");
                goto error;
            }
        }

        for (i = 0; i < n; i++) {
            if (pthread_join(arr[i], NULL)) {
                PyErr_SetString(PyExc_TypeError, "thread join failed");
                goto error;
            }
        }
        Py_XINCREF(Py_None);
        rv = Py_None;
    error:
        PyMem_RawFree(arr);
        Py_XDECREF(sec);
        Py_XDECREF(num);
        Py_XDECREF(py_callback);
        // restore GIL
        Py_END_ALLOW_THREADS
        return rv;
    }

    static PyMethodDef methods[] = {
        {"foo", (PyCFunction)foo, METH_VARARGS, NULL},
        {NULL, NULL, 0, NULL}
    };

    static struct PyModuleDef module = {
        PyModuleDef_HEAD_INIT, "foo", NULL, -1, methods
    };

    PyMODINIT_FUNC PyInit_foo(void)
    {
        return PyModule_Create(&module);
    }

输出:

.. code-block:: bash

    $ python setup.py -q build
    $ python setup.py -q install
    $ pyton -q
    >>> import foo
    >>> from datetime import datetime
    >>> def cb(s):
    ...     now = datetime.now()
    ...     print(f'{now}: {s}')
    ...
    >>> foo.foo(3, 1, cb)
    2018-11-05 09:33:50.642543: Awesome Python!
    2018-11-05 09:33:50.642634: Awesome Python!
    2018-11-05 09:33:50.642672: Awesome Python!

如果线程被C/C++创建，这些线程不会拥有GIL。没有获得GIL锁，
这个解释起不能安全地访问Python的函数。
例如

.. code-block:: c

    void *
    foo_thread(void *args)
    {
        ...
        // without acquiring the GIL
        rv = PyObject_CallFunction(py_callback, "s", "Awesome Python!");
        Py_XDECREF(rv);
        return NULL;
    }

输出:

.. code-block:: bash

    >>> import foo
    >>> from datetime import datetime
    >>> def cb(s):
    ...     now = datetime.now()
    ...     print(f"{now}: {s}")
    ...
    >>> foo.foo(1, 1, cb)
    [2]    8590 segmentation fault  python -q

.. warning::

    为了安全地调用python函数，我们可以简单的包装 **Python函数**
    在 ``PyGILState_Ensure`` 和 ``PyGILState_Release`` C拓展代码。

    .. code-block:: c

        PyGILState_STATE state = PyGILState_Ensure();
        // Perform Python actions
        result = PyObject_CallFunction(callback)
        // Error handling
        PyGILState_Release(state);



获取引用计数
--------------------

.. code-block:: c

    #include <Python.h>

    static PyObject *
    getrefcount(PyObject *self, PyObject *a)
    {
        return PyLong_FromSsize_t(Py_REFCNT(a));
    }

    static PyMethodDef methods[] = {
        {"getrefcount", (PyCFunction)getrefcount, METH_O, NULL},
        {NULL, NULL, 0, NULL}
    };

    static struct PyModuleDef module = {
        PyModuleDef_HEAD_INIT, "foo", NULL, -1, methods
    };

    PyMODINIT_FUNC PyInit_foo(void)
    {
        return PyModule_Create(&module);
    }

输出:

.. code-block:: bash

    $ python setup.py -q build
    $ python setup.py -q install
    $ python -q
    >>> import sys
    >>> import foo
    >>> l = [1, 2, 3]
    >>> sys.getrefcount(l[0])
    104
    >>> foo.getrefcount(l[0])
    104
    >>> i = l[0]
    >>> sys.getrefcount(l[0])
    105
    >>> foo.getrefcount(l[0])
    105

解析参数
----------------

.. code-block:: c

    #include <Python.h>

    static PyObject *
    foo(PyObject *self)
    {
        Py_RETURN_NONE;
    }

    static PyObject *
    bar(PyObject *self, PyObject *arg)
    {
        return Py_BuildValue("O", arg);
    }

    static PyObject *
    baz(PyObject *self, PyObject *args)
    {
        PyObject *x = NULL, *y = NULL;
        if (!PyArg_ParseTuple(args, "OO", &x, &y)) {
            return NULL;
        }
        return Py_BuildValue("OO", x, y);
    }

    static PyObject *
    qux(PyObject *self, PyObject *args, PyObject *kwargs)
    {
        static char *keywords[] = {"x", "y", NULL};
        PyObject *x = NULL, *y = NULL;
        if (!PyArg_ParseTupleAndKeywords(args, kwargs,
                                         "O|O", keywords,
                                         &x, &y))
        {
            return NULL;
        }
        if (!y) {
            y = Py_None;
        }
        return Py_BuildValue("OO", x, y);
    }

    static PyMethodDef methods[] = {
        {"foo", (PyCFunction)foo, METH_NOARGS, NULL},
        {"bar", (PyCFunction)bar, METH_O, NULL},
        {"baz", (PyCFunction)baz, METH_VARARGS, NULL},
        {"qux", (PyCFunction)qux, METH_VARARGS | METH_KEYWORDS, NULL},
        {NULL, NULL, 0, NULL}
    };

    static struct PyModuleDef module = {
        PyModuleDef_HEAD_INIT, "foo", NULL, -1, methods
    };

    PyMODINIT_FUNC PyInit_foo(void)
    {
        return PyModule_Create(&module);
    }

输出:

.. code-block:: bash

    $ python setup.py -q build
    $ python setup.py -q install
    $ python -q
    >>> import foo
    >>> foo.foo()
    >>> foo.bar(3.7)
    3.7
    >>> foo.baz(3, 7)
    (3, 7)
    >>> foo.qux(3, y=7)
    (3, 7)
    >>> foo.qux(x=3, y=7)
    (3, 7)
    >>> foo.qux(x=3)
    (3, None)

调用Python函数
-------------------------

.. code-block:: c

    #include <Python.h>

    static PyObject *
    foo(PyObject *self, PyObject *args)
    {
        PyObject *py_callback = NULL;
        PyObject *rv = NULL;

        if (!PyArg_ParseTuple(args, "O:callback", &py_callback))
            return NULL;

        if (!PyCallable_Check(py_callback)) {
            PyErr_SetString(PyExc_TypeError, "should be callable");
            return NULL;
        }

        // Make sure we own the GIL
        PyGILState_STATE state = PyGILState_Ensure();
        // similar to py_callback("Awesome Python!")
        rv = PyObject_CallFunction(py_callback, "s", "Awesome Python!");
        // Restore previous GIL state
        PyGILState_Release(state);
        return rv;
    }

    static PyMethodDef methods[] = {
        {"foo", (PyCFunction)foo, METH_VARARGS, NULL},
        {NULL, NULL, 0, NULL}
    };

    static struct PyModuleDef module = {
        PyModuleDef_HEAD_INIT, "foo", NULL, -1, methods
    };

    PyMODINIT_FUNC PyInit_foo(void)
    {
        return PyModule_Create(&module);
    }

输出:

.. code-block:: bash

    $ python setup.py -q build
    $ python setup.py -q install
    $ python -c "import foo; foo.foo(print)"
    Awesome Python!

抛出异常
----------------

.. code-block:: c

    #include <Python.h>

    PyDoc_STRVAR(doc_mod, "Module document\n");
    PyDoc_STRVAR(doc_foo, "foo() -> None\n\nFoo doc");

    static PyObject*
    foo(PyObject* self)
    {
        // raise NotImplementedError
        PyErr_SetString(PyExc_NotImplementedError, "Not implemented");
        return NULL;
    }

    static PyMethodDef methods[] = {
        {"foo", (PyCFunction)foo, METH_NOARGS, doc_foo},
        {NULL, NULL, 0, NULL}
    };

    static struct PyModuleDef module = {
        PyModuleDef_HEAD_INIT, "Foo", doc_mod, -1, methods
    };

    PyMODINIT_FUNC PyInit_foo(void)
    {
        return PyModule_Create(&module);
    }

输出:

.. code-block:: bash

    $ python setup.py -q build
    $ python setup.py -q install
    $ python -c "import foo; foo.foo(print)"
    $ python -c "import foo; foo.foo()"
    Traceback (most recent call last):
      File "<string>", line 1, in <module>
    NotImplementedError: Not implemented

自定义异常
--------------------

.. code-block:: c

    #include <stdio.h>
    #include <Python.h>

    static PyObject *FooError;

    PyDoc_STRVAR(doc_foo, "foo() -> void\n\n"
        "Equal to the following example:\n\n"
        "def foo():\n"
        "    raise FooError(\"Raise exception in C\")"
    );

    static PyObject *
    foo(PyObject *self __attribute__((unused)))
    {
        PyErr_SetString(FooError, "Raise exception in C");
        return NULL;
    }

    static PyMethodDef methods[] = {
        {"foo", (PyCFunction)foo, METH_NOARGS, doc_foo},
        {NULL, NULL, 0, NULL}
    };

    static struct PyModuleDef module = {
        PyModuleDef_HEAD_INIT, "foo", "doc", -1, methods
    };

    PyMODINIT_FUNC PyInit_foo(void)
    {
        PyObject *m = NULL;
        m = PyModule_Create(&module);
        if (!m) return NULL;

        FooError = PyErr_NewException("foo.FooError", NULL, NULL);
        Py_INCREF(FooError);
        PyModule_AddObject(m, "FooError", FooError);
        return m;
    }


输出:

.. code-block:: bash

    $ python setup.py -q build
    $ python setup.py -q install
    $ python -c "import foo; foo.foo()"
    Traceback (most recent call last):
      File "<string>", line 1, in <module>
    foo.FooError: Raise exception in C

迭代一个列表
---------------

.. code-block:: c

    #include <Python.h>

    #define PY_PRINTF(o) \
        PyObject_Print(o, stdout, 0); printf("\n");

    static PyObject *
    iter_list(PyObject *self, PyObject *args)
    {
        PyObject *list = NULL, *item = NULL, *iter = NULL;
        PyObject *result = NULL;

        if (!PyArg_ParseTuple(args, "O", &list))
            goto error;

        if (!PyList_Check(list))
            goto error;

        // Get iterator
        iter = PyObject_GetIter(list);
        if (!iter)
            goto error;

        // for i in arr: print(i)
        while ((item = PyIter_Next(iter)) != NULL) {
            PY_PRINTF(item);
            Py_XDECREF(item);
        }

        Py_XINCREF(Py_None);
        result = Py_None;
    error:
        Py_XDECREF(iter);
        return result;
    }

    static PyMethodDef methods[] = {
        {"iter_list", (PyCFunction)iter_list, METH_VARARGS, NULL},
        {NULL, NULL, 0, NULL}
    };

    static struct PyModuleDef module = {
        PyModuleDef_HEAD_INIT, "foo", NULL, -1, methods
    };

    PyMODINIT_FUNC PyInit_foo(void)
    {
        return PyModule_Create(&module);
    }

输出:

.. code-block:: bash

    $ python setup.py -q build
    $ python setup.py -q install
    $ python -c "import foo; foo.iter_list([1,2,3])"
    1
    2
    3

迭代一个字典
---------------------

.. code-block:: c

    #include <Python.h>

    #define PY_PRINTF(o) \
        PyObject_Print(o, stdout, 0); printf("\n");

    static PyObject *
    iter_dict(PyObject *self, PyObject *args)
    {
        PyObject *dict = NULL;
        PyObject *key = NULL, *val = NULL;
        PyObject *o = NULL, *result = NULL;
        Py_ssize_t pos = 0;

        if (!PyArg_ParseTuple(args, "O", &dict))
            goto error;

        // for k, v in d.items(): print(f"({k}, {v})")
        while (PyDict_Next(dict, &pos, &key, &val)) {
            o = PyUnicode_FromFormat("(%S, %S)", key, val);
            if (!o) continue;
            PY_PRINTF(o);
            Py_XDECREF(o);
        }

        Py_INCREF(Py_None);
        result = Py_None;
    error:
        return result;
    }

    static PyMethodDef methods[] = {
        {"iter_dict", (PyCFunction)iter_dict, METH_VARARGS, NULL},
        {NULL, NULL, 0, NULL}
    };

    static struct PyModuleDef module = {
        PyModuleDef_HEAD_INIT, "foo", NULL, -1, methods
    };

    PyMODINIT_FUNC PyInit_foo(void)
    {
        return PyModule_Create(&module);
    }

输出:

.. code-block:: bash

    $ python setup.py -q build
    $ python setup.py -q install
    $ python -c "import foo; foo.iter_dict({'k': 'v'})"
    '(k, v)'

简单类
-------------

.. code-block:: c

    #include <Python.h>

    typedef struct {
        PyObject_HEAD
    } FooObject;

    /* calss Foo(object): pass */

    static PyTypeObject FooType = {
        PyVarObject_HEAD_INIT(NULL, 0)
        .tp_name = "foo.Foo",
        .tp_doc = "Foo objects",
        .tp_basicsize = sizeof(FooObject),
        .tp_itemsize = 0,
        .tp_flags = Py_TPFLAGS_DEFAULT,
        .tp_new = PyType_GenericNew
    };

    static PyModuleDef module = {
        PyModuleDef_HEAD_INIT,
        .m_name = "foo",
        .m_doc = "module foo",
        .m_size = -1
    };

    PyMODINIT_FUNC
    PyInit_foo(void)
    {
        PyObject *m = NULL;
        if (PyType_Ready(&FooType) < 0)
            return NULL;
        if ((m = PyModule_Create(&module)) == NULL)
            return NULL;
        Py_XINCREF(&FooType);
        PyModule_AddObject(m, "Foo", (PyObject *) &FooType);
        return m;
    }

输出:

.. code-block:: bash

    $ python setup.py -q build
    $ python setup.py -q install
    $ python -q
    >>> import foo
    >>> print(type(foo.Foo))
    <class 'type'>
    >>> o = foo.Foo()
    >>> print(type(o))
    <class 'foo.Foo'>
    >>> class Foo(object): ...
    ...
    >>> print(type(Foo))
    <class 'type'>
    >>> o = Foo()
    >>> print(type(o))
    <class '__main__.Foo'>

简单类有成员和方法
--------------------------------------

.. code-block:: c

    #include <Python.h>
    #include <structmember.h>

    /*
     * class Foo:
     *     def __new__(cls, *a, **kw):
     *         foo_obj = object.__new__(cls)
     *         foo_obj.foo = ""
     *         foo_obj.bar = ""
     *         return foo_obj
     *
     *     def __init__(self, foo, bar):
     *         self.foo = foo
     *         self.bar = bar
     *
     *     def fib(self, n):
     *         if n < 2:
     *             return n
     *         return self.fib(n - 1) + self.fib(n - 2)
     */

    typedef struct {
        PyObject_HEAD
        PyObject *foo;
        PyObject *bar;
    } FooObject;

    static void
    Foo_dealloc(FooObject *self)
    {
        Py_XDECREF(self->foo);
        Py_XDECREF(self->bar);
        Py_TYPE(self)->tp_free((PyObject *) self);
    }

    static PyObject *
    Foo_new(PyTypeObject *type, PyObject *args, PyObject *kw)
    {
        int rc = -1;
        FooObject *self = NULL;
        self = (FooObject *) type->tp_alloc(type, 0);

        if (!self) goto error;

        /* allocate attributes */
        self->foo = PyUnicode_FromString("");
        if (self->foo == NULL) goto error;

        self->bar = PyUnicode_FromString("");
        if (self->bar == NULL) goto error;

        rc = 0;
    error:
        if (rc < 0) {
            Py_XDECREF(self->foo);
            Py_XINCREF(self->bar);
            Py_XDECREF(self);
        }
        return (PyObject *) self;
    }

    static int
    Foo_init(FooObject *self, PyObject *args, PyObject *kw)
    {
        int rc = -1;
        static char *keywords[] = {"foo", "bar", NULL};
        PyObject *foo = NULL, *bar = NULL, *ptr = NULL;

        if (!PyArg_ParseTupleAndKeywords(args, kw,
                                        "|OO", keywords,
                                        &foo, &bar))
        {
            goto error;
        }

        if (foo) {
            ptr = self->foo;
            Py_INCREF(foo);
            self->foo = foo;
            Py_XDECREF(ptr);
        }

        if (bar) {
            ptr = self->bar;
            Py_INCREF(bar);
            self->bar = bar;
            Py_XDECREF(ptr);
        }
        rc = 0;
    error:
        return rc;
    }

    static unsigned long
    fib(unsigned long n)
    {
        if (n < 2) return n;
        return fib(n - 1) + fib(n - 2);
    }

    static PyObject *
    Foo_fib(FooObject *self, PyObject *args)
    {
        unsigned long n = 0;
        if (!PyArg_ParseTuple(args, "k", &n)) return NULL;
        return PyLong_FromUnsignedLong(fib(n));
    }

    static PyMemberDef Foo_members[] = {
        {"foo", T_OBJECT_EX, offsetof(FooObject, foo), 0, NULL},
        {"bar", T_OBJECT_EX, offsetof(FooObject, bar), 0, NULL}
    };

    static PyMethodDef Foo_methods[] = {
        {"fib", (PyCFunction)Foo_fib, METH_VARARGS | METH_KEYWORDS, NULL},
        {NULL, NULL, 0, NULL}
    };

    static PyTypeObject FooType = {
        PyVarObject_HEAD_INIT(NULL, 0)
        .tp_name = "foo.Foo",
        .tp_doc = "Foo objects",
        .tp_basicsize = sizeof(FooObject),
        .tp_itemsize = 0,
        .tp_flags = Py_TPFLAGS_DEFAULT | Py_TPFLAGS_BASETYPE,
        .tp_new = Foo_new,
        .tp_init = (initproc) Foo_init,
        .tp_dealloc = (destructor) Foo_dealloc,
        .tp_members = Foo_members,
        .tp_methods = Foo_methods
    };

    static PyModuleDef module = {
        PyModuleDef_HEAD_INIT, "foo", NULL, -1, NULL
    };

    PyMODINIT_FUNC
    PyInit_foo(void)
    {
        PyObject *m = NULL;
        if (PyType_Ready(&FooType) < 0)
            return NULL;
        if ((m = PyModule_Create(&module)) == NULL)
            return NULL;
        Py_XINCREF(&FooType);
        PyModule_AddObject(m, "Foo", (PyObject *) &FooType);
        return m;
    }

输出:

.. code-block:: bash

    $ python setup.py -q build
    $ python setup.py -q install
    $ python -q
    >>> import foo
    >>> o = foo.Foo('foo', 'bar')
    >>> o.foo
    'foo'
    >>> o.bar
    'bar'
    >>> o.fib(10)
    55


简单类有Getter和Setter
-------------------------------------

.. code-block:: c

    #include <Python.h>

    /*
     * class Foo:
     *     def __new__(cls, *a, **kw):
     *         foo_obj = object.__new__(cls)
     *         foo_obj._foo = ""
     *         return foo_obj
     *
     *     def __init__(self, foo=None):
     *         if foo and isinstance(foo, 'str'):
     *             self._foo = foo
     *
     *     @property
     *     def foo(self):
     *         return self._foo
     *
     *     @foo.setter
     *     def foo(self, value):
     *         if not value or not isinstance(value, str):
     *             raise TypeError("value should be unicode")
     *         self._foo = value
     */

    typedef struct {
        PyObject_HEAD
        PyObject *foo;
    } FooObject;

    static void
    Foo_dealloc(FooObject *self)
    {
        Py_XDECREF(self->foo);
        Py_TYPE(self)->tp_free((PyObject *) self);
    }

    static PyObject *
    Foo_new(PyTypeObject *type, PyObject *args, PyObject *kw)
    {
        int rc = -1;
        FooObject *self = NULL;
        self = (FooObject *) type->tp_alloc(type, 0);

        if (!self) goto error;

        /* allocate attributes */
        self->foo = PyUnicode_FromString("");
        if (self->foo == NULL) goto error;

        rc = 0;
    error:
        if (rc < 0) {
            Py_XDECREF(self->foo);
            Py_XDECREF(self);
        }
        return (PyObject *) self;
    }

    static int
    Foo_init(FooObject *self, PyObject *args, PyObject *kw)
    {
        int rc = -1;
        static char *keywords[] = {"foo", NULL};
        PyObject *foo = NULL, *ptr = NULL;

        if (!PyArg_ParseTupleAndKeywords(args, kw,
                                        "|O", keywords,
                                        &foo))
        {
            goto error;
        }

        if (foo && PyUnicode_Check(foo)) {
            ptr = self->foo;
            Py_INCREF(foo);
            self->foo = foo;
            Py_XDECREF(ptr);
        }

        rc = 0;
    error:
        return rc;
    }

    static PyObject *
    Foo_getfoo(FooObject *self, void *closure)
    {
        Py_INCREF(self->foo);
        return self->foo;
    }

    static int
    Foo_setfoo(FooObject *self, PyObject *value, void *closure)
    {
        int rc = -1;

        if (!value || !PyUnicode_Check(value)) {
            PyErr_SetString(PyExc_TypeError, "value should be unicode");
            goto error;
        }
        Py_INCREF(value);
        Py_XDECREF(self->foo);
        self->foo = value;
        rc = 0;
    error:
        return rc;
    }

    static PyGetSetDef Foo_getsetters[] = {
        {"foo", (getter)Foo_getfoo, (setter)Foo_setfoo}
    };

    static PyTypeObject FooType = {
        PyVarObject_HEAD_INIT(NULL, 0)
        .tp_name = "foo.Foo",
        .tp_doc = "Foo objects",
        .tp_basicsize = sizeof(FooObject),
        .tp_itemsize = 0,
        .tp_flags = Py_TPFLAGS_DEFAULT | Py_TPFLAGS_BASETYPE,
        .tp_new = Foo_new,
        .tp_init = (initproc) Foo_init,
        .tp_dealloc = (destructor) Foo_dealloc,
        .tp_getset = Foo_getsetters,
    };

    static PyModuleDef module = {
        PyModuleDef_HEAD_INIT, "foo", NULL, -1, NULL
    };

    PyMODINIT_FUNC
    PyInit_foo(void)
    {
        PyObject *m = NULL;
        if (PyType_Ready(&FooType) < 0)
            return NULL;
        if ((m = PyModule_Create(&module)) == NULL)
            return NULL;
        Py_XINCREF(&FooType);
        PyModule_AddObject(m, "Foo", (PyObject *) &FooType);
        return m;
    }

输出:

.. code-block:: bash

    $ python setup.py -q build
    $ python setup.py -q install
    $ python -q
    >>> import foo
    >>> o = foo.Foo()
    >>> o.foo
    ''
    >>> o.foo = "foo"
    >>> o.foo
    'foo'
    >>> o.foo = None
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
    TypeError: value should be unicode

从其他类继承
-------------------------

.. code-block:: c

    #include <Python.h>
    #include <structmember.h>

    /*
     * class Foo:
     *     def __new__(cls, *a, **kw):
     *         foo_obj = object.__new__(cls)
     *         foo_obj.foo = ""
     *         return foo_obj
     *
     *     def __init__(self, foo):
     *         self.foo = foo
     *
     *     def fib(self, n):
     *         if n < 2:
     *             return n
     *         return self.fib(n - 1) + self.fib(n - 2)
     */

    /* FooObject */

    typedef struct {
        PyObject_HEAD
        PyObject *foo;
    } FooObject;

    static void
    Foo_dealloc(FooObject *self)
    {
        Py_XDECREF(self->foo);
        Py_TYPE(self)->tp_free((PyObject *) self);
    }

    static PyObject *
    Foo_new(PyTypeObject *type, PyObject *args, PyObject *kw)
    {
        int rc = -1;
        FooObject *self = NULL;
        self = (FooObject *) type->tp_alloc(type, 0);

        if (!self) goto error;

        /* allocate attributes */
        self->foo = PyUnicode_FromString("");
        if (self->foo == NULL) goto error;

        rc = 0;
    error:
        if (rc < 0) {
            Py_XDECREF(self->foo);
            Py_XDECREF(self);
        }
        return (PyObject *) self;
    }

    static int
    Foo_init(FooObject *self, PyObject *args, PyObject *kw)
    {
        int rc = -1;
        static char *keywords[] = {"foo", NULL};
        PyObject *foo = NULL, *ptr = NULL;

        if (!PyArg_ParseTupleAndKeywords(args, kw, "|O", keywords, &foo)) {
            goto error;
        }

        if (foo) {
            ptr = self->foo;
            Py_INCREF(foo);
            self->foo = foo;
            Py_XDECREF(ptr);
        }
        rc = 0;
    error:
        return rc;
    }

    static unsigned long
    fib(unsigned long n)
    {
        if (n < 2) return n;
        return fib(n - 1) + fib(n - 2);
    }

    static PyObject *
    Foo_fib(FooObject *self, PyObject *args)
    {
        unsigned long n = 0;
        if (!PyArg_ParseTuple(args, "k", &n)) return NULL;
        return PyLong_FromUnsignedLong(fib(n));
    }

    static PyMemberDef Foo_members[] = {
        {"foo", T_OBJECT_EX, offsetof(FooObject, foo), 0, NULL}
    };

    static PyMethodDef Foo_methods[] = {
        {"fib", (PyCFunction)Foo_fib, METH_VARARGS | METH_KEYWORDS, NULL},
        {NULL, NULL, 0, NULL}
    };

    static PyTypeObject FooType = {
        PyVarObject_HEAD_INIT(NULL, 0)
        .tp_name = "foo.Foo",
        .tp_doc = "Foo objects",
        .tp_basicsize = sizeof(FooObject),
        .tp_itemsize = 0,
        .tp_flags = Py_TPFLAGS_DEFAULT | Py_TPFLAGS_BASETYPE,
        .tp_new = Foo_new,
        .tp_init = (initproc) Foo_init,
        .tp_dealloc = (destructor) Foo_dealloc,
        .tp_members = Foo_members,
        .tp_methods = Foo_methods
    };

    /*
     * class Bar(Foo):
     *     def __init__(self, bar):
     *         super().__init__(bar)
     *
     *     def gcd(self, a, b):
     *         while b:
     *             a, b = b, a % b
     *         return a
     */

    /* BarObject */

    typedef struct {
        FooObject super;
    } BarObject;

    static unsigned long
    gcd(unsigned long a, unsigned long b)
    {
        unsigned long t = 0;
        while (b) {
            t = b;
            b = a % b;
            a = t;
        }
        return a;
    }

    static int
    Bar_init(FooObject *self, PyObject *args, PyObject *kw)
    {
        return FooType.tp_init((PyObject *) self, args, kw);
    }

    static PyObject *
    Bar_gcd(BarObject *self, PyObject *args)
    {
        unsigned long a = 0, b = 0;
        if (!PyArg_ParseTuple(args, "kk", &a, &b)) return NULL;
        return PyLong_FromUnsignedLong(gcd(a, b));
    }

    static PyMethodDef Bar_methods[] = {
        {"gcd", (PyCFunction)Bar_gcd, METH_VARARGS, NULL},
        {NULL, NULL, 0, NULL}
    };

    static PyTypeObject BarType = {
        PyVarObject_HEAD_INIT(NULL, 0)
        .tp_name = "foo.Bar",
        .tp_doc = "Bar objects",
        .tp_basicsize = sizeof(BarObject),
        .tp_itemsize = 0,
        .tp_flags = Py_TPFLAGS_DEFAULT | Py_TPFLAGS_BASETYPE,
        .tp_base = &FooType,
        .tp_init = (initproc) Bar_init,
        .tp_methods = Bar_methods
    };

    /* Module */

    static PyModuleDef module = {
        PyModuleDef_HEAD_INIT, "foo", NULL, -1, NULL
    };

    PyMODINIT_FUNC
    PyInit_foo(void)
    {
        PyObject *m = NULL;
        if (PyType_Ready(&FooType) < 0)
            return NULL;
        if (PyType_Ready(&BarType) < 0)
            return NULL;
        if ((m = PyModule_Create(&module)) == NULL)
            return NULL;

        Py_XINCREF(&FooType);
        Py_XINCREF(&BarType);
        PyModule_AddObject(m, "Foo", (PyObject *) &FooType);
        PyModule_AddObject(m, "Bar", (PyObject *) &BarType);
        return m;
    }

输出:

.. code-block:: bash

    $ python setup.py -q build
    $ python setup.py -q install
    $ python -q
    >>> import foo
    >>> bar = foo.Bar('bar')
    >>> bar.foo
    'bar'
    >>> bar.fib(10)
    55
    >>> bar.gcd(3, 7)
    1

运行一个python命令
---------------------

.. code-block:: c

    #include <stdio.h>
    #include <Python.h>

    int
    main(int argc, char *argv[])
    {
        int rc = -1;
        Py_Initialize();
        rc = PyRun_SimpleString(argv[1]);
        Py_Finalize();
        return rc;
    }

输出:

.. code-block:: bash

    $ clang `python3-config --cflags` -c foo.c -o foo.o
    $ clang `python3-config --ldflags` foo.o -o foo
    $ ./foo "print('Hello Python')"
    Hello Python

运行一个python文件
----------------------

.. code-block:: c

    #include <stdio.h>
    #include <Python.h>

    int
    main(int argc, char *argv[])
    {
        int rc = -1, i = 0;
        wchar_t **argv_copy = NULL;
        const char *filename = NULL;
        FILE *fp = NULL;
        PyCompilerFlags cf = {.cf_flags = 0};

        filename = argv[1];
        fp = fopen(filename, "r");
        if (!fp)
            goto error;

        // copy argv
        argv_copy = PyMem_RawMalloc(sizeof(wchar_t*) * argc);
        if (!argv_copy)
            goto error;

        for (i = 0; i < argc; i++) {
            argv_copy[i] = Py_DecodeLocale(argv[i], NULL);
            if (argv_copy[i]) continue;
            fprintf(stderr, "Unable to decode the argument");
            goto error;
        }

        Py_Initialize();
        Py_SetProgramName(argv_copy[0]);
        PySys_SetArgv(argc, argv_copy);
        rc = PyRun_AnyFileExFlags(fp, filename, 0, &cf);

    error:
        if (argv_copy) {
            for (i = 0; i < argc; i++)
                PyMem_RawFree(argv_copy[i]);
            PyMem_RawFree(argv_copy);
        }
        if (fp) fclose(fp);
        Py_Finalize();
        return rc;
    }

输出:

.. code-block:: bash

    $ clang `python3-config --cflags` -c foo.c -o foo.o
    $ clang `python3-config --ldflags` foo.o -o foo
    $ echo "import sys; print(sys.argv)" > foo.py
    $ ./foo foo.py arg1 arg2 arg3
    ['./foo', 'foo.py', 'arg1', 'arg2', 'arg3']

导入一个Python模块
-----------------------

.. code-block:: c

    #include <stdio.h>
    #include <Python.h>

    #define PYOBJECT_CHECK(obj, label) \
        if (!obj) { \
            PyErr_Print(); \
            goto label; \
        }

    int
    main(int argc, char *argv[])
    {
        int rc = -1;
        wchar_t *program = NULL;
        PyObject *json_module = NULL, *json_dict = NULL;
        PyObject *json_dumps = NULL;
        PyObject *dict = NULL;
        PyObject *result = NULL;

        program = Py_DecodeLocale(argv[0], NULL);
        if (!program) {
            fprintf(stderr, "unable to decode the program name");
            goto error;
        }

        Py_SetProgramName(program);
        Py_Initialize();

        // import json
        json_module = PyImport_ImportModule("json");
        PYOBJECT_CHECK(json_module, error);

        // json_dict = json.__dict__
        json_dict = PyModule_GetDict(json_module);
        PYOBJECT_CHECK(json_dict, error);

        // json_dumps = json.__dict__['dumps']
        json_dumps = PyDict_GetItemString(json_dict, "dumps");
        PYOBJECT_CHECK(json_dumps, error);

        // dict = {'foo': 'Foo', 'bar': 123}
        dict = Py_BuildValue("({sssi})", "foo", "Foo", "bar", 123);
        PYOBJECT_CHECK(dict, error);

        // result = json.dumps(dict)
        result = PyObject_CallObject(json_dumps, dict);
        PYOBJECT_CHECK(result, error);
        PyObject_Print(result, stdout, 0);
        printf("\n");
        rc = 0;

    error:
        Py_XDECREF(result);
        Py_XDECREF(dict);
        Py_XDECREF(json_dumps);
        Py_XDECREF(json_dict);
        Py_XDECREF(json_module);

        PyMem_RawFree(program);
        Py_Finalize();
        return rc;
    }

输出:

.. code-block:: bash

    $ clang `python3-config --cflags` -c foo.c -o foo.o
    $ clang `python3-config --ldflags` foo.o -o foo
    $ ./foo
    '{"foo": "Foo", "bar": 123}'

导入模块里所有东西
------------------------------

.. code-block:: c

    #include <stdio.h>
    #include <Python.h>

    #define PYOBJECT_CHECK(obj, label) \
        if (!obj) { \
            PyErr_Print(); \
            goto label; \
        }


    int
    main(int argc, char *argv[])
    {
        int rc = -1;
        wchar_t *program = NULL;
        PyObject *main_module = NULL, *main_dict = NULL;
        PyObject *uname = NULL;
        PyObject *sysname = NULL;
        PyObject *result = NULL;

        program = Py_DecodeLocale(argv[0], NULL);
        if (!program) {
            fprintf(stderr, "unable to decode the program name");
            goto error;
        }

        Py_SetProgramName(program);
        Py_Initialize();

        // import __main__
        main_module = PyImport_ImportModule("__main__");
        PYOBJECT_CHECK(main_module, error);

        // main_dict = __main__.__dict__
        main_dict = PyModule_GetDict(main_module);
        PYOBJECT_CHECK(main_dict, error);

        // from os import *
        result = PyRun_String("from os import *",
                              Py_file_input,
                              main_dict,
                              main_dict);
        PYOBJECT_CHECK(result, error);
        Py_XDECREF(result);
        Py_XDECREF(main_dict);

        // uname = __main__.__dict__['uname']
        main_dict = PyModule_GetDict(main_module);
        PYOBJECT_CHECK(main_dict, error);

        // result = uname()
        uname = PyDict_GetItemString(main_dict, "uname");
        PYOBJECT_CHECK(uname, error);
        result = PyObject_CallObject(uname, NULL);
        PYOBJECT_CHECK(result, error);

        // sysname = result.sysname
        sysname = PyObject_GetAttrString(result, "sysname");
        PYOBJECT_CHECK(sysname, error);
        PyObject_Print(sysname, stdout, 0);
        printf("\n");

        rc = 0;
    error:
        Py_XDECREF(sysname);
        Py_XDECREF(result);
        Py_XDECREF(uname);
        Py_XDECREF(main_dict);
        Py_XDECREF(main_module);

        PyMem_RawFree(program);
        Py_Finalize();
        return rc;
    }

输出:

.. code-block:: bash

    $ clang `python3-config --cflags` -c foo.c -o foo.o
    $ clang `python3-config --ldflags` foo.o -o foo
    $ ./foo
    'Darwin'

访问属性
------------------

.. code-block:: c

    #include <stdio.h>
    #include <Python.h>

    #define PYOBJECT_CHECK(obj, label) \
        if (!obj) { \
            PyErr_Print(); \
            goto label; \
        }

    int
    main(int argc, char *argv[])
    {
        int rc = -1;
        wchar_t *program = NULL;
        PyObject *json_module = NULL;
        PyObject *json_dumps = NULL;
        PyObject *dict = NULL;
        PyObject *result = NULL;

        program = Py_DecodeLocale(argv[0], NULL);
        if (!program) {
            fprintf(stderr, "unable to decode the program name");
            goto error;
        }

        Py_SetProgramName(program);
        Py_Initialize();

        // import json
        json_module = PyImport_ImportModule("json");
        PYOBJECT_CHECK(json_module, error);

        // json_dumps = json.dumps
        json_dumps = PyObject_GetAttrString(json_module, "dumps");
        PYOBJECT_CHECK(json_dumps, error);

        // dict = {'foo': 'Foo', 'bar': 123}
        dict = Py_BuildValue("({sssi})", "foo", "Foo", "bar", 123);
        PYOBJECT_CHECK(dict, error);

        // result = json.dumps(dict)
        result = PyObject_CallObject(json_dumps, dict);
        PYOBJECT_CHECK(result, error);
        PyObject_Print(result, stdout, 0);
        printf("\n");
        rc = 0;
    error:
        Py_XDECREF(result);
        Py_XDECREF(dict);
        Py_XDECREF(json_dumps);
        Py_XDECREF(json_module);

        PyMem_RawFree(program);
        Py_Finalize();
        return rc;
    }

输出:

.. code-block:: bash

    $ clang `python3-config --cflags` -c foo.c -o foo.o
    $ clang `python3-config --ldflags` foo.o -o foo
    $ ./foo
    '{"foo": "Foo", "bar": 123}'

C拓展的性能
---------------------------

.. code-block:: c

    #include <Python.h>

    static unsigned long
    fib(unsigned long n)
    {
        if (n < 2) return n;
        return fib(n - 1) + fib(n - 2);
    }

    static PyObject *
    fibonacci(PyObject *self, PyObject *args)
    {
        unsigned long n = 0;
        if (!PyArg_ParseTuple(args, "k", &n)) return NULL;
        return PyLong_FromUnsignedLong(fib(n));
    }

    static PyMethodDef methods[] = {
        {"fib", (PyCFunction)fibonacci, METH_VARARGS, NULL},
        {NULL, NULL, 0, NULL}
    };

    static struct PyModuleDef module = {
        PyModuleDef_HEAD_INIT, "foo", NULL, -1, methods
    };

    PyMODINIT_FUNC PyInit_foo(void)
    {
        return PyModule_Create(&module);
    }


Compare the performance with pure Python

.. code-block:: python

    >>> from time import time
    >>> import foo
    >>> def fib(n):
    ...     if n < 2: return n
    ...     return fib(n - 1) + fib(n - 2)
    ...
    >>> s = time(); _ = fib(35); e = time(); e - s
    4.953313112258911
    >>> s = time(); _ = foo.fib(35); e = time(); e - s
    0.04628586769104004

ctypes的性能
----------------------

.. code-block:: c

    // Compile (Mac)
    // -------------
    //
    //   $ clang -Wall -Werror -shared -fPIC -o libfib.dylib fib.c
    //
    unsigned int fib(unsigned int n)
    {
        if ( n < 2) {
            return n;
        }
        return fib(n-1) + fib(n-2);
    }

Compare the performance with pure Python

.. code-block:: python

    >>> from time import time
    >>> from ctypes import CDLL
    >>> def fib(n):
    ...     if n < 2: return n
    ...     return fib(n - 1) + fib(n - 2)
    ...
    >>> cfib = CDLL("./libfib.dylib").fib
    >>> s = time(); _ = fib(35); e = time(); e - s
    4.918856859207153
    >>> s = time(); _ = cfib(35); e = time(); e - s
    0.07283687591552734

ctypes错误处理
----------------------

.. code-block:: python

    from __future__ import print_function

    import os

    from ctypes import *
    from sys import platform, maxsize

    is_64bits = maxsize > 2 ** 32

    if is_64bits and platform == "darwin":
        libc = CDLL("libc.dylib", use_errno=True)
    else:
        raise RuntimeError("Not support platform: {}".format(platform))

    stat = libc.stat

    class Stat(Structure):
        """
        From /usr/include/sys/stat.h

        struct stat {
            dev_t         st_dev;
            ino_t         st_ino;
            mode_t        st_mode;
            nlink_t       st_nlink;
            uid_t         st_uid;
            gid_t         st_gid;
            dev_t         st_rdev;
        #ifndef _POSIX_SOURCE
            struct      timespec st_atimespec;
            struct      timespec st_mtimespec;
            struct      timespec st_ctimespec;
        #else
            time_t        st_atime;
            long          st_atimensec;
            time_t        st_mtime;
            long          st_mtimensec;
            time_t        st_ctime;
            long          st_ctimensec;
        #endif
            off_t         st_size;
            int64_t       st_blocks;
            u_int32_t     st_blksize;
            u_int32_t     st_flags;
            u_int32_t     st_gen;
            int32_t       st_lspare;
            int64_t       st_qspare[2];
        };
        """
        _fields_ = [
            ("st_dev", c_ulong),
            ("st_ino", c_ulong),
            ("st_mode", c_ushort),
            ("st_nlink", c_uint),
            ("st_uid", c_uint),
            ("st_gid", c_uint),
            ("st_rdev", c_ulong),
            ("st_atime", c_longlong),
            ("st_atimendesc", c_long),
            ("st_mtime", c_longlong),
            ("st_mtimendesc", c_long),
            ("st_ctime", c_longlong),
            ("st_ctimendesc", c_long),
            ("st_size", c_ulonglong),
            ("st_blocks", c_int64),
            ("st_blksize", c_uint32),
            ("st_flags", c_uint32),
            ("st_gen", c_uint32),
            ("st_lspare", c_int32),
            ("st_qspare", POINTER(c_int64) * 2),
        ]

    # stat success
    path = create_string_buffer(b"/etc/passwd")
    st = Stat()
    ret = stat(path, byref(st))
    assert ret == 0

    # if stat fail, check errno
    path = create_string_buffer(b"&%$#@!")
    st = Stat()
    ret = stat(path, byref(st))
    if ret != 0:
        errno = get_errno()  # get errno
        errmsg = "stat({}) failed. {}".format(path.raw, os.strerror(errno))
        raise OSError(errno, errmsg)

输出:

.. code-block:: console

    $ python err_handling.py   # python2
    Traceback (most recent call last):
      File "err_handling.py", line 85, in <module>
        raise OSError(errno_, errmsg)
    OSError: [Errno 2] stat(&%$#@!) failed. No such file or directory

    $ python3 err_handling.py  # python3
    Traceback (most recent call last):
      File "err_handling.py", line 85, in <module>
        raise OSError(errno_, errmsg)
    FileNotFoundError: [Errno 2] stat(b'&%$#@!\x00') failed. No such file or directory
