========
Security
========

.. contents:: Table of Contents
    :backlinks: none


Simple https server
---------------------

.. code-block:: python

    # python2

    >>> import BaseHTTPServer, SimpleHTTPServer
    >>> import ssl
    >>> host, port = 'localhost', 5566
    >>> handler = SimpleHTTPServer.SimpleHTTPRequestHandler
    >>> httpd = BaseHTTPServer.HTTPServer((host, port), handler)
    >>> httpd.socket = ssl.wrap_socket(httpd.socket,
    ...                                certfile='./cert.crt',
    ...                                keyfile='./cert.key',
    ...                                server_side=True)
    >>> httpd.serve_forever()

    # python3

    >>> from http import server
    >>> handler = server.SimpleHTTPRequestHandler
    >>> import ssl
    >>> host, port = 'localhost', 5566
    >>> httpd = server.HTTPServer((host, port), handler)
    >>> httpd.socket = ssl.wrap_socket(httpd.socket,
    ...                                certfile='./cert.crt',
    ...                                keyfile='./cert.key',
    ...                                server_side=True)
    ...
    >>> httpd.serve_forever()

Generate a SSH key pair
------------------------

.. code-block:: python

    from cryptography.hazmat.primitives import serialization
    from cryptography.hazmat.primitives.asymmetric import rsa
    from cryptography.hazmat.backends import default_backend

    key = rsa.generate_private_key(
        backend=default_backend(),
        public_exponent=65537,
        key_size=2048
    )
    private_key = key.private_bytes(
        serialization.Encoding.PEM,
        serialization.PrivateFormat.PKCS8,
        serialization.NoEncryption(),
    )
    public_key = key.public_key().public_bytes(
        serialization.Encoding.OpenSSH,
        serialization.PublicFormat.OpenSSH
    )

    with open('id_rsa', 'wb') as f, open('id_rsa.pub', 'wb') as g:
        f.write(private_key)
        g.write(public_key)

Get certificate information
----------------------------

.. code-block:: python

    from cryptography import x509
    from cryptography.hazmat.backends import default_backend

    backend = default_backend()
    with open('./cert.crt', 'rb') as f:
        crt_data = f.read()
        cert = x509.load_pem_x509_certificate(crt_data, backend)

    class Certificate:

        _fields = ['country_name',
                   'state_or_province_name',
                   'locality_name',
                   'organization_name',
                   'organizational_unit_name',
                   'common_name',
                   'email_address']

        def __init__(self, cert):
            assert isinstance(cert, x509.Certificate)
            self._cert = cert
            for attr in self._fields:
                oid = getattr(x509, 'OID_' + attr.upper())
                subject = cert.subject
                info = subject.get_attributes_for_oid(oid)
                setattr(self, attr, info)


    cert = Certificate(cert)
    for attr in cert._fields:
        for info in getattr(cert, attr):
            print("{}: {}".format(info._oid._name, info._value))

output:

.. code-block:: bash

    $ genrsa -out cert.key
    Generating RSA private key, 1024 bit long modulus
    ..........++++++
    ...++++++
    e is 65537 (0x10001)
    $ openssl req -x509 -new -nodes \
    >       -key cert.key -days 365 \
    >       -out cert.crt
    You are about to be asked to enter information that will be incorporated
    into your certificate request.
    What you are about to enter is what is called a Distinguished Name or a DN.
    There are quite a few fields but you can leave some blank
    For some fields there will be a default value,
    If you enter '.', the field will be left blank.
    -----
    Country Name (2 letter code) [AU]:TW
    State or Province Name (full name) [Some-State]:Taiwan
    Locality Name (eg, city) []:Taipei
    Organization Name (eg, company) [Internet Widgits Pty Ltd]:personal
    Organizational Unit Name (eg, section) []:personal
    Common Name (e.g. server FQDN or YOUR name) []:localhost
    Email Address []:test@example.com
    $ python3 cert.py
    countryName: TW
    stateOrProvinceName: Taiwan
    localityName: Taipei
    organizationName: personal
    organizationalUnitName: personal
    commonName: localhost
    emailAddress: test@example.com


Generate a self-signed certificate
-----------------------------------

