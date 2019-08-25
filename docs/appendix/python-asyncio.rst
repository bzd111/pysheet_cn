.. meta::
    :keywords: Python, Python3, Asyncio

===================================
Yet Another Introduction to Asyncio
===================================

.. contents:: Table of Contents
    :backlinks: none


What is Coroutine?
-------------------

.. code-block:: python

    import asyncio
    import inspect
    from functools import wraps

    Future = asyncio.futures.Future
    def coroutine(func):
        """Simple prototype of coroutine"""
        @wraps(func)
        def coro(*a, **k):
            res = func(*a, **k)
            if isinstance(res, Future) or inspect.isgenerator(res):
                res = yield from res
            return res
        return coro

    @coroutine
    def foo():
        yield from asyncio.sleep(1)
        print("Hello Foo")

    @asyncio.coroutine
    def bar():
        print("Hello Bar")

    loop = asyncio.get_event_loop()
    tasks = [loop.create_task(foo()),
             loop.create_task(bar())]
    loop.run_until_complete(
         asyncio.wait(tasks))
    loop.close()

output:

.. code-block:: console

    $ python test.py
    Hello Bar
    Hello Foo


What is Task?
--------------

.. code-block:: python

    # goal: supervise coroutine run state
    # ref: asyncio/tasks.py

    import asyncio
    Future = asyncio.futures.Future

    class Task(Future):
        """Simple prototype of Task"""

        def __init__(self, gen, *, loop):
            super().__init__(loop=loop)
            self._gen = gen
            self._loop.call_soon(self._step)

        def _step(self, val=None, exc=None):
            try:
                if exc:
                    f = self._gen.throw(exc)
                else:
                    f = self._gen.send(val)
            except StopIteration as e:
                self.set_result(e.value)
            except Exception as e:
                self.set_exception(e)
            else:
                f.add_done_callback(
                     self._wakeup)

        def _wakeup(self, fut):
            try:
                res = fut.result()
            except Exception as e:
                self._step(None, e)
            else:
                self._step(res, None)

    @asyncio.coroutine
    def foo():
        yield from asyncio.sleep(3)
        print("Hello Foo")

    @asyncio.coroutine
    def bar():
        yield from asyncio.sleep(1)
        print("Hello Bar")

    loop = asyncio.get_event_loop()
    tasks = [Task(foo(), loop=loop),
             loop.create_task(bar())]
    loop.run_until_complete(
            asyncio.wait(tasks))
    loop.close()

output:

.. code-block:: console

    $ python test.py
    Hello Bar
    hello Foo

How does event loop work?
-------------------------

.. code-block:: python

    import asyncio
    from collections import deque

    def done_callback(fut):
        fut._loop.stop()

    class Loop:
        """Simple event loop prototype"""

        def __init__(self):
            self._ready = deque()
            self._stopping = False

        def create_task(self, coro):
            Task = asyncio.tasks.Task
            task = Task(coro, loop=self)
            return task

        def run_until_complete(self, fut):
            tasks = asyncio.tasks
            # get task
            fut = tasks.ensure_future(
                        fut, loop=self)
            # add task to ready queue
            fut.add_done_callback(done_callback)
            # run tasks
            self.run_forever()
            # remove task from ready queue
            fut.remove_done_callback(done_callback)

        def run_forever(self):
            """Run tasks until stop"""
            try:
                while True:
                    self._run_once()
                    if self._stopping:
                        break
            finally:
                self._stopping = False

        def call_soon(self, cb, *args):
            """Append task to ready queue"""
            self._ready.append((cb, args))
        def call_exception_handler(self, c):
            pass

        def _run_once(self):
            """Run task at once"""
            ntodo = len(self._ready)
            for i in range(ntodo):
                t, a = self._ready.popleft()
                t(*a)

        def stop(self):
            self._stopping = True

        def close(self):
            self._ready.clear()

        def get_debug(self):
            return False

    @asyncio.coroutine
    def foo():
        print("Foo")

    @asyncio.coroutine
    def bar():
        print("Bar")

    loop = Loop()
    tasks = [loop.create_task(foo()),
             loop.create_task(bar())]
    loop.run_until_complete(
            asyncio.wait(tasks))
    loop.close()

output:

.. code-block:: console

    $ python test.py
    Foo
    Bar


How does ``asyncio.wait`` work?
--------------------------------

.. code-block:: python

    import asyncio

    async def wait(fs, loop=None):
        fs = {asyncio.ensure_future(_) for _ in set(fs)}
        if loop is None:
            loop = asyncio.get_event_loop()

        waiter = loop.create_future()
        counter = len(fs)

        def _on_complete(f):
            nonlocal counter
            counter -= 1
            if counter <= 0 and not waiter.done():
                 waiter.set_result(None)

        for f in fs:
            f.add_done_callback(_on_complete)

        # wait all tasks done
        await waiter

        done, pending = set(), set()
        for f in fs:
            f.remove_done_callback(_on_complete)
            if f.done():
                done.add(f)
            else:
                pending.add(f)
        return done, pending

    async def slow_task(n):
        await asyncio.sleep(n)
        print('sleep "{}" sec'.format(n))

    loop = asyncio.get_event_loop()

    try:
        print("---> wait")
        loop.run_until_complete(
                wait([slow_task(_) for _ in range(1, 3)]))
        print("---> asyncio.wait")
        loop.run_until_complete(
                asyncio.wait([slow_task(_) for _ in range(1, 3)]))
    finally:
        loop.close()

