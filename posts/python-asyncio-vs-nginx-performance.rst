.. title: Python asyncio vs nginx performance
.. slug: python-asyncio-vs-nginx-performance
.. date: 2017-02-24 17:23:38 UTC+02:00
.. tags: python, asyncio, tls
.. category:
.. link:
.. description:
.. type: text

While I was playing with Python asyncio I got interested in how well
it performs serving data over TLS compared to Nginx.
So I implemented a small HTTPS server with asyncio:

.. code-block:: python

    import asyncio
    import ssl
    import random
    import string

    import uvloop


    content_20k = ''.join(random.choice(string.ascii_uppercase + string.digits)
                          for _ in range(20 * 1024))
    resp_lines = [
        'HTTP/1.1 200 OK',
        'Connection: close',
        '',
        ''
    ]
    resp_20k = bytes('\r\n'.join(resp_lines) + content_20k, 'latin-1')


    class HttpServerProtocol(asyncio.Protocol):
        def __init__(self) -> None:
            super().__init__()
            self._transport = None

        def request_received(self) -> None:
            self._transport.write(bytes(resp_20k))
            self._transport.close()

        def data_received(self, data: bytes) -> None:
            # Let's assume request is complete.
            self.request_received()

        def connection_made(self, transport) -> None:
            self._transport = transport

        def close(self):
            self._transport.close()

    async def start_https_server(loop):
        tls_ctx = make_tls_context()
        return await loop.create_server(
            lambda: asyncio.sslproto.SSLProtocol(
                loop, HttpServerProtocol(), tls_ctx, None, server_side=True),
            '0.0.0.0', 8082, backlog=65535
        )

    def make_tls_context():
        ctx = ssl.SSLContext(ssl.PROTOCOL_TLS)
        ctx.load_cert_chain(
            certfile='example.com.crt',
            keyfile='https.key.pem',
        )
        return ctx

    def main():
        asyncio.set_event_loop_policy(uvloop.EventLoopPolicy())
        loop = asyncio.new_event_loop()
        loop.run_until_complete(start_https_server(loop))
        loop.run_forever()
        loop.close()

    if __name__ == '__main__':
        main()

All it does is simply serves a static 20k content over TLS.
I configure nginx to do the same.

