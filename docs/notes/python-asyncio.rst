.. meta::
    :description lang=en: Collect useful snippets of asyncio
    :keywords: Python, Python3, Asyncio, Asyncio Cheat Sheet

=======
Asyncio
=======

.. contents:: Table of Contents
    :backlinks: none

asyncio.run
------------

**New in Python 3.7**

.. code-block:: python

    >>> import asyncio
    >>> from concurrent.futures import ThreadPoolExecutor
    >>> e = ThreadPoolExecutor()
    >>> async def read_file(file_):
    ...     loop = asyncio.get_event_loop()
    ...     with open(file_) as f:
    ...         return (await loop.run_in_executor(e, f.read))
    ...
    >>> ret = asyncio.run(read_file('/etc/passwd'))

Future like object
--------------------

.. code-block:: python

    >>> import sys
    >>> PY_35 = sys.version_info >= (3, 5)
    >>> import asyncio
    >>> loop = asyncio.get_event_loop()
    >>> class SlowObj:
    ...     def __init__(self, n):
    ...         print("__init__")
    ...         self._n = n
    ...     if PY_35:
    ...         def __await__(self):
    ...             print("__await__ sleep({})".format(self._n))
    ...             yield from asyncio.sleep(self._n)
    ...             print("ok")
    ...             return self
    ...
    >>> async def main():
    ...     obj = await SlowObj(3)
    ...
    >>> loop.run_until_complete(main())
    __init__
    __await__ sleep(3)
    ok


Future like object ``__await__`` other task
--------------------------------------------

.. code-block:: python

    >>> import sys
    >>> PY_35 = sys.version_info >= (3, 5)
    >>> import asyncio
    >>> loop = asyncio.get_event_loop()
    >>> async def slow_task(n):
    ...     await asyncio.sleep(n)
    ...
    >>> class SlowObj:
    ...     def __init__(self, n):
    ...         print("__init__")
    ...         self._n = n
    ...     if PY_35:
    ...         def __await__(self):
    ...             print("__await__")
    ...             yield from slow_task(self._n).__await__()
    ...             yield from asyncio.sleep(self._n)
    ...             print("ok")
    ...             return self
    ...
    >>> async def main():
    ...     obj = await SlowObj(1)
    ...
    >>> loop.run_until_complete(main())
    __init__
    __await__
    ok


Patch loop runner ``_run_once``
--------------------------------

.. code-block:: python

    >>> import asyncio
    >>> def _run_once(self):
    ...     num_tasks = len(self._scheduled)
    ...     print("num tasks in queue: {}".format(num_tasks))
    ...     super(asyncio.SelectorEventLoop, self)._run_once()
    ...
    >>> EventLoop = asyncio.SelectorEventLoop
    >>> EventLoop._run_once = _run_once
    >>> loop = EventLoop()
    >>> asyncio.set_event_loop(loop)
    >>> async def task(n):
    ...     await asyncio.sleep(n)
    ...     print("sleep: {} sec".format(n))
    ...
    >>> coro = loop.create_task(task(3))
    >>> loop.run_until_complete(coro)
    num tasks in queue: 0
    num tasks in queue: 1
    num tasks in queue: 0
    sleep: 3 sec
    num tasks in queue: 0
    >>> loop.close()


Put blocking task into Executor
--------------------------------

.. code-block:: python

    >>> import asyncio
    >>> from concurrent.futures import ThreadPoolExecutor
    >>> e = ThreadPoolExecutor()
    >>> loop = asyncio.get_event_loop()
    >>> async def read_file(file_):
    ...     with open(file_) as f:
    ...         data = await loop.run_in_executor(e, f.read)
    ...         return data
    ...
    >>> task = loop.create_task(read_file('/etc/passwd'))
    >>> ret = loop.run_until_complete(task)


Socket with asyncio
-------------------

.. code-block:: python

    import asyncio
    import socket

    host = 'localhost'
    port = 9527
    loop = asyncio.get_event_loop()
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM, 0)
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    s.setblocking(False)
    s.bind((host, port))
    s.listen(10)

    async def handler(conn):
        while True:
            msg = await loop.sock_recv(conn, 1024)
            if not msg:
                break
            await loop.sock_sendall(conn, msg)
        conn.close()

    async def server():
        while True:
            conn, addr = await loop.sock_accept(s)
            loop.create_task(handler(conn))

    loop.create_task(server())
    loop.run_forever()
    loop.close()