.. code-block:: python

    from __future__ import print_function, unicode_literals

    from datetime import datetime, timedelta
    from OpenSSL import crypto

    # load private key
    ftype = crypto.FILETYPE_PEM
    with open('key.pem', 'rb') as f: k = f.read()
    k = crypto.load_privatekey(ftype, k)

    now    = datetime.now()
    expire = now + timedelta(days=365)

    # country (countryName, C)
    # state or province name (stateOrProvinceName, ST)
    # locality (locality, L)
    # organization (organizationName, O)
    # organizational unit (organizationalUnitName, OU)
    # common name (commonName, CN)

    cert = crypto.X509()
    cert.get_subject().C  = "TW"
    cert.get_subject().ST = "Taiwan"
    cert.get_subject().L  = "Taipei"
    cert.get_subject().O  = "pysheeet"
    cert.get_subject().OU = "cheat sheet"
    cert.get_subject().CN = "pythonsheets.com"
    cert.set_serial_number(1000)
    cert.set_notBefore(now.strftime("%Y%m%d%H%M%SZ").encode())
    cert.set_notAfter(expire.strftime("%Y%m%d%H%M%SZ").encode())
    cert.set_issuer(cert.get_subject())
    cert.set_pubkey(k)
    cert.sign(k, 'sha1')

    with open('cert.pem', "wb") as f:
        f.write(crypto.dump_certificate(ftype, cert))

output:

.. code-block:: bash

    $ openssl genrsa -out key.pem 2048
    Generating RSA private key, 2048 bit long modulus
    .............+++
    ..................................+++
    e is 65537 (0x10001)
    $ python3 x509.py
    $ openssl x509 -subject -issuer -noout -in cert.pem
    subject= /C=TW/ST=Taiwan/L=Taipei/O=pysheeet/OU=cheat sheet/CN=pythonsheets.com
    issuer= /C=TW/ST=Taiwan/L=Taipei/O=pysheeet/OU=cheat sheet/CN=pythonsheets.com


Prepare a Certificate Signing Request (csr)
--------------------------------------------

.. code-block:: python

    from __future__ import print_function, unicode_literals

    from OpenSSL import crypto

    # load private key
    ftype = crypto.FILETYPE_PEM
    with open('key.pem', 'rb') as f: key = f.read()
    key = crypto.load_privatekey(ftype, key)
    req    = crypto.X509Req()

    alt_name  = [ b"DNS:www.pythonsheeets.com",
                  b"DNS:doc.pythonsheeets.com" ]
    key_usage = [ b"Digital Signature",
                  b"Non Repudiation",
                  b"Key Encipherment" ]

    # country (countryName, C)
    # state or province name (stateOrProvinceName, ST)
    # locality (locality, L)
    # organization (organizationName, O)
    # organizational unit (organizationalUnitName, OU)
    # common name (commonName, CN)

    req.get_subject().C  = "TW"
    req.get_subject().ST = "Taiwan"
    req.get_subject().L  = "Taipei"
    req.get_subject().O  = "pysheeet"
    req.get_subject().OU = "cheat sheet"
    req.get_subject().CN = "pythonsheets.com"
    req.add_extensions([
        crypto.X509Extension( b"basicConstraints",
                              False,
                              b"CA:FALSE"),
        crypto.X509Extension( b"keyUsage",
                              False,
                              b",".join(key_usage)),
        crypto.X509Extension( b"subjectAltName",
                              False,
                              b",".join(alt_name))
    ])

    req.set_pubkey(key)
    req.sign(key, "sha256")

    csr = crypto.dump_certificate_request(ftype, req)
    with open("cert.csr", 'wb') as f: f.write(csr)

output:

.. code-block:: bash

    # create a root ca
    $ openssl genrsa -out ca-key.pem 2048
    Generating RSA private key, 2048 bit long modulus
    .....+++
    .......................................+++
    e is 65537 (0x10001)
    $ openssl req -x509 -new -nodes -key ca-key.pem \
    > -days 10000 -out ca.pem -subj "/CN=root-ca"

    # prepare a csr
    $ openssl genrsa -out key.pem 2048
    Generating RSA private key, 2048 bit long modulus
    ....+++
    ......................................+++
    e is 65537 (0x10001)
    $ python3 x509.py

    # prepare openssl.cnf
    cat <<EOF > openssl.cnf
    > [req]
    > req_extensions = v3_req
    > distinguished_name = req_distinguished_name
    > [req_distinguished_name]
    > [ v3_req ]
    > basicConstraints = CA:FALSE
    > keyUsage = nonRepudiation, digitalSignature, keyEncipherment
    > subjectAltName = @alt_names
    > [alt_names]
    > DNS.1 = www.pythonsheets.com
    > DNS.2 = doc.pythonsheets.com
    > EOF

    # sign a csr
    $ openssl x509 -req -in cert.csr -CA ca.pem \
    > -CAkey ca-key.pem -CAcreateserial -out cert.pem \
    > -days 365 -extensions v3_req -extfile openssl.cnf
    Signature ok
    subject=/C=TW/ST=Taiwan/L=Taipei/O=pysheeet/OU=cheat sheet/CN=pythonsheets.com
    Getting CA Private Key

    # check
    $ openssl x509 -in cert.pem -text -noout