output:

.. code-block:: bash

    ---> wait
    sleep "1" sec
    sleep "2" sec
    ---> asyncio.wait
    sleep "1" sec
    sleep "2" sec

Simple asyncio.run
-------------------

.. code-block:: python

    >>> import asyncio
    >>> async def getaddrinfo(host, port):
    ...     loop = asyncio.get_event_loop()
    ...     return (await loop.getaddrinfo(host, port))
    ...
    >>> def run(main):
    ...     loop = asyncio.new_event_loop()
    ...     asyncio.set_event_loop(loop)
    ...     return loop.run_until_complete(main)
    ...
    >>> ret = run(getaddrinfo('google.com', 443))
    >>> ret = asyncio.run(getaddrinfo('google.com', 443))

How does ``loop.sock_*`` work?
-------------------------------

.. code-block:: python

    import asyncio
    import socket

    def sock_accept(self, sock, fut=None, registed=False):
        fd = sock.fileno()
        if fut is None:
            fut = self.create_future()
        if registed:
            self.remove_reader(fd)
        try:
            conn, addr = sock.accept()
            conn.setblocking(False)
        except (BlockingIOError, InterruptedError):
            self.add_reader(fd, self.sock_accept, sock, fut, True)
        except Exception as e:
            fut.set_exception(e)
        else:
            fut.set_result((conn, addr))
        return fut

    def sock_recv(self, sock, n, fut=None, registed=False):
        fd = sock.fileno()
        if fut is None:
            fut = self.create_future()
        if registed:
            self.remove_reader(fd)
        try:
            data = sock.recv(n)
        except (BlockingIOError, InterruptedError):
            self.add_reader(fd, self.sock_recv, sock, n, fut, True)
        except Exception as e:
            fut.set_exception(e)
        else:
            fut.set_result(data)
        return fut

    def sock_sendall(self, sock, data, fut=None, registed=False):
        fd = sock.fileno()
        if fut is None:
            fut = self.create_future()
        if registed:
            self.remove_writer(fd)
        try:
            n = sock.send(data)
        except (BlockingIOError, InterruptedError):
            n = 0
        except Exception as e:
            fut.set_exception(e)
            return
        if n == len(data):
            fut.set_result(None)
        else:
            if n:
                data = data[n:]
            self.add_writer(fd, sock, data, fut, True)
        return fut

    async def handler(loop, conn):
        while True:
            msg = await loop.sock_recv(conn, 1024)
            if msg: await loop.sock_sendall(conn, msg)
            else: break
        conn.close()

    async def server(loop):
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM, 0)
        sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        sock.setblocking(False)
        sock.bind(('localhost', 9527))
        sock.listen(10)

        while True:
            conn, addr = await loop.sock_accept(sock)
            loop.create_task(handler(loop, conn))

    EventLoop = asyncio.SelectorEventLoop
    EventLoop.sock_accept = sock_accept
    EventLoop.sock_recv = sock_recv
    EventLoop.sock_sendall = sock_sendall
    loop = EventLoop()

    try:
        loop.run_until_complete(server(loop))
    except KeyboardInterrupt:
        pass
    finally:
        loop.close()

output:

.. code-block:: bash

    # console 1
    $ python3 async_sock.py &
    $ nc localhost 9527
    Hello
    Hello

    # console 2
    $ nc localhost 9527
    asyncio
    asyncio


How does ``loop.create_server`` work?
-------------------------------------

.. code-block:: python

    import asyncio
    import socket

    loop = asyncio.get_event_loop()

    async def create_server(loop, protocol_factory, host,
                            port, *args, **kwargs):
       sock = socket.socket(socket.AF_INET,
                            socket.SOCK_STREAM, 0)
       sock.setsockopt(socket.SOL_SOCKET,
                       socket.SO_REUSEADDR, 1)
       sock.setblocking(False)
       sock.bind((host, port))
       sock.listen(10)
       sockets = [sock]
       server = asyncio.base_events.Server(loop, sockets)
       loop._start_serving(protocol_factory, sock, None, server)

       return server


    class EchoProtocol(asyncio.Protocol):
        def connection_made(self, transport):
            peername = transport.get_extra_info('peername')
            print('Connection from {}'.format(peername))
            self.transport = transport

        def data_received(self, data):
            message = data.decode()
            self.transport.write(data)

    # Equal to: loop.create_server(EchoProtocol,
    #                              'localhost', 5566)
    coro = create_server(loop, EchoProtocol, 'localhost', 5566)
    server = loop.run_until_complete(coro)

    try:
        loop.run_forever()
    finally:
        server.close()
        loop.run_until_complete(server.wait_closed())
        loop.close()

output:

.. code-block:: bash

    # console1
    $ nc localhost 5566
    Hello
    Hello

    # console2
    $ nc localhost 5566
    asyncio
    asyncio
