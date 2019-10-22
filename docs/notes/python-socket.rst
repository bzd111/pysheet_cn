.. meta::
    :description lang=en: Collect useful snippets of Python socket
    :keywords: Python, Python3, Python Socket, Python Socket Cheat Sheet

======
Socket
======

对于大部分程序员来说，即使Python提供了很多高级的网络接口,例如httplib、urllib、imaplib、telnetlib等，
socket编程还是不可避免的。一些类Unix系统的接口是通过socket接口调用的，例如Netlink，内核加密。
为了减轻如冗长文档或者源码的痛苦，这个备忘录尝收集一些常用的或不常用的低级的socket编程的代码片段。

.. contents:: Table of Contents
    :backlinks: none

获取主机名
------------

.. code-block:: python

    >>> import socket
    >>> socket.gethostname()
    'MacBookPro-4380.local'
    >>> hostname = socket.gethostname()
    >>> socket.gethostbyname(hostname)
    '172.20.10.4'
    >>> socket.gethostbyname('localhost')
    '127.0.0.1'

从字符串中获取地址族和socket地址
-------------------------------------------------

.. code-block:: python

    import socket
    import sys


    try:
        for res in socket.getaddrinfo(sys.argv[1], None,
                                      proto=socket.IPPROTO_TCP):
            family = res[0]
            sockaddr = res[4]
            print(family, sockaddr)
    except socket.gaierror:
        print("Invalid")

输出:

.. code-block:: console

    $ gai.py 192.0.2.244
    AddressFamily.AF_INET ('192.0.2.244', 0)
    $ gai.py 2001:db8:f00d::1:d
    AddressFamily.AF_INET6 ('2001:db8:f00d::1:d', 0, 0, 0)
    $ gai.py www.google.com
    AddressFamily.AF_INET6 ('2607:f8b0:4006:818::2004', 0, 0, 0)
    AddressFamily.AF_INET ('172.217.10.132', 0)

它处理异常情况，合法和不合法：

.. code-block:: console

    $ gai.py 10.0.0.256  # octet overflow
    Invalid
    $ gai.py not-exist.example.com  # unresolvable
    Invalid
    $ gai.py fe80::1%eth0  # scoped
    AddressFamily.AF_INET6 ('fe80::1%eth0', 0, 0, 2)
    $ gai.py ::ffff:192.0.2.128  # IPv4-Mapped
    AddressFamily.AF_INET6 ('::ffff:192.0.2.128', 0, 0, 0)
    $ gai.py 0xc000027b  # IPv4 in hex
    AddressFamily.AF_INET ('192.0.2.123', 0)
    $ gai.py 3221226198  # IPv4 in decimal
    AddressFamily.AF_INET ('192.0.2.214', 0)

转换主机和网络端h
--------------------------------

.. code-block:: python

    # 小端机器
    >>> import socket
    >>> a = 1 # host endian
    >>> socket.htons(a) # network endian
    256
    >>> socket.htonl(a) # network endian
    16777216
    >>> socket.ntohs(256) # host endian
    1
    >>> socket.ntohl(16777216) # host endian
    1

    # 大端机器
    >>> import socket
    >>> a = 1 # host endian
    >>> socket.htons(a) # network endian
    1
    >>> socket.htonl(a) # network endian
    1L
    >>> socket.ntohs(1) # host endian
    1
    >>> socket.ntohl(1) # host endian
    1L


IP点分字符串和字节格式转换
-------------------------------------------

.. code-block:: python

    >>> import socket
    >>> addr = socket.inet_aton('127.0.0.1')
    >>> addr
    '\x7f\x00\x00\x01'
    >>> socket.inet_ntoa(addr)
    '127.0.0.1'

Mac地址和字节格式转换
---------------------------------

.. code-block:: python

    >>> import binascii
    >>> mac = '00:11:32:3c:c3:0b'
    >>> byte = binascii.unhexlify(mac.replace(':', ''))
    >>> byte
    '\x00\x112<\xc3\x0b'
    >>> binascii.hexlify(byte)
    '0011323cc30b'

简单的TCP Echo服务器
----------------------

.. code-block:: python

    import socket

    class Server(object):
        def __init__(self,host,port):
            self._host = host
            self._port = port
        def __enter__(self):
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
            sock.bind((self._host, self._port))
            sock.listen(10)
            self._sock = sock
            return self._sock
        def __exit__(self, *exc_info):
            if exc_info[0]:
                import traceback
                traceback.print_exception(*exc_info)
            self._sock.close()

    if __name__ == '__main__':
        host = 'localhost'
        port = 5566
        with Server(host, port) as s:
            while True:
                conn, addr = s.accept()
                msg = conn.recv(1024)
                conn.send(msg)
                conn.close()

输出:

.. code-block:: consolel

    $ nc localhost 5566
    Hello World
    Hello World

简单的TCP Echo服务器通过IPv6
------------------------------------

.. code-block:: python

    import contextlib
    import socket

    host = "::1"
    port = 5566


    @contextlib.contextmanager
    def server(host, port):
        s = socket.socket(socket.AF_INET6, socket.SOCK_STREAM, 0)
        try:
            s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
            s.bind((host, port))
            s.listen(10)
            yield s
        finally:
            s.close()


    with server(host, port) as s:
        try:
            while True:
                conn, addr = s.accept()
                msg = conn.recv(1024)

                if msg:
                    conn.send(msg)

                conn.close()
        except KeyboardInterrupt:
            pass

输出:

.. code-block:: bash

    $ python3 ipv6.py &
    [1] 25752
    $ nc -6 ::1 5566
    Hello IPv6
    Hello IPv6

仅禁用IPv6
------------------

.. code-block:: python

    #!/usr/bin/env python3

    import contextlib
    import socket

    host = "::"
    port = 5566

    @contextlib.contextmanager
    def server(host: str, port: int):
        s = socket.socket(socket.AF_INET6, socket.SOCK_STREAM, 0)
        try:
            s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
            s.setsockopt(socket.IPPROTO_IPV6, socket.IPV6_V6ONLY, 0)
            s.bind((host, port))
            s.listen(10)
            yield s
        finally:
            s.close()


    with server(host, port) as s:
        try:
            while True:
                conn, addr = s.accept()
                remote = conn.getpeername()
                print(remote)
                msg = conn.recv(1024)

                if msg:
                    conn.send(msg)

                conn.close()
        except KeyboardInterrupt:
            pass

输出:

.. code-block:: bash

    $ python3 ipv6.py &
    [1] 23914
    $ nc -4 127.0.0.1 5566
    ('::ffff:127.0.0.1', 42604, 0, 0)
    Hello IPv4
    Hello IPv4
    $ nc -6 ::1 5566
    ('::1', 50882, 0, 0)
    Hello IPv6
    Hello IPv6
    $ nc -6 fe80::a00:27ff:fe9b:50ee%enp0s3 5566
    ('fe80::a00:27ff:fe9b:50ee%enp0s3', 42042, 0, 2)
    Hello IPv6
    Hello IPv6