Generate RSA keyfile without passphrase
-----------------------------------------

.. code-block:: python

    # $ openssl genrsa cert.key 2048

    >>> from cryptography.hazmat.backends import default_backend
    >>> from cryptography.hazmat.primitives import serialization
    >>> from cryptography.hazmat.primitives.asymmetric import rsa
    >>> key = rsa.generate_private_key(
    ... public_exponent=65537,
    ... key_size=2048,
    ... backend=default_backend())
    ...
    >>> with open('cert.key', 'wb') as f:
    ...     f.write(key.private_bytes(
    ...     encoding=serialization.Encoding.PEM,
    ...     format=serialization.PrivateFormat.TraditionalOpenSSL,
    ...     encryption_algorithm=serialization.NoEncryption()))


Sign a file by a given private key
-----------------------------------

.. code-block:: python

    from __future__ import print_function, unicode_literals

    from Crypto.PublicKey import RSA
    from Crypto.Signature import PKCS1_v1_5
    from Crypto.Hash import SHA256


    def signer(privkey, data):
        rsakey = RSA.importKey(privkey)
        signer = PKCS1_v1_5.new(rsakey)
        digest = SHA256.new()
        digest.update(data)
        return signer.sign(digest)


    with open('private.key', 'rb') as f: key = f.read()
    with open('foo.tgz', 'rb') as f: data = f.read()

    sign = signer(key, data)
    with open('foo.tgz.sha256', 'wb') as f: f.write(sign)

output:

.. code-block:: bash

    # gernerate public & private key
    $ openssl genrsa -out private.key 2048
    $ openssl rsa -in private.key -pubout -out public.key

    $ python3 sign.py
    $ openssl dgst -sha256 -verify public.key -signature foo.tgz.sha256 foo.tgz
    Verified OK


Verify a file from a signed digest
-----------------------------------

.. code-block:: python

    from __future__ import print_function, unicode_literals

    import sys

    from Crypto.PublicKey import RSA
    from Crypto.Signature import PKCS1_v1_5
    from Crypto.Hash import SHA256

    def verifier(pubkey, sig, data):
        rsakey = RSA.importKey(key)
        signer = PKCS1_v1_5.new(rsakey)
        digest = SHA256.new()

        digest.update(data)
        return signer.verify(digest, sig)


    with open("public.key", 'rb') as f: key = f.read()
    with open("foo.tgz.sha256", 'rb') as f: sig = f.read()
    with open("foo.tgz", 'rb') as f: data = f.read()

    if verifier(key, sig, data):
        print("Verified OK")
    else:
        print("Verification Failure")

output:

.. code-block:: bash

    # gernerate public & private key
    $ openssl genrsa -out private.key 2048
    $ openssl rsa -in private.key -pubout -out public.key

    # do verification
    $ cat /dev/urandom | head -c 512 | base64 > foo.txt
    $ tar -zcf foo.tgz foo.txt
    $ openssl dgst -sha256 -sign private.key -out foo.tgz.sha256 foo.tgz
    $ python3 verify.py
    Verified OK

    # do verification via openssl
    $ openssl dgst -sha256 -verify public.key -signature foo.tgz.sha256 foo.tgz
    Verified OK


Simple RSA encrypt via pem file
--------------------------------

.. code-block:: python

    from __future__ import print_function, unicode_literals

    import base64
    import sys

    from Crypto.PublicKey import RSA
    from Crypto.Cipher import PKCS1_v1_5

    key_text = sys.stdin.read()

    # import key via rsa module
    pubkey = RSA.importKey(key_text)

    # create a cipher via PKCS1.5
    cipher = PKCS1_v1_5.new(pubkey)

    # encrypt
    cipher_text = cipher.encrypt(b"Hello RSA!")

    # do base64 encode
    cipher_text = base64.b64encode(cipher_text)
    print(cipher_text.decode('utf-8'))

output:

.. code-block:: bash

    $ openssl genrsa -out private.key 2048
    $ openssl rsa -in private.key -pubout -out public.key
    $ cat public.key                                |\
    > python3 rsa.py                                |\
    > openssl base64 -d -A                          |\
    > openssl rsautl -decrypt -inkey private.key
    Hello RSA!


Simple RSA encrypt via RSA module
----------------------------------

.. code-block:: python

    from __future__ import print_function, unicode_literals

    import base64
    import sys

    from Crypto.PublicKey import RSA
    from Crypto.Cipher import PKCS1_v1_5
    from Crypto.PublicKey.RSA import construct

    # prepare public key
    e = int('10001', 16)
    n = int(sys.stdin.read(), 16)
    pubkey = construct((n, e))

    # create a cipher via PKCS1.5
    cipher = PKCS1_v1_5.new(pubkey)

    # encrypt
    cipher_text = cipher.encrypt(b"Hello RSA!")

    # do base64 encode
    cipher_text = base64.b64encode(cipher_text)
    print(cipher_text.decode('utf-8'))