output: (bash 1)

.. code-block:: console

    $ nc localhost 9527
    Hello
    Hello

output: (bash 2)

.. code-block:: console

    $ nc localhost 9527
    World
    World


Event Loop with polling
-----------------------

.. code-block:: python

    # using selectors
    # ref: PyCon 2015 - David Beazley

    import asyncio
    import socket
    import selectors
    from collections import deque

    @asyncio.coroutine
    def read_wait(s):
        yield 'read_wait', s

    @asyncio.coroutine
    def write_wait(s):
        yield 'write_wait', s

    class Loop:
        """Simple loop prototype"""

        def __init__(self):
            self.ready = deque()
            self.selector = selectors.DefaultSelector()

        @asyncio.coroutine
        def sock_accept(self, s):
            yield from read_wait(s)
            return s.accept()

        @asyncio.coroutine
        def sock_recv(self, c, mb):
            yield from read_wait(c)
            return c.recv(mb)

        @asyncio.coroutine
        def sock_sendall(self, c, m):
            while m:
                yield from write_wait(c)
                nsent = c.send(m)
                m = m[nsent:]

        def create_task(self, coro):
            self.ready.append(coro)

        def run_forever(self):
            while True:
                self._run_once()

        def _run_once(self):
            while not self.ready:
                events = self.selector.select()
                for k, _ in events:
                    self.ready.append(k.data)
                    self.selector.unregister(k.fileobj)

            while self.ready:
                self.cur_t = ready.popleft()
                try:
                    op, *a = self.cur_t.send(None)
                    getattr(self, op)(*a)
                except StopIteration:
                    pass

        def read_wait(self, s):
            self.selector.register(s, selectors.EVENT_READ, self.cur_t)

        def write_wait(self, s):
            self.selector.register(s, selectors.EVENT_WRITE, self.cur_t)

    loop = Loop()
    host = 'localhost'
    port = 9527

    s = socket.socket(
            socket.AF_INET,
            socket.SOCK_STREAM, 0)
    s.setsockopt(
            socket.SOL_SOCKET,
            socket.SO_REUSEADDR, 1)
    s.setblocking(False)
    s.bind((host, port))
    s.listen(10)

    @asyncio.coroutine
    def handler(c):
        while True:
            msg = yield from loop.sock_recv(c, 1024)
            if not msg:
                break
            yield from loop.sock_sendall(c, msg)
        c.close()

    @asyncio.coroutine
    def server():
        while True:
            c, addr = yield from loop.sock_accept(s)
            loop.create_task(handler(c))

    loop.create_task(server())
    loop.run_forever()


Transport and Protocol
-----------------------

.. code-block:: python

    import asyncio

    class EchoProtocol(asyncio.Protocol):

        def connection_made(self, transport):
            peername = transport.get_extra_info('peername')
            print('Connection from {}'.format(peername))
            self.transport = transport

        def data_received(self, data):
            msg = data.decode()
            self.transport.write(data)

    loop = asyncio.get_event_loop()
    coro = loop.create_server(EchoProtocol, 'localhost', 5566)
    server = loop.run_until_complete(coro)

    try:
        loop.run_forever()
    except:
        loop.run_until_complete(server.wait_closed())
    finally:
        loop.close()

output:

.. code-block:: bash

    # console 1
    $ nc localhost 5566
    Hello
    Hello

    # console 2
    $ nc localhost 5566
    World
    World

Transport and Protocol with SSL
---------------------------------

.. code-block:: python

    import asyncio
    import ssl


    def make_header():
        head = b"HTTP/1.1 200 OK\r\n"
        head += b"Content-Type: text/html\r\n"
        head += b"\r\n"
        return head


    def make_body():
        resp = b"<html>"
        resp += b"<h1>Hello SSL</h1>"
        resp += b"</html>"
        return resp


    sslctx = ssl.SSLContext(ssl.PROTOCOL_SSLv23)
    sslctx.load_cert_chain(
        certfile="./root-ca.crt", keyfile="./root-ca.key"
    )


    class Service(asyncio.Protocol):
        def connection_made(self, tr):
            self.tr = tr
            self.total = 0

        def data_received(self, data):
            if data:
                resp = make_header()
                resp += make_body()
                self.tr.write(resp)
            self.tr.close()


    async def start():
        server = await loop.create_server(
            Service, "localhost", 4433, ssl=sslctx
        )
        await server.wait_closed()


    try:
        loop = asyncio.get_event_loop()
        loop.run_until_complete(start())
    finally:
        loop.close()