通过SocketServer实现简单的TCP的Echo服务
---------------------------------------

.. code-block:: python

    >>> import SocketServer
    >>> bh = SocketServer.BaseRequestHandler
    >>> class handler(bh):
    ...   def handle(self):
    ...     data = self.request.recv(1024)
    ...     print(self.client_address)
    ...     self.request.sendall(data)
    ...
    >>> host = ('localhost', 5566)
    >>> s = SocketServer.TCPServer(
    ...   host, handler)
    >>> s.serve_forever()

输出:

.. code-block:: console

    $ nc localhost 5566
    Hello World
    Hello World


简单的TLS/SSL TCP Echo服务
--------------------------------

.. code-block:: python

    import socket
    import ssl

    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM, 0)
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    sock.bind(('localhost', 5566))
    sock.listen(10)

    sslctx = ssl.SSLContext(ssl.PROTOCOL_TLSv1)
    sslctx.load_cert_chain(certfile='./root-ca.crt',
                           keyfile='./root-ca.key')

    try:
        while True:
            conn, addr = sock.accept()
            sslconn = sslctx.wrap_socket(conn, server_side=True)
            msg = sslconn.recv(1024)
            if msg:
                sslconn.send(msg)
            sslconn.close()
    finally:
        sock.close()

输出:

.. code-block:: bash

    # console 1
    $ openssl genrsa -out root-ca.key 2048
    $ openssl req -x509 -new -nodes -key root-ca.key -days 365 -out root-ca.crt
    $ python3 ssl_tcp_server.py

    # console 2
    $ openssl s_client -connect localhost:5566
    ...
    Hello SSL
    Hello SSL
    read:errno=0


为TLS/SSL TCP Echo服务设置密码
---------------------------------------

.. code-block:: python

    import socket
    import json
    import ssl

    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM, 0)
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    sock.bind(('localhost', 5566))
    sock.listen(10)

    sslctx = ssl.SSLContext(ssl.PROTOCOL_SSLv23)
    sslctx.load_cert_chain(certfile='cert.pem',
                           keyfile='key.pem')
    # set ssl ciphers
    sslctx.set_ciphers('ECDH-ECDSA-AES128-GCM-SHA256')
    print(json.dumps(sslctx.get_ciphers(), indent=2))

    try:
        while True:
            conn, addr = sock.accept()
            sslconn = sslctx.wrap_socket(conn, server_side=True)
            msg = sslconn.recv(1024)
            if msg:
                sslconn.send(msg)
            sslconn.close()
    finally:
        sock.close()

output:

.. code-block:: bash

    $ openssl ecparam -out key.pem -genkey -name prime256v1
    $ openssl req -x509 -new -key key.pem -out cert.pem
    $ python3 tls.py&
    [2] 64565
    [
      {
        "id": 50380845,
        "name": "ECDH-ECDSA-AES128-GCM-SHA256",
        "protocol": "TLSv1/SSLv3",
        "description": "ECDH-ECDSA-AES128-GCM-SHA256 TLSv1.2 Kx=ECDH/ECDSA Au=ECDH Enc=AESGCM(128) Mac=AEAD",
        "strength_bits": 128,
        "alg_bits": 128
      }
    ]
    $ openssl s_client -connect localhost:5566 -cipher "ECDH-ECDSA-AES128-GCM-SHA256"
    ...
    ---
    Hello ECDH-ECDSA-AES128-GCM-SHA256
    Hello ECDH-ECDSA-AES128-GCM-SHA256
    read:errno=0


简单的UDP Echo服务
----------------------

.. code-block:: python

    import socket

    class UDPServer(object):
        def __init__(self, host, port):
            self._host = host
            self._port = port

        def __enter__(self):
            sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
            sock.bind((self._host, self._port))
            self._sock = sock
            return sock
       def __exit__(self, *exc_info):
            if exc_info[0]:
                import traceback
                traceback.print_exception(*exc_info)
            self._sock.close()

    if __name__ == '__main__':
        host = 'localhost'
        port = 5566
        with UDPServer(host, port) as s:
            while True:
                msg, addr = s.recvfrom(1024)
                s.sendto(msg, addr)

输出:

.. code-block:: console

    $ nc -u localhost 5566
    Hello World
    Hello World


通过SocketServer实现简单的UDP Echo服务
---------------------------------------

.. code-block:: python

    >>> import SocketServer
    >>> bh = SocketServer.BaseRequestHandler
    >>> class handler(bh):
    ...   def handle(self):
    ...     m,s = self.request
    ...     s.sendto(m,self.client_address)
    ...     print(self.client_address)
    ...
    >>> host = ('localhost', 5566)
    >>> s = SocketServer.UDPServer(
    ...   host, handler)
    >>> s.serve_forever()

输出:

.. code-block:: console

    $ nc -u localhost 5566
    Hello World
    Hello World


简单的UDP客户端 - 发送者
--------------------------

.. code-block:: python

    >>> import socket
    >>> import time
    >>> sock = socket.socket(
    ...   socket.AF_INET,
    ...   socket.SOCK_DGRAM)
    >>> host = ('localhost', 5566)
    >>> while True:
    ...   sock.sendto("Hello\n", host)
    ...   time.sleep(5)
    ...

output:

.. code-block:: console

    $ nc -lu localhost 5566
    Hello
    Hello

广播UDP包
---------------------

.. code-block:: python

    >>> import socket
    >>> import time
    >>> sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    >>> sock.bind(('', 0))
    >>> sock.setsockopt(socket.SOL_SOCKET, socket.SO_BROADCAST, 1)
    >>> while True:
    ...   m = '{0}\n'.format(time.time())
    ...   sock.sendto(m,('<broadcast>', 5566))
    ...   time.sleep(5)
    ...

输出:

.. code-block:: console

    $ nc -k -w 1 -ul 5566
    1431473025.72

简单的UNIX Domain Socket
-------------------------

.. code-block:: python

    import socket
    import contextlib
    import os

    @contextlib.contextmanager
    def DomainServer(addr):
        try:
            if os.path.exists(addr):
                os.unlink(addr)
            sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
            sock.bind(addr)
            sock.listen(10)
            yield sock
        finally:
            sock.close()
            if os.path.exists(addr):
                os.unlink(addr)

    addr = "./domain.sock"
    with DomainServer(addr) as sock:
        while True:
            conn, _ = sock.accept()
            msg = conn.recv(1024)
            conn.send(msg)
            conn.close()

输出:

.. code-block:: console

    $ nc -U ./domain.sock
    Hello
    Hello


简单的双通道通信
---------------------------------------

.. code-block:: python

    import os
    import socket

    child, parent = socket.socketpair()
    pid = os.fork()
    try:

        if pid == 0:
            print('chlid pid: {}'.format(os.getpid()))

            child.send(b'Hello Parent')
            msg = child.recv(1024)
            print('p[{}] ---> c[{}]: {}'.format(
                os.getppid(), os.getpid(), msg))
        else:
            print('parent pid: {}'.format(os.getpid()))

            # simple echo server (parent)
            msg = parent.recv(1024)
            print('c[{}] ---> p[{}]: {}'.format(
                    pid, os.getpid(), msg))
            parent.send(msg)

    except KeyboardInterrupt:
        pass
    finally:
        child.close()
        parent.close()

