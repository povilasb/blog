.. title: Running Python async function
.. slug: running-python-async-function
.. date: 2016-12-14 07:37:58 UTC+02:00
.. tags: python3,asyncio
.. category:
.. link:
.. description:
.. type: text

Lately I was playing with asyncio and
`aiohttp <https://github.com/KeepSafe/aiohttp>`_ in python 3.5.2.
First time I hit the wall was when I was trying to test async methods:

.. code-block:: python

    async def say_hi():
        print('Hi!')

First of all I did not know how to call `say_hi()` directly without event loop.
Reading `docs <https://docs.python.org/3/library/asyncio-task.html>`_ didn't
really help me.
I already knew I could do:

.. code-block:: python

    loop = asyncio.get_event_loop()
    loop.run_until_complete(say_hi())
    loop.close()

But I was more interested how asyncio event loop triggers this method by
itself.
So I decided to dig into asyncio source code.

Summary
=======

I've found two ways to call async `say_hi()` without event loop.

1. Send something to coroutine:

.. code-block:: python

    cor = say_hi()
    try:
        cor.send(None)
    except StopIteration:
        pass

2. Wrap coroutine in Future:

.. code-block:: python

    future = asyncio.ensure_future(say_hi())
    future._step()

BaseEventLoop
=============

Turns out on Debian `asyncio.get_event_loop()` returns `UnixSelectEventLoop`.
It extends some other event loops:

::

    UnixSelectEventLoop --> BaseSelectorEventLoop --> BaseEventLoop --> AbstractEventLoop

`BaseEventLoop` is the one that interests me, because it implements
`run_until_complete()` method.

`run_until_complete()` calls `self.run_forever()` which calls
`self._run_once()`.

What `_run_once()` essentially does is it gets the handle from the task queue
and calls `handle._run()`:

.. code-block:: python

    class BaseEventLoop(events.AbstractEventLoop):

        def __init__(self):
            self._ready = collections.deque()

        def _run_once(self):
            handle = self._ready.popleft()
            handle._run()

`handle` type is `asyncio.events.Handle`. It's simplified version is:

.. code-block:: python

    class Handle:
        def __init__(self, callback, args, loop):
            self._loop = loop
            self._callback = callback
            self._args = args

    def _run(self):
        try:
            self._callback(*self._args)
        except Exception as exc:
            # ...

Now this is where the call chain started from `run_until_complete()` stops.
And it's clear to me that I have to find the spot where handles are created
and added to the `event_loop._ready` queue (or dequeu to be precise).

How my async function is turned into Handle?
============================================

Let's go back to `loop.run_until_complete()`. Simplified version could be read
as:

.. code-block:: python

    def run_until_complete(self, future):
        future = tasks.ensure_future(future, loop=self)

        try:
            self.run_forever()
        except:
            # ...
        return future.result()

Ok, let's investigate `tasks.ensure_future()`. In my case it does:

.. code-block:: python

    return loop.create_task(coro_or_future)

Which by itself does not do a lot:

.. code-block:: python

    return tasks.Task(coro, loop=self)

But `Task` constructor calls something interesting:

.. code-block:: python

    def __init__(self, coro, *, loop=None):
        super().__init__(loop=loop)
        self._coro = coro
        self._loop.call_soon(self._step)

And `Task._step()` method is even more interesting:

.. code-block:: python

    def _step(self, exc=None):
        # ...
        coro = self._coro
        # ...
        try:
            result = coro.send(None)
        except:
            # ...
        return result

AHA, seems like `coro.send(None)` triggers the async function.
I quickly test it and indeed it works :) :

.. code-block:: python

    say_hi().send(None)

Anyway, let's keep hunting how this coroutine (my async function) is turned
into Handle...

The next interesting function in `Task` contructor is `loop.call_soon()`.
It simply calls `self._call_soon()`.
Which does the rest:

.. code-block:: python

    def _call_soon(self, callback, args):
        if (coroutines.iscoroutine(callback)
        or coroutines.iscoroutinefunction(callback)):
            raise TypeError("coroutines cannot be used with call_soon()")
        self._check_closed()
        handle = events.Handle(callback, args, self)
        if handle._source_traceback:
            del handle._source_traceback[-1]
        self._ready.append(handle)
        return handle

It wraps callback (our async function) into `Handle` and appends it
to the `loop._ready` queue.

So the call graph is something like this::

                            loop.run_until_complete(future)
                                        |
                                        V
                            tasks.ensure_future(future, loop)
                                        |
                                        V
                               loop.create_task(future)
                                        |
                                        V
                               task.Task(future, loop)
                                        |
                                        V
                               loop.call_soon(task._step)
                                        |
                                        V
                               loop._call_soon(task._step)
                                        |
                                        V
                         loop._ready.append(Handle(task._step))
