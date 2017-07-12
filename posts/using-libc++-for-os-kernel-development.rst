.. title: Using libc++ for OS kernel development
.. slug: using-libc++-for-os-kernel-development
.. date: 2017-07-11 12:38:11 UTC+03:00
.. tags: c++, osdev, libc++
.. category:
.. link:
.. description:
.. type: text

Lately I started porting `my hobby OS <https://github.com/povilasb/simple-os>`_
to C++.
So far I've used C style error handling: function return value is interpreted
as error code on failure. Unfortunately this is very inconvenient.
One always has to look up the function documentation to figure out what
values mean errors, etc. I think C++ exceptions are a superior mechanism.
Unfortunately it's not so easy to get them working in kernel environment:
http://wiki.osdev.org/C%2B%2B_Exception_Support.

Lately I have been learning Rust language. And it's error handling looks
consistent and quite easy to use. One of the core classes is `Result
<https://doc.rust-lang.org/std/result/>`_. Basically it's a simple enum
which holds heterogeneous values: one for return value on success, other for
error values.
Sounds like something like this could be easily implemented in C++ too.
A quick google search revealed already existing library:
https://github.com/oktal/result.

`oktal/result` is a single file header only library. Thus it seemed like it
would be easy to integrate into my kernel.
The tricky thing is that it depends on couple of standard libraries:

.. code-block:: cpp

    #include <iostream>
    #include <functional>
    #include <type_traits>

So I thought why not porting the necessary libraries from libc++.

libc++
======

`libc++ <https://libcxx.llvm.org/>`_ is the implementation of C++ standard
library. Usually it is used together with clang compiler.
You can get latest libc++ sources from https://github.com/llvm-mirror/libcxx.
So I thought I'll just copy/paste 2 libraries (`functional` and `type_traits`)
to my own project and I'll remove the `iostream` use, because it depends
on I/O functions which currently are too different in my kernel...
Turns out it's not so easy. `result.h` includes graph roughly looks like this::

                                result.h
                                 |     |
                                 |     +-------+
                                 V             V
                            functional    type_traits
                             |   | |        |     |
           +-----+---+---+---+   | +---+    |     |
           V     |   |   |       V     V    V     V
         tuple   |   |   |  typeinfo  __config   cstddef --+
                 V   |   V                        |        |
           exception | memory                     V        V
                     V                       __nullptr   stddef.h
                  utility

Porting `type_traits` was easy. I've just copied `__nullptr`, `cstddef` and
`__config` from libc++ and made small hacks in `__config`.
I took `stddef.h` from GNU libc. Then `type_traits` just worked.

But I faced more issues, when I started investigating `functional`
library. The problem is that it depends on exceptions which are disabled
in my kernel environment.
From this point it seemed just too much work to continue.

System predefined macros
========================

`__config` in libc++ has such code:

.. code-block:: cpp

    // Need to detect which libc we're using if we're on Linux.
    #if defined(__linux__)
    #include <features.h>
    #endif // defined(__linux__)

`__linux__` is a predefined compiler macro which in this case enables
`features.h` include, if you're compiling a Linux program.
Although, my OS kernel code is not meant to be run on Linux, g++ and clang
still have this macro defined if I'm compiling on Linux.
I can test predefined macros like this::

    $ touch dummy.hh
    $ g++ -dM -E dummy.hh
    #define __unix__ 1
    #define __cpp_binary_literals 201304
    #define __GCC_ATOMIC_CHAR32_T_LOCK_FREE 2
    #define __x86_64 1
    #define __linux 1
    #define __unix 1
    #define __UINT32_MAX__ 0xffffffffU
    #define __linux__ 1
    ...

Obviously this macro causes me problems: I don't have `features.h` for my
system. Thus I want to undefine `__linux__`, `__linux`, etc.

Undefining macros
-----------------

I think superior method to solve macro issues is to create a cross
compiler. E.g. Redox OS has taken this path with it's `gcc port
<https://github.com/redox-os/gcc/commit/37820fd5d9a7c9037a4a1be0816610cbd00ae59d#diff-dbff4af31a2e5a58eeb80832dead0b95R19>`_:

.. code-block:: cpp

    #define TARGET_OS_CPP_BUILTINS()      \
       do {                                \
         builtin_define ("__redox__");      \
         builtin_define ("__unix__");      \
         builtin_assert ("system=redox");   \
         builtin_assert ("system=unix");   \
         builtin_assert ("system=posix");   \
       } while(0);

Unfortunately, cross compiler is too much work for my project.

The other way to workaround macro issues is to use `#undef` directive:

.. code-block:: cpp

    #undef __linux__
    #if defined(__linux__)
    #include <features.h>
    #endif // defined(__linux__)

This was the change I made to `__config`.

Conclusions
===========

Using C++ standard library in kernel environment would definitely save me a
lot of work. Unfortunately, it relies on exceptions and RTTI support.
Thus porting libc++ to current kernel environment is just too much work.
The other approach could be to reimplement standard library that does not use
exceptions.

P.S. https://github.com/electronicarts/EASTL is STL implementation from
Electronic Arts. It provides the ability to disable exceptions.
Unfortunately, in such case no alternative error handling is provided.