output:

.. code-block:: bash

    $ openssl genrsa -out private.key 2048
    $ openssl rsa -in private.key -pubout -out public.key
    $ # check (n, e)
    $ openssl rsa -pubin -inform PEM -text -noout < public.key
    Public-Key: (2048 bit)
    Modulus:
        00:93:d5:58:0c:18:cf:91:f0:74:af:1b:40:09:73:
        0c:d8:13:23:6c:44:60:0d:83:71:e6:f9:61:85:e5:
        b2:d0:8a:73:5c:02:02:51:9a:4f:a7:ab:05:d5:74:
        ff:4d:88:3d:e2:91:b8:b0:9f:7e:a9:a3:b2:3c:99:
        1c:9a:42:4d:ac:2f:6a:e7:eb:0f:a7:e0:a5:81:e5:
        98:49:49:d5:15:3d:53:42:12:08:db:b0:e7:66:2d:
        71:5b:ea:55:4e:2d:9b:40:79:f8:7d:6e:5d:f4:a7:
        d8:13:cb:13:91:c9:ac:5b:55:62:70:44:25:50:ca:
        94:de:78:5d:97:e8:a9:33:66:4f:90:10:00:62:21:
        b6:60:52:65:76:bd:a3:3b:cf:2a:db:3f:66:5f:0d:
        a3:35:ff:29:34:26:6d:63:a2:a6:77:96:5a:84:c7:
        6a:0c:4f:48:52:70:11:8f:85:11:a0:78:f8:60:4b:
        5d:d8:4b:b2:64:e5:ec:99:72:c5:a8:1b:ab:5c:09:
        e1:80:70:91:06:22:ba:97:33:56:0b:65:d8:f3:35:
        66:f8:f9:ea:b9:84:64:8e:3c:14:f7:3d:1f:2c:67:
        ce:64:cf:f9:c5:16:6b:03:a1:7a:c7:fa:4c:38:56:
        ee:e0:4d:5f:ec:46:7e:1f:08:7c:e6:45:a1:fc:17:
        1f:91
    Exponent: 65537 (0x10001)
    $ openssl rsa -pubin -in public.key -modulus -noout |\
    > cut -d'=' -f 2                                    |\
    > python3 rsa.py                                    |\
    > openssl base64 -d -A                              |\
    > openssl rsautl -decrypt -inkey private.key
    Hello RSA!

Simple RSA decrypt via pem file
--------------------------------

.. code-block:: python

    from __future__ import print_function, unicode_literals

    import base64
    import sys

    from Crypto.PublicKey import RSA
    from Crypto.Cipher import PKCS1_v1_5

    # read key file
    with open('private.key') as f: key_text = f.read()

    # create a private key object
    privkey = RSA.importKey(key_text)

    # create a cipher object
    cipher = PKCS1_v1_5.new(privkey)

    # decode base64
    cipher_text = base64.b64decode(sys.stdin.read())

    # decrypt
    plain_text = cipher.decrypt(cipher_text, None)
    print(plain_text.decode('utf-8').strip())

output:

.. code-block:: bash

    $ openssl genrsa -out private.key 2048
    $ openssl rsa -in private.key -pubout -out public.key
    $ echo "Hello openssl RSA encrypt"                 |\
    > openssl rsautl -encrypt -pubin -inkey public.key |\
    > openssl base64 -e -A                             |\
    > python3 rsa.py
    Hello openssl RSA encrypt


Simple RSA encrypt with OAEP
-----------------------------

.. code-block:: python

    from __future__ import print_function, unicode_literals

    import base64
    import sys

    from Crypto.PublicKey import RSA
    from Crypto.Cipher import PKCS1_OAEP

    # read key file
    key_text = sys.stdin.read()

    # create a public key object
    pubkey = RSA.importKey(key_text)

    # create a cipher object
    cipher = PKCS1_OAEP.new(pubkey)

    # encrypt plain text
    cipher_text = cipher.encrypt(b"Hello RSA OAEP!")

    # encode via base64
    cipher_text = base64.b64encode(cipher_text)
    print(cipher_text.decode('utf-8'))

output:

.. code-block:: bash

    $ openssl genrsa -out private.key 2048
    $ openssl rsa -in private.key -pubout -out public.key
    $ cat public.key         |\
    > python3 rsa.py         |\
    > openssl base64 -d -A   |\
    > openssl rsautl -decrypt -oaep -inkey private.key
    Hello RSA OAEP!


Simple RSA decrypt with OAEP
-----------------------------

