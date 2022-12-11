.. title: Rust - First Steps
.. slug: rust-first-steps
.. date: 2017-05-08 08:57:26 UTC+03:00
.. tags: rust
.. category:
.. link:
.. description:
.. type: text

I was interested in Rust quite a while but never found spare time to
experiment with it.
Also, recently I found https://www.redox-os.org/ - an OS with Rust based
microkernel. I always thought that it would be nice to customize the OS kernel.
But Linux being a monolithic kernel and written in C sounds like a lot of work
to do any changes.
So, in order to play with Redox I have to know some Rust :)

Hello World
===========

As usual the very first code I wrote was a hello world application:

.. code-block:: rust

    fn main() {
        println!("Hello world");
    }

This reminds me a lot of C: main entry point is called `main`, double
quoted strings and semicolon after the statement.

Variables
=========

In Rust variables are immutable (const in C++) by default:

.. code-block:: rust

    let n: u8 = 8;
    n = 9; // results in error

Mutable variables are marked explicitly:

.. code-block:: rust

    let mut n: u8 = 8;
    n = 9;

Rust uses static typing system but you don't necessarily have to specify
the variable type:

.. code-block:: rust

    let n = 8;

Compiler infers the variable type just like
`C++ <http://en.cppreference.com/w/cpp/language/auto>`_ and
`Swift <https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/TheBasics.html#//apple_ref/doc/uid/TP40014097-CH5-ID322>`_ does.

Strings
=======

String operations are very common in every language. So good string support
can make developer's life a lot easier.

Rust has two string types: `&str` and `String`.
`String` handles UTF-8
I was suprised to see that symbol indexing does not work with `String`:

.. code-block:: rust

    let s: String = "sample string".to_string();
    println!("{}", s[1]) // results in compilation error

And this makes sense: it's not so easy to index UTF-8 strings, because
each symbol has a variable length of bytes. Thus accessing UTF-8 string
character at position `x` has O(n) complexity.
Usually, when we are indexing an array we expect O(1) operation.
So we're better off accessing characters explicitly:

.. code-block:: rust

    "hello".to_string().chars().nth(1).unwrap() // returns 'e'

Rust has a bytes literal identical to Python

.. code-block:: rust

    b"binary data"

Functions
=========

Rust function definitions remind me of `Swift ones
<https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Functions.html>`_
or `Python type hints <http://www.mypy-lang.org/examples.html>`_.

.. code-block:: rust

    fn sum(a: i64, b: i64) -> i64 {
        return a + b;
    }

In Rust almost everything is an expression. Just like in Ruby. So `return`
statement might be omitted:

.. code-block:: rust

    fn sum(a: i64, b: i64) -> i64 {
        a + b
    }

Lambda functions are called `closures
<https://doc.rust-lang.org/book/closures.html>`_ in Rust.
Their syntax reminds me of `Ruby
<http://culttt.com/2015/05/13/what-are-lambdas-in-ruby/>`_:

.. code-block:: rust

    for n in [1, 2, 3].iter().map(|n| n + 1) {
        println!("{}", n); // prints 2, 3, 4
    }

Optional
========

Rust has `optional types <https://doc.rust-lang.org/std/option/>_` which
represent variables that might not have any value.
They seem similar to Swift's optionals:

.. code-block:: rust

    fn div(a: f64, b: f64) -> Option<f64> {
        if b == 0.0 {
            None
        } else {
            Some(a / b)
        }
    }

    println!("{}", div(10.0, 2.0).unwrap());

Also, in case of None I can specify default value:

.. code-block:: rust

    println!("{}", div(10.0, 0.0).unwrap_or(0.0));

Dependency Manager
==================

Rust has a dependency manager, Cargo, which is also a build system.
First of all, this is super cool, because C and C++ doesn't have a widely
adopted dependency manager. Except a couple attempts to implement one:

* https://github.com/biicode/
* https://www.conan.io/

Also, C and C++ have so many build systems that it's easy to get lost:

* `GNU Make <https://www.gnu.org/software/make/>`_
* `Automake + Autoconf <https://www.gnu.org/software/automake/>`_
* `CMake <https://cmake.org/>`_
* `Ninja <https://ninja-build.org/>`_
* `Scons <http://scons.org/>`_

And recently I just found out that Google is building another one - `Bazel
<https://bazel.build/>`_. Which they are using to build Tensorflow...

From time to time I see those used actively, not just listed in Wikipedia.

I like the Zen of Python:

  There should be one-- and preferably only one --obvious way to do it.

And regarding build tools in C, C++ world, this is not the case :/
So although I used Cargo only for two days, I loved the way it works.

Compiler
========

`rustc` - initially implemented in OCaml, later rewritten in Rust itself.
It has some nice features that caught my eye.
It has plugin system: https://doc.rust-lang.org/book/compiler-plugins.html,
which allows us to extend the compilers behavior like manipulate the AST, etc.

Another feature I find really attractive is `attributes
<https://doc.rust-lang.org/book/attributes.html>`_. They allow to
annotate/label definitions. E.g. there is an attribute that labels function
as a test:

.. code-block:: rust

    #[test]
    fn it_works() {
        assert!(2 * 2 == 4);
    }

Then cargo finds those labelled functions and execute them simply running::

    $ cargo test

Also, attributes can be used to select which code to compile for
specific platform.
Compare:

.. code-block:: rust

    #[cfg(target_os = "linux")]
    fn do_stuff() {
        println!("You are running linux!")
    }

    #[cfg(target_os = "windows")]
    fn do_stuff() {
        println!("You are running windows!")
    }

with C++ implementation:

.. code-block:: cpp

    #ifdef __linux
    void do_stuff() {
        std::cout << "Your are running linux!\n";
    }
    #elif _WIN32
    void do_stuff() {
        std::cout << "Your are running windows!\n";
    }
    #endif