output:

.. code-block:: bash

    $ openssl genrsa -out root-ca.key 2048
    $ openssl req -x509 -new -nodes -key root-ca.key -days 365 -out root-ca.crt
    $ python3 ssl_web_server.py

    # then open browser: https://localhost:4433

Asynchronous Iterator
---------------------

.. code-block:: python

    # ref: PEP-0492
    # need Python >= 3.5

    >>> class AsyncIter:
    ...     def __init__(self, it):
    ...         self._it = iter(it)
    ...     async def __aiter__(self):
    ...         return self
    ...     async def __anext__(self):
    ...         await asyncio.sleep(1)
    ...         try:
    ...             val = next(self._it)
    ...         except StopIteration:
    ...             raise StopAsyncIteration
    ...         return val
    ...
    >>> async def foo():
    ...     it = [1, 2, 3]
    ...     async for _ in AsyncIter(it):
    ...         print(_)
    ...
    >>> loop = asyncio.get_event_loop()
    >>> loop.run_until_complete(foo())
    1
    2
    3

What is asynchronous iterator
------------------------------

.. code-block:: python

    >>> import asyncio
    >>> class AsyncIter:
    ...     def __init__(self, it):
    ...         self._it = iter(it)
    ...     async def __aiter__(self):
    ...         return self
    ...     async def __anext__(self):
    ...         await asyncio.sleep(1)
    ...         try:
    ...             val = next(self._it)
    ...         except StopIteration:
    ...             raise StopAsyncIteration
    ...         return val
    ...
    >>> async def foo():
    ...     _ = [1, 2, 3]
    ...     running = True
    ...     it = AsyncIter(_)
    ...     while running:
    ...         try:
    ...             res = await it.__anext__()
    ...             print(res)
    ...         except StopAsyncIteration:
    ...             running = False
    ...
    >>> loop = asyncio.get_event_loop()
    >>> loop.run_until_complete(loop.create_task(foo()))
    1
    2
    3

Asynchronous context manager
----------------------------

.. code-block:: python

    # ref: PEP-0492
    # need Python >= 3.5

    >>> class AsyncCtxMgr:
    ...     async def __aenter__(self):
    ...         await asyncio.sleep(3)
    ...         print("__anter__")
    ...         return self
    ...     async def __aexit__(self, *exc):
    ...         await asyncio.sleep(1)
    ...         print("__aexit__")
    ...
    >>> async def hello():
    ...     async with AsyncCtxMgr() as m:
    ...         print("hello block")
    ...
    >>> async def world():
    ...     print("world block")
    ...
    >>> t = loop.create_task(world())
    >>> loop.run_until_complete(hello())
    world block
    __anter__
    hello block
    __aexit__


What is asynchronous context manager
-------------------------------------

.. code-block:: python

    >>> import asyncio
    >>> class AsyncManager:
    ...     async def __aenter__(self):
    ...         await asyncio.sleep(5)
    ...         print("__aenter__")
    ...     async def __aexit__(self, *exc_info):
    ...         await asyncio.sleep(3)
    ...         print("__aexit__")
    ...
    >>> async def foo():
    ...     import sys
    ...     mgr = AsyncManager()
    ...     await mgr.__aenter__()
    ...     print("body")
    ...     await mgr.__aexit__(*sys.exc_info())
    ...
    >>> loop = asyncio.get_event_loop()
    >>> loop.run_until_complete(loop.create_task(foo()))
    __aenter__
    body
    __aexit__


decorator ``@asynccontextmanager``
------------------------------------

**New in Python 3.7**

- Issue 29679_ - Add @contextlib.asynccontextmanager

.. code-block:: python

    >>> import asyncio
    >>> from contextlib import asynccontextmanager
    >>> @asynccontextmanager
    ... async def coro(msg):
    ...     await asyncio.sleep(1)
    ...     yield msg
    ...     await asyncio.sleep(0.5)
    ...     print('done')
    ...
    >>> async def main():
    ...     async with coro("Hello") as m:
    ...         await asyncio.sleep(1)
    ...         print(m)
    ...
    >>> loop = asyncio.get_event_loop()
    >>> loop.run_until_complete(main())
    Hello
    done