.. code-block:: python

    from __future__ import print_function, unicode_literals

    import base64
    import sys

    from Crypto.PublicKey import RSA
    from Crypto.Cipher import PKCS1_OAEP

    # read key file
    with open('private.key') as f: key_text = f.read()

    # create a private key object
    privkey = RSA.importKey(key_text)

    # create a cipher object
    cipher = PKCS1_OAEP.new(privkey)

    # decode base64
    cipher_text = base64.b64decode(sys.stdin.read())

    # decrypt
    plain_text = cipher.decrypt(cipher_text)
    print(plain_text.decode('utf-8').strip())

output:

.. code-block:: bash

    $ openssl genrsa -out private.key 2048
    $ openssl rsa -in private.key -pubout -out public.key
    $ echo "Hello RSA encrypt via OAEP"                      |\
    > openssl rsautl -encrypt -pubin -oaep -inkey public.key |\
    > openssl base64 -e -A                                   |\
    > python3 rsa.py
    Hello RSA encrypt via OAEP


Using DSA to proof of identity
--------------------------------

.. code-block:: python

    import socket

    from cryptography.exceptions import InvalidSignature
    from cryptography.hazmat.backends import default_backend
    from cryptography.hazmat.primitives import hashes
    from cryptography.hazmat.primitives.asymmetric import dsa

    alice, bob = socket.socketpair()

    def gen_dsa_key():
        private_key = dsa.generate_private_key(
            key_size=2048, backend=default_backend())
        return private_key, private_key.public_key()


    def sign_data(data, private_key):
        signature = private_key.sign(data, hashes.SHA256())
        return signature


    def verify_data(data, signature, public_key):
        try:
            public_key.verify(signature, data, hashes.SHA256())
        except InvalidSignature:
            print("recv msg: {} not trust!".format(data))
        else:
            print("check msg: {} success!".format(data))


    # generate alice private & public key
    alice_private_key, alice_public_key = gen_dsa_key()

    # alice send message to bob, then bob recv
    alice_msg = b"Hello Bob"
    b = alice.send(alice_msg)
    bob_recv_msg = bob.recv(1024)

    # alice send signature to bob, then bob recv
    signature = sign_data(alice_msg, alice_private_key)
    b = alice.send(signature)
    bob_recv_signature = bob.recv(1024)

    # bob check message recv from alice
    verify_data(bob_recv_msg, bob_recv_signature, alice_public_key)

    # attacker modify the msg will make the msg check fail
    verify_data(b"I'm attacker!", bob_recv_signature, alice_public_key)

output:

.. code-block:: bash

    $ python3 test_dsa.py
    check msg: b'Hello Bob' success!
    recv msg: b"I'm attacker!" not trust!


Using AES CBC mode encrypt a file
----------------------------------

.. code-block:: python

    from __future__ import print_function, unicode_literals

    import struct
    import sys
    import os

    from cryptography.hazmat.primitives import padding
    from cryptography.hazmat.backends import default_backend
    from cryptography.hazmat.primitives.ciphers import (
        Cipher,
        algorithms,
        modes)

    backend = default_backend()
    key = os.urandom(32)
    iv  = os.urandom(16)

    def encrypt(ptext):
        pad = padding.PKCS7(128).padder()
        ptext = pad.update(ptext) + pad.finalize()

        alg = algorithms.AES(key)
        mode = modes.CBC(iv)
        cipher = Cipher(alg, mode, backend=backend)
        encryptor = cipher.encryptor()
        ctext = encryptor.update(ptext) + encryptor.finalize()

        return ctext

    print("key: {}".format(key.hex()))
    print("iv: {}".format(iv.hex()))

    if len(sys.argv) != 3:
        raise Exception("usage: cmd [file] [enc file]")

    # read plain text from file
    with open(sys.argv[1], 'rb') as f:
        plaintext = f.read()

    # encrypt file
    ciphertext = encrypt(plaintext)
    with open(sys.argv[2], 'wb') as f:
        f.write(ciphertext)

output:

.. code-block:: bash

    $ echo "Encrypt file via AES-CBC" > test.txt
    $ python3 aes.py test.txt test.enc
    key: f239d9609e3f318b7afda7e4bb8db5b8734f504cf67f55e45dfe75f90d24fefc
    iv: 8d6383b469f100d25293fb244ccb951e
    $ openssl aes-256-cbc -d -in test.enc -out secrets.txt.new            \
    > -K f239d9609e3f318b7afda7e4bb8db5b8734f504cf67f55e45dfe75f90d24fefc \
    > -iv 8d6383b469f100d25293fb244ccb951e
    $ cat secrets.txt.new
    Encrypt file via AES-CBC


Using AES CBC mode decrypt a file
----------------------------------