输出:

.. code-block:: bash

    $ python3 socketpair_demo.py
    parent pid: 9497
    chlid pid: 9498
    c[9498] ---> p[9497]: b'Hello Parent'
    p[9497] ---> c[9498]: b'Hello Parent'


简单的异步TCP服务 - 线程
---------------------------------------

.. code-block:: python

    >>> from threading import Thread
    >>> import socket
    >>> def work(conn):
    ...   while True:
    ...     msg = conn.recv(1024)
    ...     conn.send(msg)
    ...
    >>> sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    >>> sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    >>> sock.bind(('localhost', 5566))
    >>> sock.listen(5)
    >>> while True:
    ...   conn,addr = sock.accept()
    ...   t=Thread(target=work,args=(conn,))
    ...   t.daemon=True
    ...   t.start()
    ...

输出: (bash 1)

.. code-block:: console

    $ nc localhost 5566
    Hello
    Hello

输出: (bash 2)

.. code-block:: console

    $ nc localhost 5566
    Ker Ker
    Ker Ker

简单的异步TCP服务 - select
---------------------------------------

.. code-block:: python

    from select import select
    import socket

    host = ('localhost', 5566)
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    sock.bind(host)
    sock.listen(5)
    rl = [sock]
    wl = []
    ml = {}
    try:
        while True:
            r, w, _ = select(rl, wl, [])
            # process ready to ready
            for _ in r:
                if _ == sock:
                    conn, addr = sock.accept()
                    rl.append(conn)
                else:
                    msg = _.recv(1024)
                    ml[_.fileno()] = msg
                    wl.append(_)
            # process ready to write
            for _ in w:
                msg = ml[_.fileno()]
                _.send(msg)
                wl.remove(_)
                del ml[_.fileno()]
    except:
        sock.close()

输出: (bash 1)

.. code-block:: console

    $ nc localhost 5566
    Hello
    Hello

输出: (bash 2)

.. code-block:: console

    $ nc localhost 5566
    Ker Ker
    Ker Ker


简单的异步TCP服务 - poll
--------------------------------------

.. code-block:: python

    from __future__ import print_function, unicode_literals

    import socket
    import select
    import contextlib

    host = 'localhost'
    port = 5566

    con = {}
    req = {}
    resp = {}

    @contextlib.contextmanager
    def Server(host,port):
        try:
            s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
            s.setblocking(False)
            s.bind((host,port))
            s.listen(10)
            yield s
        except socket.error:
            print("Get socket error")
            raise
        finally:
            if s: s.close()


    @contextlib.contextmanager
    def Poll():
        try:
            e = select.poll()
            yield e
        finally:
            for fd, c in con.items():
                e.unregister(fd)
                c.close()


    def accept(server, poll):
        conn, addr = server.accept()
        conn.setblocking(False)
        fd = conn.fileno()
        poll.register(fd, select.POLLIN)
        req[fd] = conn
        con[fd] = conn


    def recv(fd, poll):
        if fd not in req:
            return

        conn = req[fd]
        msg = conn.recv(1024)
        if msg:
            resp[fd] = msg
            poll.modify(fd, select.POLLOUT)
        else:
            conn.close()
            del con[fd]

        del req[fd]


    def send(fd, poll):
        if fd not in resp:
            return

        conn = con[fd]
        msg = resp[fd]
        b = 0
        total = len(msg)
        while total > b:
            l = conn.send(msg)
            msg = msg[l:]
            b += l

        del resp[fd]
        req[fd] = conn
        poll.modify(fd, select.POLLIN)

    try:
        with Server(host, port) as server, Poll() as poll:

            poll.register(server.fileno())

            while True:
                events = poll.poll(1)
                for fd, e in events:
                    if fd == server.fileno():
                        accept(server, poll)
                    elif e & (select.POLLIN | select.POLLPRI):
                        recv(fd, poll)
                    elif e & select.POLLOUT:
                        send(fd, poll)
    except KeyboardInterrupt:
        pass

输出: (bash 1)

.. code-block:: console

    $ python3 poll.py &
    [1] 3036
    $ nc localhost 5566
    Hello poll
    Hello poll
    Hello Python Socket Programming
    Hello Python Socket Programming

输出: (bash 2)

.. code-block:: console

    $ nc localhost 5566
    Hello Python
    Hello Python
    Hello Awesome Python
    Hello Awesome Python


简单的异步TCP服务 - epoll
---------------------------------------

.. code-block:: python

    from __future__ import print_function, unicode_literals

    import socket
    import select
    import contextlib


    host = 'localhost'
    port = 5566

    con = {}
    req = {}
    resp = {}

    @contextlib.contextmanager
    def Server(host,port):
        try:
            s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
            s.setblocking(False)
            s.bind((host,port))
            s.listen(10)
            yield s
        except socket.error:
            print("Get socket error")
            raise
        finally:
            if s: s.close()


    @contextlib.contextmanager
    def Epoll():
        try:
            e = select.epoll()
            yield e
        finally:
            for fd in con: e.unregister(fd)
            e.close()


    def accept(server, epoll):
        conn, addr = server.accept()
        conn.setblocking(0)
        fd = conn.fileno()
        epoll.register(fd, select.EPOLLIN)
        req[fd] = conn
        con[fd] = conn


    def recv(fd, epoll):
        if fd not in req:
            return

        conn = req[fd]
        msg = conn.recv(1024)
        if msg:
            resp[fd] = msg
            epoll.modify(fd, select.EPOLLOUT)
        else:
            conn.close()
            del con[fd]

        del req[fd]


    def send(fd, epoll):
        if fd not in resp:
            return

        conn = con[fd]
        msg = resp[fd]
        b = 0
        total = len(msg)
        while total > b:
            l = conn.send(msg)
            msg = msg[l:]
            b += l

        del resp[fd]
        req[fd] = conn
        epoll.modify(fd, select.EPOLLIN)


    try:
        with Server(host, port) as server, Epoll() as epoll:

            epoll.register(server.fileno())

            while True:
                events = epoll.poll(1)
                for fd, e in events:
                    if fd == server.fileno():
                        accept(server, epoll)
                    elif e & select.EPOLLIN:
                        recv(fd, epoll)
                    elif e & select.EPOLLOUT:
                        send(fd, epoll)
    except KeyboardInterrupt:
        pass


输出: (bash 1)

.. code-block:: console

    $ python3 epoll.py &
    [1] 3036
    $ nc localhost 5566
    Hello epoll
    Hello epoll
    Hello Python Socket Programming
    Hello Python Socket Programming

输出: (bash 2)

.. code-block:: console

    $ nc localhost 5566
    Hello Python
    Hello Python
    Hello Awesome Python
    Hello Awesome Python


简单的异步TCP服务 - kqueue
----------------------------------------