Simple asyncio connection pool
-------------------------------

.. code-block:: python

    import asyncio
    import socket
    import uuid

    class Transport:

        def __init__(self, loop, host, port):
            self.used = False

            self._loop = loop
            self._host = host
            self._port = port
            self._sock = socket.socket(
                    socket.AF_INET, socket.SOCK_STREAM)
            self._sock.setblocking(False)
            self._uuid = uuid.uuid1()

        async def connect(self):
            loop, sock = self._loop, self._sock
            host, port = self._host, self._port
            return (await loop.sock_connect(sock, (host, port)))

        async def sendall(self, msg):
            loop, sock = self._loop, self._sock
            return (await loop.sock_sendall(sock, msg))

        async def recv(self, buf_size):
            loop, sock = self._loop, self._sock
            return (await loop.sock_recv(sock, buf_size))

        def close(self):
            if self._sock: self._sock.close()

        @property
        def alive(self):
            ret = True if self._sock else False
            return ret

        @property
        def uuid(self):
            return self._uuid


    class ConnectionPool:

        def __init__(self, loop, host, port, max_conn=3):
            self._host = host
            self._port = port
            self._max_conn = max_conn
            self._loop = loop

            conns = [Transport(loop, host, port) for _ in range(max_conn)]
            self._conns = conns

        def __await__(self):
            for _c in self._conns:
                yield from _c.connect().__await__()
            return self

        def getconn(self, fut=None):
            if fut is None:
                fut = self._loop.create_future()

            for _c in self._conns:
                if _c.alive and not _c.used:
                    _c.used = True
                    fut.set_result(_c)
                    break
            else:
                loop.call_soon(self.getconn, fut)

            return fut

        def release(self, conn):
            if not conn.used:
                return
            for _c in self._conns:
                if _c.uuid != conn.uuid:
                    continue
                _c.used = False
                break

        def close(self):
            for _c in self._conns:
                _c.close()


    async def handler(pool, msg):
        conn = await pool.getconn()
        byte = await conn.sendall(msg)
        mesg = await conn.recv(1024)
        pool.release(conn)
        return 'echo: {}'.format(mesg)


    async def main(loop, host, port):
        try:
            # creat connection pool
            pool = await ConnectionPool(loop, host, port)

            # generate messages
            msgs = ['coro_{}'.format(_).encode('utf-8') for _ in range(5)]

            # create tasks
            fs = [loop.create_task(handler(pool, _m)) for _m in msgs]

            # wait all tasks done
            done, pending = await asyncio.wait(fs)
            for _ in done: print(_.result())
        finally:
            pool.close()


    loop = asyncio.get_event_loop()
    host = '127.0.0.1'
    port = 9527

    try:
        loop.run_until_complete(main(loop, host, port))
    except KeyboardInterrupt:
        pass
    finally:
        loop.close()

output:

.. code-block:: bash

    $ ncat -l 9527 --keep-open --exec "/bin/cat" &
    $ python3 conn_pool.py
    echo: b'coro_1'
    echo: b'coro_0'
    echo: b'coro_2'
    echo: b'coro_3'
    echo: b'coro_4'

Get domain name
----------------

.. code-block:: python

    >>> import asyncio
    >>> async def getaddrinfo(host, port):
    ...     loop = asyncio.get_event_loop()
    ...     return (await loop.getaddrinfo(host, port))
    ...
    >>> addrs = asyncio.run(getaddrinfo('github.com', 443))
    >>> for a in addrs:
    ...     family, typ, proto, name, sockaddr = a
    ...     print(sockaddr)
    ...
    ('192.30.253.113', 443)
    ('192.30.253.113', 443)
    ('192.30.253.112', 443)
    ('192.30.253.112', 443)

Gather Results
--------------