.. code-block:: python

    from __future__ import print_function, unicode_literals

    import struct
    import sys
    import os

    from binascii import unhexlify

    from cryptography.hazmat.primitives import padding
    from cryptography.hazmat.backends import default_backend
    from cryptography.hazmat.primitives.ciphers import (
        Cipher,
        algorithms,
        modes)

    backend = default_backend()

    def decrypt(key, iv, ctext):
        alg = algorithms.AES(key)
        mode = modes.CBC(iv)
        cipher = Cipher(alg, mode, backend=backend)
        decryptor = cipher.decryptor()
        ptext = decryptor.update(ctext) + decryptor.finalize()

        unpadder = padding.PKCS7(128).unpadder() # 128 bit
        ptext = unpadder.update(ptext) + unpadder.finalize()

        return ptext

    if len(sys.argv) != 4:
        raise Exception("usage: cmd [key] [iv] [file]")

    # read cipher text from file
    with open(sys.argv[3], 'rb') as f:
        ciphertext = f.read()

    # decrypt file
    key, iv = unhexlify(sys.argv[1]), unhexlify(sys.argv[2])
    plaintext = decrypt(key, iv, ciphertext)
    print(plaintext)

output:

.. code-block:: bash

    $ echo "Encrypt file via AES-CBC" > test.txt
    $ key=`openssl rand -hex 32`
    $ iv=`openssl rand -hex 16`
    $ openssl enc -aes-256-cbc -in test.txt -out test.enc -K $key -iv $iv
    $ python3 aes.py $key $iv test.enc


AES CBC mode encrypt via password (using cryptography)
-------------------------------------------------------

.. code-block:: python

    from __future__ import print_function, unicode_literals

    import base64
    import struct
    import sys
    import os

    from hashlib import md5, sha1

    from cryptography.hazmat.primitives import padding
    from cryptography.hazmat.backends import default_backend
    from cryptography.hazmat.primitives.ciphers import (
        Cipher,
        algorithms,
        modes)

    backend = default_backend()

    def EVP_ByteToKey(pwd, md, salt, key_len, iv_len):
        buf = md(pwd + salt).digest()
        d = buf
        while len(buf) < (iv_len + key_len):
            d = md(d + pwd + salt).digest()
            buf += d
        return buf[:key_len], buf[key_len:key_len + iv_len]


    def aes_encrypt(pwd, ptext, md):
        key_len, iv_len = 32, 16

        # generate salt
        salt = os.urandom(8)

        # generate key, iv from password
        key, iv = EVP_ByteToKey(pwd, md, salt, key_len, iv_len)

        # pad plaintext
        pad = padding.PKCS7(128).padder()
        ptext = pad.update(ptext) + pad.finalize()

        # create an encryptor
        alg = algorithms.AES(key)
        mode = modes.CBC(iv)
        cipher = Cipher(alg, mode, backend=backend)
        encryptor = cipher.encryptor()

        # encrypt plain text
        ctext = encryptor.update(ptext) + encryptor.finalize()
        ctext = b'Salted__' + salt + ctext

        # encode base64
        ctext = base64.b64encode(ctext)
        return ctext


    if len(sys.argv) != 2: raise Exception("usage: CMD [md]")

    md = globals()[sys.argv[1]]

    plaintext = sys.stdin.read().encode('utf-8')
    pwd = b"password"

    print(aes_encrypt(pwd, plaintext, md).decode('utf-8'))

output:

.. code-block:: bash

    # with md5 digest
    $ echo "Encrypt plaintext via AES-CBC from a given password" |\
    > python3 aes.py md5                                         |\
    > openssl base64 -d -A                                       |\
    > openssl aes-256-cbc -md md5 -d -k password
    Encrypt plaintext via AES-CBC from a given password

    # with sha1 digest
    $ echo "Encrypt plaintext via AES-CBC from a given password" |\
    > python3 aes.py sha1                                        |\
    > openssl base64 -d -A                                       |\
    > openssl aes-256-cbc -md sha1 -d -k password
    Encrypt plaintext via AES-CBC from a given password


AES CBC mode decrypt via password (using cryptography)
--------------------------------------------------------