.. code-block:: python

    from __future__ import print_function, unicode_literals

    import socket
    import select
    import contextlib

    if not hasattr(select, 'kqueue'):
        print("Not support kqueue")
        exit(1)


    host = 'localhost'
    port = 5566

    con = {}
    req = {}
    resp = {}

    @contextlib.contextmanager
    def Server(host, port):
        try:
            s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
            s.setblocking(False)
            s.bind((host, port))
            s.listen(10)
            yield s
        except socket.error:
            print("Get socket error")
            raise
        finally:
            if s: s.close()


    @contextlib.contextmanager
    def Kqueue():
        try:
            kq = select.kqueue()
            yield kq
        finally:
            kq.close()
            for fd, c in con.items(): c.close()


    def accept(server, kq):
        conn, addr = server.accept()
        conn.setblocking(False)
        fd = conn.fileno()
        ke = select.kevent(conn.fileno(),
                           select.KQ_FILTER_READ,
                           select.KQ_EV_ADD)
        kq.control([ke], 0)
        req[fd] = conn
        con[fd] = conn


    def recv(fd, kq):
        if fd not in req:
            return

        conn = req[fd]
        msg = conn.recv(1024)
        if msg:
            resp[fd] = msg
            # remove read event
            ke = select.kevent(fd,
                               select.KQ_FILTER_READ,
                               select.KQ_EV_DELETE)
            kq.control([ke], 0)
            # add write event
            ke = select.kevent(fd,
                               select.KQ_FILTER_WRITE,
                               select.KQ_EV_ADD)
            kq.control([ke], 0)
            req[fd] = conn
            con[fd] = conn
        else:
            conn.close()
            del con[fd]

        del req[fd]


    def send(fd, kq):
        if fd not in resp:
            return

        conn = con[fd]
        msg = resp[fd]
        b = 0
        total = len(msg)
        while total > b:
            l = conn.send(msg)
            msg = msg[l:]
            b += l

        del resp[fd]
        req[fd] = conn
        # remove write event
        ke = select.kevent(fd,
                           select.KQ_FILTER_WRITE,
                           select.KQ_EV_DELETE)
        kq.control([ke], 0)
        # add read event
        ke = select.kevent(fd,
                           select.KQ_FILTER_READ,
                           select.KQ_EV_ADD)
        kq.control([ke], 0)


    try:
        with Server(host, port) as server, Kqueue() as kq:

            max_events = 1024
            timeout = 1

            ke = select.kevent(server.fileno(),
                               select.KQ_FILTER_READ,
                               select.KQ_EV_ADD)

            kq.control([ke], 0)
            while True:
                events = kq.control(None, max_events, timeout)
                for e in events:
                    fd = e.ident
                    if fd == server.fileno():
                        accept(server, kq)
                    elif e.filter == select.KQ_FILTER_READ:
                        recv(fd, kq)
                    elif e.filter == select.KQ_FILTER_WRITE:
                        send(fd, kq)
    except KeyboardInterrupt:
        pass

输出: (bash 1)

.. code-block:: console

    $ python3 kqueue.py &
    [1] 3036
    $ nc localhost 5566
    Hello kqueue
    Hello kqueue
    Hello Python Socket Programming
    Hello Python Socket Programming

输出: (bash 2)

.. code-block:: console

    $ nc localhost 5566
    Hello Python
    Hello Python
    Hello Awesome Python
    Hello Awesome Python


高阶API - selectors
--------------------------

.. code-block:: python

    # Pyton3.4+ only
    # Reference: selectors
    import selectors
    import socket
    import contextlib

    @contextlib.contextmanager
    def Server(host,port):
       try:
            s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
            s.bind((host,port))
            s.listen(10)
            sel = selectors.DefaultSelector()
            yield s, sel
        except socket.error:
            print("Get socket error")
            raise
        finally:
            if s:
                s.close()

    def read_handler(conn, sel):
        msg = conn.recv(1024)
        if msg:
            conn.send(msg)
        else:
            sel.unregister(conn)
            conn.close()

    def accept_handler(s, sel):
        conn, _ = s.accept()
        sel.register(conn, selectors.EVENT_READ, read_handler)

    host = 'localhost'
    port = 5566
    with Server(host, port) as (s, sel):
        sel.register(s, selectors.EVENT_READ, accept_handler)
        while True:
            events = sel.select()
            for sel_key, m in events:
                handler = sel_key.data
                handler(sel_key.fileobj, sel)

输出: (bash 1)

.. code-block:: console

    $ nc localhost 5566
    Hello
    Hello

输出: (bash 2)

.. code-block:: console

    $ nc localhost 5566
    Hi
    Hi


简单不阻塞socket通过selectors
--------------------------------------------------

.. code-block:: python

    import socket
    import selectors
    import contextlib
    import ssl

    from functools import partial

    sslctx = ssl.create_default_context(ssl.Purpose.CLIENT_AUTH)
    sslctx.load_cert_chain(certfile="cert.pem", keyfile="key.pem")

    @contextlib.contextmanager
    def Server(host,port):
        try:
            s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
            s.bind((host,port))
            s.listen(10)
            sel = selectors.DefaultSelector()
            yield s, sel
        except socket.error:
            print("Get socket error")
            raise
        finally:
            if s: s.close()
            if sel: sel.close()


    def accept(s, sel):
        conn, _ = s.accept()
        sslconn = sslctx.wrap_socket(conn,
                                     server_side=True,
                                     do_handshake_on_connect=False)
        sel.register(sslconn, selectors.EVENT_READ, do_handshake)


    def do_handshake(sslconn, sel):
        sslconn.do_handshake()
        sel.modify(sslconn, selectors.EVENT_READ, read)


    def read(sslconn, sel):
        msg = sslconn.recv(1024)
        if msg:
            sel.modify(sslconn,
                       selectors.EVENT_WRITE,
                       partial(write, msg=msg))
        else:
            sel.unregister(sslconn)
            sslconn.close()


    def write(sslconn, sel, msg=None):
        if msg:
            sslconn.send(msg)
        sel.modify(sslconn, selectors.EVENT_READ, read)


    host = 'localhost'
    port = 5566
    try:
        with Server(host, port) as (s, sel):
            sel.register(s, selectors.EVENT_READ, accept)
            while True:
                events = sel.select()
                for sel_key, m in events:
                    handler = sel_key.data
                    handler(sel_key.fileobj, sel)
    except KeyboardInterrupt:
        pass


输出:

.. code-block:: console

    # console 1
    $ openssl genrsa -out key.pem 2048
    $ openssl req -x509 -new -nodes -key key.pem -days 365 -out cert.pem
    $ python3 ssl_tcp_server.py &
    $ openssl s_client -connect localhost:5566
    ...
    ---
    Hello TLS
    Hello TLS

    # console 2
    $ openssl s_client -connect localhost:5566
    ...
    ---
    Hello SSL
    Hello SSL


"socketpair" - 相似 PIPE
------------------------------

