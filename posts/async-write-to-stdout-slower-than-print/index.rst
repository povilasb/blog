.. title: Async write to stdout slower than print?
.. slug: async-write-to-stdout-slower-than-print
.. date: 2017-01-04 21:16:55 UTC+02:00
.. tags: python,asyncio
.. category:
.. link:
.. description:
.. type: text

While implementing HTTP benchmarking tool with asyncio, I was playing with
asyncronous writes to stdout - aka async print.
My motivation was to improve performance of course :).

But after running some small benchmarks:

.. code-block:: python

    import asyncio
    import os
    import time

    async def make_stdout(loop):
        writer_transport, writer_protocol = await loop.connect_write_pipe(
            FlowControlMixin, os.fdopen(1, 'wb'))
        return asyncio.StreamWriter(writer_transport, writer_protocol, None, loop)

    async def do_async_print(loop):
        stdout = await make_stdout(loop)
        started = time.time()
        for _ in range(4000):
            stdout.write(b'works works works works works works works works works works\n')
        stdout.write(bytes('Time taken: {}'.format(time.time() - started), 'ascii'))

    def test_async_print():
        loop = asyncio.get_event_loop()
        loop.run_until_complete(do_async_print(loop))
        loop.close()

    def test_sync_print():
        started = time.time()
        for _ in range(4000):
            print('works works works works works works works works works works')
        print('Time taken:', time.time() - started)

Turns out that `test_sync_print()` runs quite faster than `test_async_print()`.
In average 4000 sync prints run in **100 ms**, whereas async print in **150
ms**.

Also, if I go beyond 4000 prints asynchronous method blocks for unknown reasons.
Synchronous works as expected.