.. code-block:: python

    import asyncio
    import ssl


    path = ssl.get_default_verify_paths()
    sslctx = ssl.SSLContext()
    sslctx.verify_mode = ssl.CERT_REQUIRED
    sslctx.check_hostname = True
    sslctx.load_verify_locations(path.cafile)


    async def fetch(host, port):
        r, w = await asyncio.open_connection(host, port, ssl=sslctx)
        req = "GET / HTTP/1.1\r\n"
        req += f"Host: {host}\r\n"
        req += "Connection: close\r\n"
        req += "\r\n"

        # send request
        w.write(req.encode())

        # recv response
        resp = ""
        while True:
            line = await r.readline()
            if not line:
                break
            line = line.decode("utf-8")
            resp += line

        # close writer
        w.close()
        await w.wait_closed()
        return resp


    async def main():
        loop = asyncio.get_running_loop()
        url = ["python.org", "github.com", "google.com"]
        fut = [fetch(u, 443) for u in url]
        resps = await asyncio.gather(*fut)
        for r in resps:
            print(r.split("\r\n")[0])


    asyncio.run(main())

output:

.. code-block:: bash

    $ python fetch.py
    HTTP/1.1 301 Moved Permanently
    HTTP/1.1 200 OK
    HTTP/1.1 301 Moved Permanently

Simple asyncio UDP echo server
--------------------------------

.. code-block:: python

    import asyncio
    import socket

    loop = asyncio.get_event_loop()

    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM, 0)
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    sock.setblocking(False)

    host = 'localhost'
    port = 3553

    sock.bind((host, port))

    def recvfrom(loop, sock, n_bytes, fut=None, registed=False):
        fd = sock.fileno()
        if fut is None:
            fut = loop.create_future()
        if registed:
            loop.remove_reader(fd)

        try:
            data, addr = sock.recvfrom(n_bytes)
        except (BlockingIOError, InterruptedError):
            loop.add_reader(fd, recvfrom, loop, sock, n_bytes, fut, True)
        else:
            fut.set_result((data, addr))
        return fut

    def sendto(loop, sock, data, addr, fut=None, registed=False):
        fd = sock.fileno()
        if fut is None:
            fut = loop.create_future()
        if registed:
            loop.remove_writer(fd)
        if not data:
            return

        try:
            n = sock.sendto(data, addr)
        except (BlockingIOError, InterruptedError):
            loop.add_writer(fd, sendto, loop, sock, data, addr, fut, True)
        else:
            fut.set_result(n)
        return fut

    async def udp_server(loop, sock):
        while True:
            data, addr = await recvfrom(loop, sock, 1024)
            n_bytes = await sendto(loop, sock, data, addr)

    try:
        loop.run_until_complete(udp_server(loop, sock))
    finally:
        loop.close()

output:

.. code-block:: bash

    $ python3 udp_server.py
    $ nc -u localhost 3553
    Hello UDP
    Hello UDP


Simple asyncio Web server
-------------------------

.. code-block:: python

    import asyncio
    import socket

    host = 'localhost'
    port = 9527
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    s.setblocking(False)
    s.bind((host, port))
    s.listen(10)

    loop = asyncio.get_event_loop()

    def make_header():
        header  = b"HTTP/1.1 200 OK\r\n"
        header += b"Content-Type: text/html\r\n"
        header += b"\r\n"
        return header

    def make_body():
        resp  = b'<html>'
        resp += b'<body><h3>Hello World</h3></body>'
        resp += b'</html>'
        return resp

    async def handler(conn):
        req = await loop.sock_recv(conn, 1024)
        if req:
            resp = make_header()
            resp += make_body()
            await loop.sock_sendall(conn, resp)
        conn.close()

    async def server(sock, loop):
        while True:
            conn, addr = await loop.sock_accept(sock)
            loop.create_task(handler(conn))

    try:
        loop.run_until_complete(server(s, loop))
    except KeyboardInterrupt:
        pass
    finally:
        loop.close()
        s.close()
    # Then open browser with url: localhost:9527


Simple HTTPS Web Server
------------------------

.. code-block:: python

    import asyncio
    import ssl

    ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
    ctx.load_cert_chain('crt.pem', 'key.pem')

    async def conn(reader, writer):
        _ = await reader.read(1024)
        head = b"HTTP/1.1 200 OK\r\n"
        head += b"Content-Type: text/html\r\n"
        head += b"\r\n"

        body = b"<!doctype html>"
        body += b"<html>"
        body += b"<body><h1>Awesome Python</h1></body>"
        body += b"</html>"

        writer.write(head + body)
        writer.close()


    async def main(host, port):
        srv = await asyncio.start_server(conn, host, port, ssl=ctx)
        async with srv:
            await srv.serve_forever()

    asyncio.run(main('0.0.0.0', 8000))


Simple HTTPS Web server (low-level api)
----------------------------------------