.. code-block:: python

    from __future__ import print_function, unicode_literals

    import base64
    import struct
    import sys
    import os

    from hashlib import md5, sha1

    from cryptography.hazmat.primitives import padding
    from cryptography.hazmat.backends import default_backend
    from cryptography.hazmat.primitives.ciphers import (
        Cipher,
        algorithms,
        modes)

    backend = default_backend()

    def EVP_ByteToKey(pwd, md, salt, key_len, iv_len):
        buf = md(pwd + salt).digest()
        d = buf
        while len(buf) < (iv_len + key_len):
            d = md(d + pwd + salt).digest()
            buf += d
        return buf[:key_len], buf[key_len:key_len + iv_len]


    def aes_decrypt(pwd, ctext, md):
        ctext = base64.b64decode(ctext)

        # check magic
        if ctext[:8] != b'Salted__':
            raise Exception("bad magic number")

        # get salt
        salt = ctext[8:16]

        # generate key, iv from password
        key, iv = EVP_ByteToKey(pwd, md, salt, 32, 16)

        # decrypt
        alg = algorithms.AES(key)
        mode = modes.CBC(iv)
        cipher = Cipher(alg, mode, backend=backend)
        decryptor = cipher.decryptor()
        ptext = decryptor.update(ctext[16:]) + decryptor.finalize()

        # unpad plaintext
        unpadder = padding.PKCS7(128).unpadder() # 128 bit
        ptext = unpadder.update(ptext) + unpadder.finalize()
        return ptext.strip()

    if len(sys.argv) != 2: raise Exception("usage: CMD [md]")

    md = globals()[sys.argv[1]]

    ciphertext = sys.stdin.read().encode('utf-8')
    pwd = b"password"

    print(aes_decrypt(pwd, ciphertext, md).decode('utf-8'))

output:

.. code-block:: bash

    # with md5 digest
    $ echo "Decrypt ciphertext via AES-CBC from a given password" |\
    > openssl aes-256-cbc -e -md md5 -salt -A -k password         |\
    > openssl base64 -e -A                                        |\
    > python3 aes.py md5
    Decrypt ciphertext via AES-CBC from a given password

    # with sha1 digest
    $ echo "Decrypt ciphertext via AES-CBC from a given password" |\
    > openssl aes-256-cbc -e -md sha1 -salt -A -k password        |\
    > openssl base64 -e -A                                        |\
    > python3 aes.py sha1
    Decrypt ciphertext via AES-CBC from a given password


AES CBC mode encrypt via password (using pycrypto)
---------------------------------------------------

.. code-block:: python

    from __future__ import print_function, unicode_literals

    import struct
    import base64
    import sys

    from hashlib import md5, sha1
    from Crypto.Cipher import AES
    from Crypto.Random.random import getrandbits

    # AES CBC requires blocks to be aligned on 16-byte boundaries.
    BS = 16

    pad = lambda s: s + (BS - len(s) % BS) * chr(BS - len(s) % BS).encode('utf-8')
    unpad = lambda s : s[0:-ord(s[-1])]

    def EVP_ByteToKey(pwd, md, salt, key_len, iv_len):
        buf = md(pwd + salt).digest()
        d = buf
        while len(buf) < (iv_len + key_len):
            d = md(d + pwd + salt).digest()
            buf += d
        return buf[:key_len], buf[key_len:key_len + iv_len]


    def aes_encrypt(pwd, plaintext, md):
        key_len, iv_len = 32, 16

        # generate salt
        salt = struct.pack('=Q', getrandbits(64))

        # generate key, iv from password
        key, iv = EVP_ByteToKey(pwd, md, salt, key_len, iv_len)

        # pad plaintext
        plaintext = pad(plaintext)

        # create a cipher object
        cipher = AES.new(key, AES.MODE_CBC, iv)

        # ref: openssl/apps/enc.c
        ciphertext = b'Salted__' + salt + cipher.encrypt(plaintext)

        # encode base64
        ciphertext = base64.b64encode(ciphertext)
        return ciphertext

    if len(sys.argv) != 2: raise Exception("usage: CMD [md]")

    md = globals()[sys.argv[1]]

    plaintext = sys.stdin.read().encode('utf-8')
    pwd = b"password"

    print(aes_encrypt(pwd, plaintext, md).decode('utf-8'))

output:

.. code-block:: bash

    # with md5 digest
    $ echo "Encrypt plaintext via AES-CBC from a given password" |\
    > python3 aes.py md5                                         |\
    > openssl base64 -d -A                                       |\
    > openssl aes-256-cbc -md md5 -d -k password
    Encrypt plaintext via AES-CBC from a given password

    # with sha1 digest
    $ echo "Encrypt plaintext via AES-CBC from a given password" |\
    > python3 aes.py sha1                                        |\
    > openssl base64 -d -A                                       |\
    > openssl aes-256-cbc -md sha1 -d -k password
    Encrypt plaintext via AES-CBC from a given password


AES CBC mode decrypt via password (using pycrytpo)
---------------------------------------------------

