.. title: Compiling programs for RedoxOS
.. slug: compiling-programs-for-redoxos
.. date: 2017-06-14 08:10:28 UTC+03:00
.. tags: redox,builds
.. category:
.. link:
.. description:
.. type: text

This writting is based on
https://github.com/redox-os/redox/tree/0eba99659e6ef9d2184e64eaee045c4a720132db
version.

Currently it's quite easy to get your program to be compiled and shipped
together with Redox OS.
All you need to do is:

1. list your program in `filesystem.toml`
2. create `recipe.sh` in `cookbook/recipes/hello-world`::

    $ mkdir cookbook/recipes/hello-world
    $ echo "GIT=https://github.com/povilasb/redox-hello-world" > cookbook/recipes/hello-world/recipe.sh

3. build redox::

   $ make
   $ make qemu

The problem with this approach is that every time I make a change to my program
I have to recompile the whole OS and restart qemu.
It takes time!

I cannot compile rust programs inside Redox yet. Although the progress is
there: https://www.reddit.com/r/Redox/comments/6h4qm9/rustc/
Thus what I want is to cross compile my program and ship it to Redox OS.

Compiling Redox programs
========================

Redox build system is based on so called `cookbook
<https://github.com/redox-os/cookbook>`_. Which basically is a bunch of
shell scripts.
I can use this cookbook to build my programs for Redox.

Let's say my program source code is located in
`/home/povilas/projects/redox-hello-world` and Redox in
`/home/povilas/projects/redox`.
I have to register my program in cookbook::

    $ cd ~/projects/redox/cookbook
    $ mkdir recipes/hello-world
    $ echo "GIT=file:///home/povilas/projects/redox-hello-world" > recipes/hello-world/recipe.sh

Building for the first time
---------------------------

There's a slight difference when I'm building for the first and N-th time.
In case when the program was never built I have to initiate it inside the
cookbook::

    $ cd ~/projects/redox/cookbook
    $ ./repo.sh hello-world

This command will fetch the sources from git and build them.

Rebuilding
----------

When the program was already built, I have to use another script from cookbook::

    $ cd ~/projects/redox/cookbook
    $ ./cook.sh hello-world build

One issue with this approach is that if I don't commit my changes to
`hello-world` repo, they will not be refetched by `cook.sh`.
A workaround is to create a symbolic link to `hello-world` program sources
in cookbook build dir::

    $ cd ~/projects/redox/cookbook/recipes/hello-world/build
    $ ln -s ~/projects/redox-hello-world/src src

Now "`./cook.sh hello-world build`" will always rebuild the changes.

Transfer program to Redox
=========================

When I develop and experiment, I run Redox inside qemu.
Currently the easiest way to transfer the built program to Redox is
to `wget` it from Redox itself.

1. start HTTP server in you host machine::

    $ cd ~/projects/redox/cookbook/recipes/hello-world/build/target/x86_64-unknown-redox/release
    $ python3 -m http.server

2. download the built program inside Redox::

   $ wget http://10.0.2.2:8000/hello_world hello_world
   $ chmod 0755 hello_world
   $ ./hello_world

`10.0.2.2` is the default gateway address in qemu
(http://wiki.qemu.org/Documentation/Networking#User_Networking_.28SLIRP.29).
`8000` is default port for python based HTTP server.