.. code-block:: python

    import asyncio
    import socket
    import ssl

    def make_header():
        head  = b'HTTP/1.1 200 OK\r\n'
        head += b'Content-type: text/html\r\n'
        head += b'\r\n'
        return head

    def make_body():
        resp  = b'<html>'
        resp += b'<h1>Hello SSL</h1>'
        resp += b'</html>'
        return resp

    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM, 0)
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    sock.setblocking(False)
    sock.bind(('localhost' , 4433))
    sock.listen(10)

    sslctx = ssl.SSLContext(ssl.PROTOCOL_SSLv23)
    sslctx.load_cert_chain(certfile='./root-ca.crt',
                           keyfile='./root-ca.key')


    def do_handshake(loop, sock, waiter):
        sock_fd = sock.fileno()
        try:
            sock.do_handshake()
        except ssl.SSLWantReadError:
            loop.remove_reader(sock_fd)
            loop.add_reader(sock_fd, do_handshake,
                            loop, sock, waiter)
            return
        except ssl.SSLWantWriteError:
            loop.remove_writer(sock_fd)
            loop.add_writer(sock_fd, do_handshake,
                            loop, sock, waiter)
            return

        loop.remove_reader(sock_fd)
        loop.remove_writer(sock_fd)
        waiter.set_result(None)


    def handle_read(loop, conn, waiter):
        try:
            req = conn.recv(1024)
        except ssl.SSLWantReadError:
            loop.remove_reader(conn.fileno())
            loop.add_reader(conn.fileno(), handle_read,
                            loop, conn, waiter)
            return
        loop.remove_reader(conn.fileno())
        waiter.set_result(req)


    def handle_write(loop, conn, msg, waiter):
        try:
            resp = make_header()
            resp += make_body()
            ret = conn.send(resp)
        except ssl.SSLWantReadError:
            loop.remove_writer(conn.fileno())
            loop.add_writer(conn.fileno(), handle_write,
                            loop, conn, waiter)
            return
        loop.remove_writer(conn.fileno())
        conn.close()
        waiter.set_result(None)


    async def server(loop):
        while True:
            conn, addr = await loop.sock_accept(sock)
            conn.setblocking(False)
            sslconn = sslctx.wrap_socket(conn,
                                         server_side=True,
                                         do_handshake_on_connect=False)
            # wait SSL handshake
            waiter = loop.create_future()
            do_handshake(loop, sslconn, waiter)
            await waiter

            # wait read request
            waiter = loop.create_future()
            handle_read(loop, sslconn, waiter)
            msg = await waiter

            # wait write response
            waiter = loop.create_future()
            handle_write(loop, sslconn, msg, waiter)
            await waiter

    loop = asyncio.get_event_loop()
    try:
        loop.run_until_complete(server(loop))
    finally:
        loop.close()

output:

.. code-block:: bash

    # console 1

    $ openssl genrsa -out root-ca.key 2048
    $ openssl req -x509 -new -nodes -key root-ca.key -days 365 -out root-ca.crt
    $ python3 Simple_https_server.py

    # console 2

    $ curl https://localhost:4433 -v          \
    >      --resolve localhost:4433:127.0.0.1 \
    >      --cacert ~/test/root-ca.crt


TLS Upgrade
------------

**New in Python 3.7**

.. code-block:: python

    import asyncio
    import ssl


    class HttpClient(asyncio.Protocol):
        def __init__(self, on_con_lost):
            self.on_con_lost = on_con_lost
            self.resp = b""

        def data_received(self, data):
            self.resp += data

        def connection_lost(self, exc):
            resp = self.resp.decode()
            print(resp.split("\r\n")[0])
            self.on_con_lost.set_result(True)


    async def main():
        paths = ssl.get_default_verify_paths()
        sslctx = ssl.SSLContext()
        sslctx.verify_mode = ssl.CERT_REQUIRED
        sslctx.check_hostname = True
        sslctx.load_verify_locations(paths.cafile)

        loop = asyncio.get_running_loop()
        on_con_lost = loop.create_future()

        tr, proto = await loop.create_connection(
            lambda: HttpClient(on_con_lost), "github.com", 443
        )
        new_tr = await loop.start_tls(tr, proto, sslctx)
        req = f"GET / HTTP/1.1\r\n"
        req += "Host: github.com\r\n"
        req += "Connection: close\r\n"
        req += "\r\n"
        new_tr.write(req.encode())

        await on_con_lost
        new_tr.close()


    asyncio.run(main())