Then I ran a small benchmark with `httpmeter
<http://github.com/povilasb/httpmeter>`_. I am doing one request per connection
as I am mainly interested in TLS performance.
Here's the results for asyncio based server::

    $ httpmeter -c 1000 -n 3000 -H 'Connection: close' https://192.168.2.222:8082/
    Concurrency Level:        1000
    Completed Requests:       3000
    Requests Per Second:      544.186120 [#/sec] (mean)
    Request Durations:        [min: 0.147766, avg: 1.016474, max: 1.847824] seconds
    Document Length:          [min: 20480, avg: 20480.000000, max: 20480] bytes
    Status codes:
            200 3000

And here's for nginx::

    $ httpmeter -c 1000 -n 3000 -H 'Connection: close' https://192.168.2.222:8443/20k
    Concurrency Level:        1000
    Completed Requests:       3000
    Requests Per Second:      1210.957884 [#/sec] (mean)
    Request Durations:        [min: 0.068251, avg: 0.454704, max: 0.834079] seconds
    Document Length:          [min: 20480, avg: 20480.000000, max: 20480] bytes
    Status codes:
            200 3000

Both nginx and python server loaded up to 100% single CPU core.

Basically nginx performed 2x better than asyncio.

I profiled the asyncio server::

    $ pyenv/bin/python server.py
         361637 function calls (361625 primitive calls) in 9.351 seconds

    Ordered by: internal time

    ncalls  tottime  percall  cumtime  percall filename:lineno(function)
     9000    4.770    0.001    4.770    0.001 {method 'do_handshake' of '_ssl._SSLSocket' objects}
        1    4.088    4.088    9.349    9.349 {method 'run_forever' of 'uvloop.loop.Loop' objects}
    15000    0.076    0.000    4.975    0.000 sslproto.py:169(feed_ssldata)
     3000    0.068    0.000    0.068    0.000 {method 'write' of '_ssl._SSLSocket' objects}
     9000    0.045    0.000    0.045    0.000 {method 'read' of '_ssl._SSLSocket' objects}
    12000    0.033    0.000    0.221    0.000 sslproto.py:624(_process_write_backlog)
     3000    0.024    0.000    0.025    0.000 sslproto.py:413(__init__)
     9000    0.018    0.000    5.107    0.001 sslproto.py:497(data_received)
     3000    0.017    0.000    0.035    0.000 sslproto.py:572(_on_handshake_complete)
     3000    0.015    0.000    0.015    0.000 {method '_wrap_bio' of '_ssl._SSLContext' objects}
    12000    0.013    0.000    0.013    0.000 {method 'read' of '_ssl.MemoryBIO' objects}
     3000    0.012    0.000    0.012    0.000 {method 'shutdown' of '_ssl._SSLSocket' objects}
     3000    0.012    0.000    0.091    0.000 sslproto.py:243(feed_appdata)
     9000    0.010    0.000    4.782    0.001 ssl.py:681(do_handshake)
     3000    0.009    0.000    0.019    0.000 sslproto.py:460(connection_made)
    12000    0.008    0.000    0.008    0.000 {method 'write' of 'uvloop.loop.UVStream' objects}
     6000    0.007    0.000    0.155    0.000 sslproto.py:555(_write_appdata)
     9000    0.007    0.000    0.007    0.000 {method 'write' of '_ssl.MemoryBIO' objects}
     3000    0.006    0.000    0.061    0.000 sslproto.py:118(do_handshake)
     3000    0.006    0.000    0.168    0.000 server.py:25(request_received)
     3000    0.006    0.000    0.008    0.000 sslproto.py:521(eof_received)
    48045    0.005    0.000    0.005    0.000 {built-in method builtins.len}
     3000    0.005    0.000    0.005    0.000 sslproto.py:68(__init__)
     3000    0.005    0.000    0.034    0.000 server.py:43(<lambda>)
     9000    0.005    0.000    0.050    0.000 ssl.py:618(read)
     9000    0.005    0.000    0.005    0.000 {method 'call_soon' of 'uvloop.loop.Loop' objects}
    15018    0.005    0.000    0.005    0.000 {built-in method builtins.getattr}
     3000    0.004    0.000    0.009    0.000 weakref.py:155(__setitem__)
     3000    0.004    0.000    0.030    0.000 sslproto.py:139(shutdown)
     3000    0.004    0.000    0.021    0.000 ssl.py:403(wrap_bio)
     3000    0.004    0.000    0.006    0.000 sslproto.py:471(connection_lost)
     3000    0.003    0.000    0.114    0.000 sslproto.py:379(write)
     3000    0.003    0.000    0.003    0.000 server.py:21(__init__)
     3000    0.003    0.000    0.005    0.000 sslproto.py:560(_start_handshake)
     3000    0.003    0.000    0.003    0.000 {method 'cipher' of '_ssl._SSLSocket' objects}
     3000    0.003    0.000    0.003    0.000 {method 'update' of 'dict' objects}
    15028    0.002    0.000    0.002    0.000 {method 'append' of 'list' objects}
     3000    0.002    0.000    0.004    0.000 ssl.py:638(getpeercert)
     3000    0.002    0.000    0.002    0.000 weakref.py:310(__init__)
     3000    0.002    0.000    0.002    0.000 ssl.py:577(__init__)
     3000    0.002    0.000    0.048    0.000 sslproto.py:317(close)
     3000    0.002    0.000    0.002    0.000 {method 'peer_certificate' of '_ssl._SSLSocket' objects}
     3000    0.002    0.000    0.014    0.000 ssl.py:690(unwrap)
     9000    0.002    0.000    0.002    0.000 sslproto.py:450(_wakeup_waiter)
     3000    0.002    0.000    0.046    0.000 sslproto.py:549(_start_shutdown)
     3000    0.001    0.000    0.003    0.000 weakref.py:305(__new__)
     3000    0.001    0.000    0.001    0.000 {method 'close' of 'uvloop.loop.UVBaseTransport' objects}
     3000    0.001    0.000    0.001    0.000 ssl.py:584(context)
     3000    0.001    0.000    0.170    0.000 server.py:29(data_received)
     1454    0.001    0.000    0.001    0.000 weakref.py:108(remove)
     3002    0.001    0.000    0.001    0.000 {built-in method __new__ of type object at 0x88d200}
     3000    0.001    0.000    0.069    0.000 ssl.py:630(write)
     3000    0.001    0.000    0.001    0.000 sslproto.py:297(__init__)
     9000    0.001    0.000    0.001    0.000 {method 'get_debug' of 'uvloop.loop.Loop' objects}
     3000    0.001    0.000    0.004    0.000 ssl.py:661(cipher)
     3000    0.001    0.000    0.002    0.000 ssl.py:672(compression)
     9000    0.001    0.000    0.001    0.000 {method 'append' of 'collections.deque' objects}
     3000    0.001    0.000    0.001    0.000 server.py:33(connection_made)
     3018    0.001    0.000    0.001    0.000 {built-in method builtins.hasattr}
     3022    0.001    0.000    0.001    0.000 {built-in method builtins.isinstance}
     3000    0.001    0.000    0.001    0.000 sslproto.py:95(ssl_object)
     3000    0.000    0.000    0.000    0.000 {method 'compression' of '_ssl._SSLSocket' objects}
     3000    0.000    0.000    0.000    0.000 protocols.py:94(eof_received)
        2    0.000    0.000    0.000    0.000 sre_compile.py:250(_optimize_charset)
     3000    0.000    0.000    0.000    0.000 protocols.py:25(connection_lost)
     2397    0.000    0.000    0.000    0.000 sslproto.py:332(__del__)

It turns out that most of the time is spent doing TLS handshake.
This is weird because I expected TLS to perform similar to nginx because
python is only wrapping OpenSSL C implementation.

One of the possible improvements that nginx could have is TLS session
resumption.
But it's not possible in this case because I used `httpmeter` which is
implemented in python.
And by the time I ran benchmarks, python TLS API did not support TLS sessions
(https://bugs.python.org/issue19500).

So I need to do more research to better understand nginx over python asyncio
TLS performance.
