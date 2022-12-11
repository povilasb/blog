.. title: Python SSL MemoryBIO usage
.. slug: python-ssl-memorybio-usage
.. date: 2017-02-07 18:47:23 UTC+02:00
.. tags: ssl,tls,python,python3
.. category: ssl
.. link:
.. description: Samples how to use ssl.MemoryBIO introduced in python 3.
.. type: text

OpenSSL has a basic I/O abstraction which is abbreviated as BIO.

This abstraction let's you encode raw stream to TLS and back in memory -
without actually doing any network I/O.

Python 3.5 introduced an API to use this feature:
https://docs.python.org/3/library/ssl.html#memory-bio-support.

I haven't found any samples/tutorials how to use these objects, so I'm
going to describe it briefly.

`SSLObject <https://docs.python.org/3/library/ssl.html#ssl.SSLObject>`_ and
`MemoryBIO <https://docs.python.org/3/library/ssl.html#ssl.MemoryBIO>`_ are
the core objects to do TLS.
`SSLObject` does the data encryption/decryption and `MemoryBIO` objects
are used to feed data to `SSLObject` and receive it back.

The only way to create `SSLObject` is to use
`SSLContext.wrap_bio() <https://docs.python.org/3/library/ssl.html#ssl.SSLContext.wrap_bio>`_.

.. code-block:: python

    import ssl

    tls_in_buff = ssl.MemoryBIO()
    tls_out_buff = ssl.MemoryBIO()

    ctx = ssl.SSLContext(ssl.PROTOCOL_SSLv23)
    ctx.load_cert_chain('localhost.crt', 'private_key.pem')
    tls_obj = ctx.wrap_bio(tls_in_buff, tls_out_buff, server_side=True)

To decrypt data we write it into input `MemoryBIO` object and then read
the raw data from `SSLObject`:

.. code-block:: python

        tls_in_buff.write(sock.recv(4096))
        http_req = tls_obj.read()
        print(http_req)

To encrypt data we write into `SSLObject` and then read the encrypted data
from output `MemoryBIO`:

.. code-block:: python

        tls_obj.write(b'HTTP/1.1 200 OK\r\n\r\n')
        sock.sendall(tls_out_buff.read())

Basically encryption/decryption could be depicted as::

         b'data'    b'\x17\x03\x03\x00\x1c\xc1...'
            |                  |
    write() |                  | write()
            |                  v
            |             +-----------+
            |             |   input   |
            |             | MemoryBIO |
            |             +-----------+
            |                  |
            v                  v
         +-----------------------------+
         |                             |
         |          SSLObject          |
         |                             |
         +-----------------------------+
            |                 |
            |                 | read()
            v                 v
      +-----------+         b'data'
      |   output  |
      | MemoryBIO |
      +-----------+
           |
    read() |
           V

    b'\x17\x03\x03\x00\x1c\xc1...'

Sample HTTPS Server
===================

.. code-block:: python
    :linenos:

    import socket
    import ssl


    def do_tls_handshake(sock: socket.socket, tls_obj, tls_in_buff: ssl.MemoryBIO,
                         tls_out_buff: ssl.MemoryBIO) -> None:
        client_hello = sock.recv(4096)
        tls_in_buff.write(client_hello)

        try:
            tls_obj.do_handshake()
        except ssl.SSLWantReadError:
            server_hello = tls_out_buff.read()
            sock.sendall(server_hello)

        client_fin = sock.recv(4096)
        tls_in_buff.write(client_fin)
        tls_obj.do_handshake()

        server_fin = tls_out_buff.read()
        sock.sendall(server_fin)


    def make_tls_objects():
        tls_in_buff = ssl.MemoryBIO()
        tls_out_buff = ssl.MemoryBIO()

        ctx = ssl.SSLContext(ssl.PROTOCOL_SSLv23)
        ctx.load_cert_chain('localhost.crt', 'private_key.pem')
        tls_obj = ctx.wrap_bio(tls_in_buff, tls_out_buff, server_side=True)

        return tls_obj, tls_in_buff, tls_out_buff


    def respond_to_http_request(sock: socket.socket) -> None:
        tls_obj, tls_in_buff, tls_out_buff = make_tls_objects()
        do_tls_handshake(sock, tls_obj, tls_in_buff, tls_out_buff)

        tls_in_buff.write(sock.recv(4096))
        http_req = tls_obj.read()
        print(http_req)

        tls_obj.write(b'HTTP/1.1 200 OK\r\n\r\n')
        sock.sendall(tls_out_buff.read())


    server_sock = socket.socket()
    server_sock.bind(('0.0.0.0', 5000))
    server_sock.listen(65535)

    client_sock, _ = server_sock.accept()
    respond_to_http_request(client_sock)
    client_sock.close()