output:

.. code-block:: bash

    $ python3 --version
    Python 3.7.0
    $ python3 https.py
    HTTP/1.1 200 OK

Using sendfile
---------------

**New in Python 3.7**

.. code-block:: python

    import asyncio

    path = "index.html"

    async def conn(reader, writer):

        loop = asyncio.get_event_loop()
        _ = await reader.read(1024)

        with open(path, "rb") as f:
            tr = writer.transport
            head = b"HTTP/1.1 200 OK\r\n"
            head += b"Content-Type: text/html\r\n"
            head += b"\r\n"

            tr.write(head)
            await loop.sendfile(tr, f)
            writer.close()

    async def main(host, port):
        # run a simplle http server
        srv = await asyncio.start_server(conn, host, port)
        async with srv:
            await srv.serve_forever()

    asyncio.run(main("0.0.0.0", 8000))

output:

.. code-block:: bash

    $ echo '<!doctype html><h1>Awesome Python</h1>' > index.html
    $ python http.py &
    [2] 60506
    $ curl http://localhost:8000
    <!doctype html><h1>Awesome Python</h1>


Simple asyncio WSGI web server
------------------------------

.. code-block:: python

    # ref: PEP333

    import asyncio
    import socket
    import io
    import sys

    from flask import Flask, Response

    host = 'localhost'
    port = 9527
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    s.setblocking(False)
    s.bind((host, port))
    s.listen(10)

    loop = asyncio.get_event_loop()

    class WSGIServer(object):

        def __init__(self, sock, app):
            self._sock = sock
            self._app = app
            self._header = []

        def parse_request(self, req):
            """ HTTP Request Format:

            GET /hello.htm HTTP/1.1\r\n
            Accept-Language: en-us\r\n
            ...
            Connection: Keep-Alive\r\n
            """
            # bytes to string
            req_info = req.decode('utf-8')
            first_line = req_info.splitlines()[0]
            method, path, ver = first_line.split()
            return method, path, ver

        def get_environ(self, req, method, path):
            env = {}

            # Required WSGI variables
            env['wsgi.version']      = (1, 0)
            env['wsgi.url_scheme']   = 'http'
            env['wsgi.input']        = req
            env['wsgi.errors']       = sys.stderr
            env['wsgi.multithread']  = False
            env['wsgi.multiprocess'] = False
            env['wsgi.run_once']     = False

            # Required CGI variables
            env['REQUEST_METHOD']    = method    # GET
            env['PATH_INFO']         = path      # /hello
            env['SERVER_NAME']       = host      # localhost
            env['SERVER_PORT']       = str(port) # 9527
            return env

        def start_response(self, status, resp_header, exc_info=None):
            header = [('Server', 'WSGIServer 0.2')]
            self.headers_set = [status, resp_header + header]

        async def finish_response(self, conn, data, headers):
            status, resp_header = headers

            # make header
            resp = 'HTTP/1.1 {0}\r\n'.format(status)
            for header in resp_header:
                resp += '{0}: {1}\r\n'.format(*header)
            resp += '\r\n'

            # make body
            resp += '{0}'.format(data)
            try:
                await loop.sock_sendall(conn, str.encode(resp))
            finally:
                conn.close()

        async def run_server(self):
            while True:
                conn, addr = await loop.sock_accept(self._sock)
                loop.create_task(self.handle_request(conn))

        async def handle_request(self, conn):
            # get request data
            req = await loop.sock_recv(conn, 1024)
            if req:
                method, path, ver = self.parse_request(req)
                # get environment
                env = self.get_environ(req, method, path)
                # get application execute result
                res = self._app(env, self.start_response)
                res = [_.decode('utf-8') for _ in list(res)]
                res = ''.join(res)
                loop.create_task(
                     self.finish_response(conn, res, self.headers_set))

    app = Flask(__name__)

    @app.route('/hello')
    def hello():
        return Response("Hello WSGI",mimetype="text/plain")

    server = WSGIServer(s, app.wsgi_app)
    try:
        loop.run_until_complete(server.run_server())
    except:
        pass
    finally:
        loop.close()

    # Then open browser with url: localhost:9527/hello


.. _29679: https://bugs.python.org/issue29679