.. code-block:: python

    import socket
    import os
    import time

    c_s, p_s = socket.socketpair()
    try:
        pid = os.fork()
    except OSError:
        print("Fork Error")
        raise

    if pid:
        # parent process
        c_s.close()
        while True:
            p_s.sendall("Hi! Child!")
            msg = p_s.recv(1024)
            print(msg)
            time.sleep(3)
        os.wait()
    else:
        # child process
        p_s.close()
        while True:
            msg = c_s.recv(1024)
            print(msg)
            c_s.sendall("Hi! Parent!")

输出:

.. code-block:: console

    $ python ex.py
    Hi! Child!
    Hi! Parent!
    Hi! Child!
    Hi! Parent!
    ...

使用sendfile去copy
------------------------

.. code-block:: python

    # need python 3.3 or above
    from __future__ import print_function, unicode_literals

    import os
    import sys

    if len(sys.argv) != 3:
        print("Usage: cmd src dst")
        exit(1)

    src = sys.argv[1]
    dst = sys.argv[2]

    with open(src, 'r') as s, open(dst, 'w') as d:
        st = os.fstat(s.fileno())

        offset = 0
        count = 4096
        s_len = st.st_size

        sfd = s.fileno()
        dfd = d.fileno()

        while s_len > 0:
            ret = os.sendfile(dfd, sfd, offset, count)
            offset += ret
            s_len -= ret

output:

.. code-block:: console

    $ dd if=/dev/urandom of=dd.in bs=1M count=1024
    1024+0 records in
    1024+0 records out
    1073741824 bytes (1.1 GB, 1.0 GiB) copied, 108.02 s, 9.9 MB/s
    $ python3 sendfile.py dd.in dd.out
    $ md5sum dd.in
    e79afdd6aba71b7174142c0bbc289674  dd.in
    $ md5sum dd.out
    e79afdd6aba71b7174142c0bbc289674  dd.out


通过sendfile发送一个文件
---------------------------------

.. code-block:: python

    # need python 3.5 or above
    from __future__ import print_function, unicode_literals

    import os
    import sys
    import time
    import socket
    import contextlib

    @contextlib.contextmanager
    def server(host, port):
        try:
            s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
            s.bind((host, port))
            s.listen(10)
            yield s
        finally:
            s.close()


    @contextlib.contextmanager
    def client(host, port):
        try:
            c = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            c.connect((host, port))
            yield c
        finally:
            c.close()


    def do_sendfile(fout, fin, count, fin_len):
        l = fin_len
        offset = 0
        while l > 0:
            ret = fout.sendfile(fin, offset, count)
            offset += ret
            l -= ret


    def do_recv(fout, fin):
        while True:
            data = fin.recv(4096)

            if not data: break

            fout.write(data)


    host = 'localhost'
    port = 5566

    if len(sys.argv) != 3:
        print("usage: cmd src dst")
        exit(1)

    src = sys.argv[1]
    dst = sys.argv[2]
    offset = 0

    pid = os.fork()

    if pid ==  0:
        # client
        time.sleep(3)
        with client(host, port) as c, open(src, 'rb') as f:
            fd = f.fileno()
            st = os.fstat(fd)
            count = 4096

            flen = st.st_size
            do_sendfile(c, f, count, flen)

    else:
        # server
        with server(host, port) as s, open(dst, 'wb') as f:
            conn, addr = s.accept()
            do_recv(f, conn)

输出:

.. code-block:: console

    $ dd if=/dev/urandom of=dd.in bs=1M count=512
    512+0 records in
    512+0 records out
    536870912 bytes (537 MB, 512 MiB) copied, 3.17787 s, 169 MB/s
    $ python3 sendfile.py dd.in dd.out
    $ md5sum dd.in
    eadfd96c85976b1f46385e89dfd9c4a8  dd.in
    $ md5sum dd.out
    eadfd96c85976b1f46385e89dfd9c4a8  dd.out


Linux内核加密API - AF_ALG
---------------------------------

.. code-block:: python

    # need python 3.6 or above & Linux >=2.6.38
    import socket
    import hashlib
    import contextlib

    @contextlib.contextmanager
    def create_alg(typ, name):
        s = socket.socket(socket.AF_ALG, socket.SOCK_SEQPACKET, 0)
        try:
            s.bind((typ, name))
            yield s
        finally:
            s.close()

    msg = b'Python is awesome!'

    with create_alg('hash', 'sha256') as algo:
        op, _ = algo.accept()
        with op:
            op.sendall(msg)
            data = op.recv(512)
            print(data.hex())

            # check data
            h = hashlib.sha256(msg).digest()
            if h != data:
                raise Exception(f"sha256({h}) != af_alg({data})")

输出:

.. code-block:: console

    $ python3 af_alg.py
    9d50bcac2d5e33f936ec2db7dc7b6579cba8e1b099d77c31d8564df46f66bdf5


AES-CBC加密/解密通过AF_ALG
-----------------------------------

.. code-block:: python

    # need python 3.6 or above & Linux >=4.3
    import contextlib
    import socket
    import os

    BS = 16  # Bytes
    pad = lambda s: s + (BS - len(s) % BS) * \
                     chr(BS - len(s) % BS).encode('utf-8')

    upad = lambda s : s[0:-s[-1]]


    @contextlib.contextmanager
    def create_alg(typ, name):
        s = socket.socket(socket.AF_ALG, socket.SOCK_SEQPACKET, 0)
        try:
            s.bind((typ, name))
            yield s
        finally:
            s.close()


    def encrypt(plaintext, key, iv):
        ciphertext = None
        with create_alg('skcipher', 'cbc(aes)') as algo:
            algo.setsockopt(socket.SOL_ALG, socket.ALG_SET_KEY, key)
            op, _ = algo.accept()
            with op:
                plaintext = pad(plaintext)
                op.sendmsg_afalg([plaintext],
                                 op=socket.ALG_OP_ENCRYPT,
                                 iv=iv)
                ciphertext = op.recv(len(plaintext))

        return ciphertext


    def decrypt(ciphertext, key, iv):
        plaintext = None
        with create_alg('skcipher', 'cbc(aes)') as algo:
            algo.setsockopt(socket.SOL_ALG, socket.ALG_SET_KEY, key)
            op, _ = algo.accept()
            with op:
                op.sendmsg_afalg([ciphertext],
                                 op=socket.ALG_OP_DECRYPT,
                                 iv=iv)
                plaintext = op.recv(len(ciphertext))

        return upad(plaintext)


    key = os.urandom(32)
    iv  = os.urandom(16)

    plaintext = b"Demo AF_ALG"
    ciphertext = encrypt(plaintext, key, iv)
    plaintext = decrypt(ciphertext, key, iv)

    print(ciphertext.hex())
    print(plaintext)

输出:

.. code-block:: console

    $ python3 aes_cbc.py
    01910e4bd6932674dba9bebd4fdf6cf2
    b'Demo AF_ALG'


AES-GCM加密/解密通过AF_ALG
-----------------------------------

