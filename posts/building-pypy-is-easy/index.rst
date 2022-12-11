.. title: Building PyPy is easy
.. slug: building-pypy-is-easy
.. date: 2017-03-03 07:35:21 UTC+02:00
.. tags: python, pypy
.. category:
.. link:
.. description:
.. type: text

The moment I saw a `benchmark on PyPy 3.5
<https://morepypy.blogspot.lt/2017/03/async-http-benchmarks-on-pypy3.html?spref=tw>`_
I was eager to test it.
PyPy 3.5 support is still in development so I had to build it before trying it.

PyPy is hosted on bitbucket https://bitbucket.org/pypy/pypy and thus requires
mercurial to clone it::

    $ apt install mercurial
    $ hg clone https://bitbucket.org/pypy/pypy

Now let's checkout the `py3.5` branch::

    $ cd pypy
    $ hg up py3.5

Before building the `pypy` I had some missing dependencies on my Debian 8
system::

    $ apt install pypy libbz2-dev libexpat1-dev libffi-dev pkg-config libncurses5-dev

Yes, `pypy` build process requires an older version of `pypy` :D
Although, docs say that `it should be possible to build i with CPython
<http://doc.pypy.org/en/latest/build.html>`_.

Anyway, now we're ready to build the `pypy` and that's as easy as executing::

    $ rpython/bin/rpython -Ojit pypy/goal/targetpypystandalone.py

Eventually, on my computer after 37 minutes, there will be `pypy-c` which
is your Python interpreter/JIT.
