.. title: Opportunistic TLS with python asyncio
.. slug: opportunistic-tls-with-python-asyncio
.. date: 2017-02-20 07:40:47 UTC+02:00
.. tags: python3, asyncio, tls, ssl, starttls
.. category:
.. link:
.. description:
.. type: text

Recently I was implementing an HTTPS proxy that was doing a man in the middle
attack similar to what https://mitmproxy.org/ does.
This server upgrades TCP connection to TLS on `HTTP CONNECT
<https://en.wikipedia.org/wiki/HTTP_tunnel#HTTP_CONNECT_tunneling>`_ method.
While researching how to do this with asyncio I came across this python
issue https://bugs.python.org/issue23749.

After reading the discussion it was clear that `asyncio.sslproto.SSLProtocol`
has TLS/SSL implementation.
Also Yury Selivanov posted a recent patch that allows us to wrap application
protocol inside `SSLProtocol`: https://hg.python.org/cpython/rev/3e6739e5c2d0.

Eventually, I ended up with a prototype:

.. code-block:: python

    # https://github.com/benoitc/http-parser/
    from http_parser.parser import HttpParser


    class HttpServerProtocol(asyncio.Protocol):
        def __init__(self) -> None:
            super().__init__()
            self._http_parser = HttpParser()
            self._transport = None

        def request_received(self) -> None:
            self._transport.write(
                b'HTTP/1.1 200 OK\r\n'
                b'Content-Length: 2\r\n'
                b'Connection: close\r\n\r\n'
                b'OK'
            )
            self._transport.close()

        def data_received(self, data: bytes) -> None:
            self._http_parser.execute(data, len(data))
            if self._http_parser.is_message_complete():
                self.request_received()

        def connection_made(self, transport) -> None:
            self._transport = transport

        def close(self):
            self._transport.close()


    class HttpsServerProtocol(HttpServerProtocol):
        def __init__(self, loop) -> None:
            super().__init__()

            self._over_tls = False
            self._tls_proto = asyncio.sslproto.SSLProtocol(
                loop, HttpServerProtocol(), make_tls_context(), None,
                server_side=True
            )

        def connect_received(self) -> None:
            self._over_tls = True
            self._tls_proto.connection_made(self._transport)

        def request_received(self) -> None:
            if self._http_parser.get_method() == 'CONNECT':
                self._transport.write(b'HTTP/1.1 200 OK\r\n\r\n')
                self.connect_received()
            else:
                super().request_received()

        def data_received(self, data: bytes) -> None:
            if self._over_tls:
                self._tls_proto.data_received(data)
            else:
                super().data_received(data)


    async def start_server(loop):
        return await loop.create_server(
            lambda: HttpsServerProtocol(loop), '0.0.0.0', 8082, backlog=65535)


    def make_tls_context():
        ctx = ssl.SSLContext(ssl.PROTOCOL_SSLv23)
        ctx.load_cert_chain('example.com.crt', 'https.key.pem')
        return ctx


    def main():
        loop = asyncio.new_event_loop()
        loop.run_until_complete(start_server(loop))
        loop.run_forever()
        loop.close()

Initially I have `HttpServerProtocol` which receives data stream, parses
HTTP messages and calls `request_received` which returns just a dummy response::

    ➜  ~ curl http://localhost:8082/any/path -v
    > GET /any/path HTTP/1.1
    > User-Agent: curl/7.38.0
    > Host: localhost:8082
    > Accept: */*
    >
    < HTTP/1.1 200 OK
    < Content-Length: 2
    < Connection: close
    <

    OK

Then I have `HttpsServerProtocol` which is just the extension of
`HttpServerProtocol` but also is capable of upgrading connection to TLS.
The way it works is, it overrides `request_received()` method which
checks if HTTP request is `CONNECT`.
If it is, `HttpsServerProtocol` tells `SSLProtocol` to handle a new connection::

    def connect_received(self) -> None:
        self._over_tls = True
        self._tls_proto.connection_made(self._transport)

Then `SSLProtocol` does the TLS handshake and data encryption/decryption::

    ➜  ~ curl --proxy localhost:8082 https://dummy.org -k -v
    > CONNECT dummy.org:443 HTTP/1.1
    > Host: dummy.org:443
    > User-Agent: curl/7.38.0
    > Proxy-Connection: Keep-Alive
    >
    < HTTP/1.1 200 OK
    <
    * Proxy replied OK to CONNECT request
    * successfully set certificate verify locations:
    *   CAfile: none
      CApath: /etc/ssl/certs
    * SSLv3, TLS handshake, Client hello (1):
    * SSLv3, TLS handshake, Server hello (2):
    * SSLv3, TLS handshake, CERT (11):
    * SSLv3, TLS handshake, Server key exchange (12):
    * SSLv3, TLS handshake, Server finished (14):
    * SSLv3, TLS handshake, Client key exchange (16):
    * SSLv3, TLS change cipher, Client hello (1):
    * SSLv3, TLS handshake, Finished (20):
    * SSLv3, TLS change cipher, Client hello (1):
    * SSLv3, TLS handshake, Finished (20):
    * SSL connection using TLSv1.2 / ECDHE-RSA-AES256-GCM-SHA384
    * Server certificate:
    *        subject: CN=dummy.org
    *        start date: 2017-02-02 08:28:42 GMT
    *        expire date: 2017-02-02 08:28:42 GMT
    *        issuer: CN=development ca
    *        SSL certificate verify result: unable to get local issuer certificate (20), continuing anyway.
    > GET / HTTP/1.1
    > User-Agent: curl/7.38.0
    > Host: dummy.org
    > Accept: */*
    >
    < HTTP/1.1 200 OK
    < Content-Length: 2
    < Connection: close
    <
    OK

When `SSLProtocol` receives some data, it decrypts it and passes to
`HttpServerProtocol.data_received()`.
And from now on we're communicating over TLS.