.. code-block:: python

    # need python 3.6 or above & Linux >=4.9
    import contextlib
    import socket
    import os

    @contextlib.contextmanager
    def create_alg(typ, name):
        s = socket.socket(socket.AF_ALG, socket.SOCK_SEQPACKET, 0)
        try:
            s.bind((typ, name))
            yield s
        finally:
            s.close()


    def encrypt(key, iv, assoc, taglen, plaintext):
        """ doing aes-gcm encrypt

        :param key: the aes symmetric key
        :param iv: initial vector
        :param assoc: associated data (integrity protection)
        :param taglen: authenticator tag len
        :param plaintext: plain text data
        """

        assoclen = len(assoc)
        ciphertext = None
        tag = None

        with create_alg('aead', 'gcm(aes)') as algo:
            algo.setsockopt(socket.SOL_ALG,
                            socket.ALG_SET_KEY, key)
            algo.setsockopt(socket.SOL_ALG,
                            socket.ALG_SET_AEAD_AUTHSIZE,
                            None,
                            assoclen)

            op, _ = algo.accept()
            with op:
                msg = assoc + plaintext
                op.sendmsg_afalg([msg],
                                 op=socket.ALG_OP_ENCRYPT,
                                 iv=iv,
                                 assoclen=assoclen)

                res = op.recv(assoclen + len(plaintext) + taglen)
                ciphertext = res[assoclen:-taglen]
                tag = res[-taglen:]

        return ciphertext, tag


    def decrypt(key, iv, assoc, tag, ciphertext):
        """ doing aes-gcm decrypt

        :param key: the AES symmetric key
        :param iv: initial vector
        :param assoc: associated data (integrity protection)
        :param tag: the GCM authenticator tag
        :param ciphertext: cipher text data
        """
        plaintext = None
        assoclen = len(assoc)

        with create_alg('aead', 'gcm(aes)') as algo:
            algo.setsockopt(socket.SOL_ALG,
                            socket.ALG_SET_KEY, key)
            algo.setsockopt(socket.SOL_ALG,
                            socket.ALG_SET_AEAD_AUTHSIZE,
                            None,
                            assoclen)
            op, _ = algo.accept()
            with op:
                msg = assoc + ciphertext + tag
                op.sendmsg_afalg([msg],
                                 op=socket.ALG_OP_DECRYPT, iv=iv,
                                 assoclen=assoclen)

                taglen = len(tag)
                res = op.recv(len(msg) - taglen)
                plaintext = res[assoclen:]

        return plaintext

    key = os.urandom(16)
    iv  = os.urandom(12)
    assoc = os.urandom(16)

    plaintext = b"Hello AES-GCM"
    ciphertext, tag = encrypt(key, iv, assoc, 16, plaintext)
    plaintext = decrypt(key, iv, assoc, tag, ciphertext)

    print(ciphertext.hex())
    print(plaintext)

输出:

.. code-block:: console

	$ python3 aes_gcm.py
	2e27b67234e01bcb0ab6b451f4f870ce
	b'Hello AES-GCM'


AES-GCM使用sendfile加密/解密文件
-------------------------------------------

.. code-block:: python

    # need python 3.6 or above & Linux >=4.9
    import contextlib
    import socket
    import sys
    import os

    @contextlib.contextmanager
    def create_alg(typ, name):
        s = socket.socket(socket.AF_ALG, socket.SOCK_SEQPACKET, 0)
        try:
            s.bind((typ, name))
            yield s
        finally:
            s.close()


    def encrypt(key, iv, assoc, taglen, pfile):
        assoclen = len(assoc)
        ciphertext = None
        tag = None

        pfd = pfile.fileno()
        offset = 0
        st = os.fstat(pfd)
        totalbytes = st.st_size

        with create_alg('aead', 'gcm(aes)') as algo:
            algo.setsockopt(socket.SOL_ALG,
                            socket.ALG_SET_KEY, key)
            algo.setsockopt(socket.SOL_ALG,
                            socket.ALG_SET_AEAD_AUTHSIZE,
                            None,
                            assoclen)

            op, _ = algo.accept()
            with op:
                op.sendmsg_afalg(op=socket.ALG_OP_ENCRYPT,
                                 iv=iv,
                                 assoclen=assoclen,
                                 flags=socket.MSG_MORE)

                op.sendall(assoc, socket.MSG_MORE)

                # using sendfile to encrypt file data
                os.sendfile(op.fileno(), pfd, offset, totalbytes)

                res = op.recv(assoclen + totalbytes + taglen)
                ciphertext = res[assoclen:-taglen]
                tag = res[-taglen:]

        return ciphertext, tag


    def decrypt(key, iv, assoc, tag, ciphertext):
        plaintext = None
        assoclen = len(assoc)

        with create_alg('aead', 'gcm(aes)') as algo:
            algo.setsockopt(socket.SOL_ALG,
                            socket.ALG_SET_KEY, key)
            algo.setsockopt(socket.SOL_ALG,
                            socket.ALG_SET_AEAD_AUTHSIZE,
                            None,
                            assoclen)
            op, _ = algo.accept()
            with op:
                msg = assoc + ciphertext + tag
                op.sendmsg_afalg([msg],
                                 op=socket.ALG_OP_DECRYPT, iv=iv,
                                 assoclen=assoclen)

                taglen = len(tag)
                res = op.recv(len(msg) - taglen)
                plaintext = res[assoclen:]

        return plaintext

    key = os.urandom(16)
    iv  = os.urandom(12)
    assoc = os.urandom(16)

    if len(sys.argv) != 2:
        print("usage: cmd plain")
        exit(1)

    plain = sys.argv[1]

    with open(plain, 'r') as pf:
        ciphertext, tag = encrypt(key, iv, assoc, 16, pf)
        plaintext = decrypt(key, iv, assoc, tag, ciphertext)

        print(ciphertext.hex())
        print(plaintext)


输出:

.. code-block:: console

    $ echo "Test AES-GCM with sendfile" > plain.txt
    $ python3 aes_gcm.py plain.txt
    b3800044520ed07fa7f20b29c2695bae9ab596065359db4f009dd6
    b'Test AES-GCM with sendfile\n'


比较AF_ALG和cryptography的性能
--------------------------------------------------