.. code-block:: python

    from __future__ import print_function, unicode_literals

    import struct
    import base64
    import sys

    from hashlib import md5, sha1
    from Crypto.Cipher import AES
    from Crypto.Random.random import getrandbits

    # AES CBC requires blocks to be aligned on 16-byte boundaries.
    BS = 16

    unpad = lambda s : s[0:-s[-1]]

    def EVP_ByteToKey(pwd, md, salt, key_len, iv_len):
        buf = md(pwd + salt).digest()
        d = buf
        while len(buf) < (iv_len + key_len):
            d = md(d + pwd + salt).digest()
            buf += d
        return buf[:key_len], buf[key_len:key_len + iv_len]


    def aes_decrypt(pwd, ciphertext, md):
        ciphertext = base64.b64decode(ciphertext)

        # check magic
        if ciphertext[:8] != b'Salted__':
            raise Exception("bad magic number")

        # get salt
        salt = ciphertext[8:16]

        # get key, iv
        key, iv = EVP_ByteToKey(pwd, md, salt, 32, 16)

        # decrypt
        cipher = AES.new(key, AES.MODE_CBC, iv)
        return unpad(cipher.decrypt(ciphertext[16:])).strip()


    if len(sys.argv) != 2: raise Exception("usage: CMD [md]")

    md = globals()[sys.argv[1]]

    ciphertext = sys.stdin.read().encode('utf-8')
    pwd = b"password"

    print(aes_decrypt(pwd, ciphertext, md).decode('utf-8'))

output:

.. code-block:: bash

    # with md5 digest
    $ echo "Decrypt ciphertext via AES-CBC from a given password" |\
    > openssl aes-256-cbc -e -md md5 -salt -A -k password         |\
    > openssl base64 -e -A                                        |\
    > python3 aes.py md5
    Decrypt ciphertext via AES-CBC from a given password

    # with sha1 digest
    $ echo "Decrypt ciphertext via AES-CBC from a given password" |\
    > openssl aes-256-cbc -e -md sha1 -salt -A -k password        |\
    > openssl base64 -e -A                                        |\
    > python3 aes.py sha1
    Decrypt ciphertext via AES-CBC from a given password


Ephemeral Diffie Hellman Key Exchange via cryptography
-------------------------------------------------------

.. code-block:: python

    >>> from cryptography.hazmat.backends import default_backend
    >>> from cryptography.hazmat.primitives.asymmetric import dh
    >>> params = dh.generate_parameters(2, 512, default_backend())
    >>> a_key = params.generate_private_key()  # alice's private key
    >>> b_key = params.generate_private_key()  # bob's private key
    >>> a_pub_key = a_key.public_key()
    >>> b_pub_key = b_key.public_key()
    >>> a_shared_key = a_key.exchange(b_pub_key)
    >>> b_shared_key = b_key.exchange(a_pub_key)
    >>> a_shared_key == b_shared_key
    True

Calculate DH shared key manually via cryptography
---------------------------------------------------

.. code-block:: python

    >>> from cryptography.hazmat.backends import default_backend
    >>> from cryptography.hazmat.primitives.asymmetric import dh
    >>> from cryptography.utils import int_from_bytes
    >>> a_key = params.generate_private_key()  # alice's private key
    >>> b_key = params.generate_private_key()  # bob's private key
    >>> a_pub_key = a_key.public_key()
    >>> b_pub_key = b_key.public_key()
    >>> shared_key = int_from_bytes(a_key.exchange(b_pub_key), 'big')
    >>> shared_key_manual = pow(a_pub_key.public_numbers().y,
    ...                         b_key.private_numbers().x,
    ...                         params.parameter_numbers().p)
    >>> shared_key == shared_key_manual
    True

Calculate DH shared key from (p, g, pubkey)
---------------------------------------------

.. code-block:: python

    from cryptography.hazmat.backends import default_backend
    from cryptography.hazmat.primitives.asymmetric import dh
    from cryptography.utils import int_from_bytes

    backend = default_backend()

    p = int("11859949538425015739337467917303613431031019140213666"
            "12902540730065402658508634532306628480096346320424639"
            "0256567934582260424238844463330887962689642467123")

    g = 2

    y = int("32155788395534640648739966373159697798396966919821525"
            "72238852825117261342483718574508213761865276905503199"
            "969908098203345481366464874759377454476688391248")

    x = int("409364065449673443397833358558926598469347813468816037"
            "268451847116982490733450463194921405069999008617231539"
            "7147035896687401350877308899732826446337707128")

    params = dh.DHParameterNumbers(p, g)
    public = dh.DHPublicNumbers(y, params)
    private = dh.DHPrivateNumbers(x, public)

    key = private.private_key(backend)
    shared_key = key.exchange(public.public_key(backend))

    # check shared key
    shared_key = int_from_bytes(shared_key, 'big')
    shared_key_manual = pow(y, x, p)   # y^x mod p

    assert shared_key == shared_key_manual