.. code-block:: python

    # need python 3.6 or above & Linux >=4.9
    import contextlib
    import socket
    import time
    import os

    from cryptography.hazmat.primitives.ciphers.aead import AESGCM

    @contextlib.contextmanager
    def create_alg(typ, name):
        s = socket.socket(socket.AF_ALG, socket.SOCK_SEQPACKET, 0)
        try:
            s.bind((typ, name))
            yield s
        finally:
            s.close()


    def encrypt(key, iv, assoc, taglen, op, pfile, psize):
        assoclen = len(assoc)
        ciphertext = None
        tag = None
        offset = 0

        pfd = pfile.fileno()
        totalbytes = psize

        op.sendmsg_afalg(op=socket.ALG_OP_ENCRYPT,
                         iv=iv,
                         assoclen=assoclen,
                         flags=socket.MSG_MORE)

        op.sendall(assoc, socket.MSG_MORE)

        # using sendfile to encrypt file data
        os.sendfile(op.fileno(), pfd, offset, totalbytes)

        res = op.recv(assoclen + totalbytes + taglen)
        ciphertext = res[assoclen:-taglen]
        tag = res[-taglen:]

        return ciphertext, tag


    def decrypt(key, iv, assoc, tag, op, ciphertext):
        plaintext = None
        assoclen = len(assoc)

        msg = assoc + ciphertext + tag
        op.sendmsg_afalg([msg],
                         op=socket.ALG_OP_DECRYPT, iv=iv,
                         assoclen=assoclen)

        taglen = len(tag)
        res = op.recv(len(msg) - taglen)
        plaintext = res[assoclen:]

        return plaintext


    key = os.urandom(16)
    iv  = os.urandom(12)
    assoc = os.urandom(16)
    assoclen = len(assoc)

    count = 1000000
    plain = "tmp.rand"

    # crate a tmp file
    with open(plain, 'wb') as f:
        f.write(os.urandom(4096))
        f.flush()


    # profile AF_ALG with sendfile (zero-copy)
    with open(plain, 'rb') as pf,\
         create_alg('aead', 'gcm(aes)') as enc_algo,\
         create_alg('aead', 'gcm(aes)') as dec_algo:

        enc_algo.setsockopt(socket.SOL_ALG,
                            socket.ALG_SET_KEY, key)
        enc_algo.setsockopt(socket.SOL_ALG,
                            socket.ALG_SET_AEAD_AUTHSIZE,
                            None,
                            assoclen)

        dec_algo.setsockopt(socket.SOL_ALG,
                            socket.ALG_SET_KEY, key)
        dec_algo.setsockopt(socket.SOL_ALG,
                            socket.ALG_SET_AEAD_AUTHSIZE,
                            None,
                            assoclen)

        enc_op, _ = enc_algo.accept()
        dec_op, _ = dec_algo.accept()

        st = os.fstat(pf.fileno())
        psize = st.st_size

        with enc_op, dec_op:

            s = time.time()

            for _ in range(count):
                ciphertext, tag = encrypt(key, iv, assoc, 16, enc_op, pf, psize)
                plaintext = decrypt(key, iv, assoc, tag, dec_op, ciphertext)

            cost = time.time() - s

            print(f"total cost time: {cost}. [AF_ALG]")


    # profile cryptography (no zero-copy)
    with open(plain, 'rb') as pf:

        aesgcm = AESGCM(key)

        s = time.time()

        for _ in range(count):
            pf.seek(0, 0)
            plaintext = pf.read()
            ciphertext = aesgcm.encrypt(iv, plaintext, assoc)
            plaintext = aesgcm.decrypt(iv, ciphertext, assoc)

        cost = time.time() - s

        print(f"total cost time: {cost}. [cryptography]")

    # clean up
    os.remove(plain)

输出:

.. code-block:: console

    $ python3 aes-gcm.py
    total cost time: 15.317010641098022. [AF_ALG]
    total cost time: 50.256704807281494. [cryptography]


IP数据包嗅探器
------------------

.. code-block:: python

    from ctypes import *
    import socket
    import struct

    # ref: IP protocol numbers
    PROTO_MAP = {
            1 : "ICMP",
            2 : "IGMP",
            6 : "TCP",
            17: "UDP",
            27: "RDP"}

    class IP(Structure):
        ''' IP header Structure

        In linux api, it define as below:

        strcut ip {
            u_char         ip_hl:4; /* header_len */
            u_char         ip_v:4;  /* version */
            u_char         ip_tos;  /* type of service */
            short          ip_len;  /* total len */
            u_short        ip_id;   /* identification */
            short          ip_off;  /* offset field */
            u_char         ip_ttl;  /* time to live */
            u_char         ip_p;    /* protocol */
            u_short        ip_sum;  /* checksum */
            struct in_addr ip_src;  /* source */
            struct in_addr ip_dst;  /* destination */
        };
        '''
        _fields_ = [("ip_hl" , c_ubyte, 4), # 4 bit
                    ("ip_v"  , c_ubyte, 4), # 1 byte
                    ("ip_tos", c_uint8),    # 2 byte
                    ("ip_len", c_uint16),   # 4 byte
                    ("ip_id" , c_uint16),   # 6 byte
                    ("ip_off", c_uint16),   # 8 byte
                    ("ip_ttl", c_uint8),    # 9 byte
                    ("ip_p"  , c_uint8),    # 10 byte
                    ("ip_sum", c_uint16),   # 12 byte
                    ("ip_src", c_uint32),   # 16 byte
                    ("ip_dst", c_uint32)]   # 20 byte

        def __new__(cls, buf=None):
            return cls.from_buffer_copy(buf)
        def __init__(self, buf=None):
            src = struct.pack("<L", self.ip_src)
            self.src = socket.inet_ntoa(src)
            dst = struct.pack("<L", self.ip_dst)
            self.dst = socket.inet_ntoa(dst)
            try:
                self.proto = PROTO_MAP[self.ip_p]
            except KeyError:
                print("{} Not in map".format(self.ip_p))
                raise

    host = '0.0.0.0'
    s = socket.socket(socket.AF_INET,
                      socket.SOCK_RAW,
                      socket.IPPROTO_ICMP)
    s.setsockopt(socket.IPPROTO_IP, socket.IP_HDRINCL, 1)
    s.bind((host, 0))

    print("Sniffer start...")
    try:
        while True:
            buf = s.recvfrom(65535)[0]
            ip_header = IP(buf[:20])
            print('{0}: {1} -> {2}'.format(ip_header.proto,
                                           ip_header.src,
                                           ip_header.dst))
    except KeyboardInterrupt:
        s.close()

输出: (bash 1)

.. code-block:: console

    python sniffer.py
    Sniffer start...
    ICMP: 127.0.0.1 -> 127.0.0.1
    ICMP: 127.0.0.1 -> 127.0.0.1
    ICMP: 127.0.0.1 -> 127.0.0.1

输出: (bash 2)

.. code-block:: console

    $ ping -c 3 localhost
    PING localhost (127.0.0.1): 56 data bytes
    64 bytes from 127.0.0.1: icmp_seq=0 ttl=64 time=0.063 ms
    64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.087 ms
    64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.159 ms

    --- localhost ping statistics ---
    3 packets transmitted, 3 packets received, 0.0% packet loss
    round-trip min/avg/max/stddev = 0.063/0.103/0.159/0.041 ms


TCP数据包嗅探器
------------------

.. code-block:: python

    #!/usr/bin/env python3.6
    """
    Based on RFC-793, the following figure shows the TCP header format:

    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |          Source Port          |       Destination Port        |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                        Sequence Number                        |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                    Acknowledgment Number                      |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |  Data |           |U|A|P|R|S|F|                               |
    | Offset| Reserved  |R|C|S|S|Y|I|            Window             |
    |       |           |G|K|H|T|N|N|                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |           Checksum            |         Urgent Pointer        |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                    Options                    |    Padding    |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                             data                              |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

    In linux api (uapi/linux/tcp.h), it defines the TCP header:

    struct tcphdr {
        __be16  source;
        __be16  dest;
        __be32  seq;
        __be32  ack_seq;
    #if defined(__LITTLE_ENDIAN_BITFIELD)
        __u16   res1:4,
                doff:4,
                fin:1,
                syn:1,
                rst:1,
                psh:1,
                ack:1,
                urg:1,
                ece:1,
                cwr:1;
    #elif defined(__BIG_ENDIAN_BITFIELD)
        __u16   doff:4,
                res1:4,
                cwr:1,
                ece:1,
                urg:1,
                ack:1,
                psh:1,
                rst:1,
                syn:1,
                fin:1;
    #else
    #error      "Adjust your <asm/byteorder.h> defines"
    #endif
        __be16  window;
        __sum16 check;
        __be16  urg_ptr;
    };
    """
    import sys
    import socket
    import platform

    from struct import unpack
    from contextlib import contextmanager

    un = platform.system()
    if un != "Linux":
        print(f"{un} is not supported!")
        sys.exit(1)

    @contextmanager
    def create_socket():
        ''' Create a TCP raw socket '''
        s = socket.socket(socket.AF_INET,
                          socket.SOCK_RAW,
                          socket.IPPROTO_TCP)
        try:
            yield s
        finally:
            s.close()


    try:
        with create_socket() as s:
            while True:
                pkt, addr = s.recvfrom(65535)

                # the first 20 bytes are ip header
                iphdr = unpack('!BBHHHBBH4s4s', pkt[0:20])
                iplen = (iphdr[0] & 0xf) * 4

                # the next 20 bytes are tcp header
                tcphdr = unpack('!HHLLBBHHH', pkt[iplen:iplen+20])
                source = tcphdr[0]
                dest = tcphdr[1]
                seq = tcphdr[2]
                ack_seq = tcphdr[3]
                dr = tcphdr[4]
                flags = tcphdr[5]
                window = tcphdr[6]
                check = tcphdr[7]
                urg_ptr = tcphdr[8]

                doff = dr >> 4
                fin = flags & 0x01
                syn = flags & 0x02
                rst = flags & 0x04
                psh = flags & 0x08
                ack = flags & 0x10
                urg = flags & 0x20
                ece = flags & 0x40
                cwr = flags & 0x80

                tcplen = (doff) * 4
                h_size = iplen + tcplen

                #get data from the packet
                data = pkt[h_size:]

                if not data:
                    continue

                print("------------ TCP_HEADER --------------")
                print(f"Source Port:           {source}")
                print(f"Destination Port:      {dest}")
                print(f"Sequence Number:       {seq}")
                print(f"Acknowledgment Number: {ack_seq}")
                print(f"Data offset:           {doff}")
                print(f"FIN:                   {fin}")
                print(f"SYN:                   {syn}")
                print(f"RST:                   {rst}")
                print(f"PSH:                   {psh}")
                print(f"ACK:                   {ack}")
                print(f"URG:                   {urg}")
                print(f"ECE:                   {ece}")
                print(f"CWR:                   {cwr}")
                print(f"Window:                {window}")
                print(f"Checksum:              {check}")
                print(f"Urgent Point:          {urg_ptr}")
                print("--------------- DATA -----------------")
                print(data)

    except KeyboardInterrupt:
        pass

输出:

.. code-block:: console

    $ python3.6 tcp.py
    ------------ TCP_HEADER --------------
    Source Port:           38352
    Destination Port:      8000
    Sequence Number:       2907801591
    Acknowledgment Number: 398995857
    Data offset:           8
    FIN:                   0
    SYN:                   0
    RST:                   0
    PSH:                   8
    ACK:                   16
    URG:                   0
    ECE:                   0
    CWR:                   0
    Window:                342
    Checksum:              65142
    Urgent Point:          0
    --------------- DATA -----------------
    b'GET / HTTP/1.1\r\nHost: localhost:8000\r\nUser-Agent: curl/7.47.0\r\nAccept: */*\r\n\r\n'

ARP数据包嗅探器
------------------

.. code-block:: python

    """
    Ehternet Packet Header

    struct ethhdr {
        unsigned char h_dest[ETH_ALEN];   /* destination eth addr */
        unsigned char h_source[ETH_ALEN]; /* source ether addr    */
        __be16        h_proto;            /* packet type ID field */
    } __attribute__((packed));

    ARP Packet Header

    struct arphdr {
        uint16_t htype;    /* Hardware Type           */
        uint16_t ptype;    /* Protocol Type           */
        u_char   hlen;     /* Hardware Address Length */
        u_char   plen;     /* Protocol Address Length */
        uint16_t opcode;   /* Operation Code          */
        u_char   sha[6];   /* Sender hardware address */
        u_char   spa[4];   /* Sender IP address       */
        u_char   tha[6];   /* Target hardware address */
        u_char   tpa[4];   /* Target IP address       */
    };
    """

    import socket
    import struct
    import binascii

    rawSocket = socket.socket(socket.AF_PACKET,
                              socket.SOCK_RAW,
                              socket.htons(0x0003))

    while True:

        packet = rawSocket.recvfrom(2048)
        ethhdr = packet[0][0:14]
        eth = struct.unpack("!6s6s2s", ethhdr)

        arphdr = packet[0][14:42]
        arp = struct.unpack("2s2s1s1s2s6s4s6s4s", arphdr)
        # skip non-ARP packets
        ethtype = eth[2]
        if ethtype != '\x08\x06': continue

        print("-------------- ETHERNET_FRAME -------------")
        print("Dest MAC:        ", binascii.hexlify(eth[0]))
        print("Source MAC:      ", binascii.hexlify(eth[1]))
        print("Type:            ", binascii.hexlify(ethtype))
        print("--------------- ARP_HEADER ----------------")
        print("Hardware type:   ", binascii.hexlify(arp[0]))
        print("Protocol type:   ", binascii.hexlify(arp[1]))
        print("Hardware size:   ", binascii.hexlify(arp[2]))
        print("Protocol size:   ", binascii.hexlify(arp[3]))
        print("Opcode:          ", binascii.hexlify(arp[4]))
        print("Source MAC:      ", binascii.hexlify(arp[5]))
        print("Source IP:       ", socket.inet_ntoa(arp[6]))
        print("Dest MAC:        ", binascii.hexlify(arp[7]))
        print("Dest IP:         ", socket.inet_ntoa(arp[8]))
        print("-------------------------------------------")

输出:

.. code-block:: console

    $ python arp.py
    -------------- ETHERNET_FRAME -------------
    Dest MAC:         ffffffffffff
    Source MAC:       f0257252f5ca
    Type:             0806
    --------------- ARP_HEADER ----------------
    Hardware type:    0001
    Protocol type:    0800
    Hardware size:    06
    Protocol size:    04
    Opcode:           0001
    Source MAC:       f0257252f5ca
    Source IP:        140.112.91.254
    Dest MAC:         000000000000
    Dest IP:          140.112.91.20
    -------------------------------------------
