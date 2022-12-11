.. title: Silence does not mean stagnation
.. slug: silence-does-not-mean-stagnation
.. date: 2018-11-05 21:48:02 UTC+02:00
.. tags: development
.. category:
.. link:
.. description: Summary of my progress over the last year.
.. type: text

It's been over a year since I last wrote something on this blog. At the end of
2016, when I returned from `the Recurse Center <https://recurse.com>`_, I told
myself I will be blogging about all new things I learn or experiment with -
cause that's a good way to organize ones thoughts. Anyway, good habits
require at least 2 things: will and time. I wouldn't say I don't have any of
those but in the end I did not write anything for a whole year!

One of the major factors that kept me silent was my new job. At the end of
October in 2017 I started working with `MaidSafe <https://maidsafe.net>`_ on a
peer-to-peer privacy and security oriented `data network
<https://safenetwork.tech>`_. The peer-to-peer technology was totally new to
me as a programmer. Also, most of our code is written in Rust which I didn't
have professional experience with. On top of that, I started working remotely
and that was another thing I've never done before. So, I've kept myself quite
busy and couldn't find enough time to properly log some of my discoveries :)

Work at MaidSafe
================

One of the nice things about working at MaidSafe is that everything is open
source. You can even go and checkout out what I've been doing `today
<https://github.com/povilasb>`_ :) At MaidSafe I'm mainly working on the p2p
networking library called Crust which puts into use a number of other libraries.
I will mention only those that I've made a fair amount of contributions.

p2p
---

`p2p <https://github.com/ustulation/p2p/>`_ is a Rust library that does TCP and
UDP hole punching. Hole punching is a technique that makes direct connections
possible between two peers that are behind routers with Network Address
Translation. This topic deserves a blog post of it's own that I will probably
write some time soon. p2p is a very important library for peer-to-peer
connections and is integrated into Crust. I made sure the integration was
smooth and was one of the library maintainers.

tokio-utp
---------

`tokio-utp <https://github.com/maidsafe/tokio_utp>`_ is another Rust networking
library. It implements uTP protocol in pure Rust and exposes Tokio/futures
based API. uTP protocol adds reliability logic on top of UDP protocol. It also
describes the expected congestion control mechanism that yields to other
network traffic in the system. I took over the maintenance of this library from
ex developers. I did a lot of investigation on how congestion control was
implemented, I worked on graceful connection shutdown, improved test coverage
and code quality in general.

rust-libutp
-----------

`tokio-utp <https://github.com/maidsafe/tokio_utp>`_ was written from scratch
and at that time it wasn't as stable as people would like it to be. As a
result, I wrote the wrappers for the libutp which is C implementation of uTP
protocol. `libutp <https://github.com/bittorrent/libutp>`_ is the mainstream
implementation and is supposed to be robust and stable. That was the first time
I wrote C bindings for Rust, hence learned something new. I have to say it was
quite easy: Rust has this tool `bindgen
<https://github.com/rust-lang-nursery/rust-bindgen>`_ that automatically
generates Rust wrappers for a given C header file. The generated wrappers are
unsafe so I did have to provide safe API on top of. Anyway, work is still in
progress there.

Crust
-----

Most of my time I spent working on `Crust <https://github.com/maidsafe/crust>`_
library. It's a peer-to-peer communications library written in Rust. It
supports multiple transport protocols (TCP and uTP at this moment), encryption,
hole punching, etc. I did some work to integrate tokio and futures crates into
Crust, greatly expanded automated test suite, implemented examples, expanded
documentation and maintained library overall quality.

I also did some minor changes to MaidSafe encryption libray
`safe_crypto <https://github.com/maidsafe/safe_crypto/commits?author=povilasb>`_
and carried safe_crypto integration into Crust.

netsim
------

Testing peer-to-peer networking code is hard, especially the hole punching
implementation. As a result my collegue `Andrew <https://github.com/canndrew>`_
created a library that simulates computer networks with Rust - `netsim
<https://github.com/canndrew/netsim>`_. It runs completely in memory and gives
you a full control over the simulated network: you can construct hierarchical
networks, exchange IP packets, introduce packet loss and latency, simulate misc
network address translation behaviors, etc.
I mostly tested the library from the user perspective and gave my feedback.
In addition, I did some minor
`contributions <https://github.com/canndrew/netsim/commits?author=povilasb>`_
and since this is such an amazing tool I did a couple of presentations at our
local events in Lithuania: `no trolls allowed 2018
<https://2018.notrollsallowed.com/pranesimai/60>`_ and `Vilnius Rust meetup
<https://www.meetup.com/Rust-in-Vilnius/events/254403141/>`_ :)

Open Source Projects
====================

Besides my daily job I usually tend to spend some time on my personal or
other open source projects. A lot of times the work I do for the companies
influences my interests after work too.

A year ago I did some UDP and TCP hole punching experiments using python.
The code is `here <https://github.com/povilasb/hole-punching>`_.
I also wrote a command line utility to forward ports in your router:
https://github.com/povilasb/pyigd which I actively use to this day.

At MaidSafe I work with Rust on a daily basis where we use a lot of open source
libraries. During the last year I did some small contributions to some of them.
I added boolean matchers to rust hamcrest library:
https://github.com/ujh/hamcrest-rust/commits?author=povilasb . Unfortunately,
the development of hamcrest-rust got frozen. But then an active `fork
<https://github.com/Valloric/hamcrest2-rust>`_ emerged which accepted a small
`PR of mine <https://github.com/Valloric/hamcrest2-rust/pull/4>`_. I made
`contains()` matcher generic which allowed it to accept single element or
vector.  Hence `assert_that!(vec![1, 2, 3], contains(2))` became possible.
Another rust library I contributed to is `rust-igd
<https://github.com/sbstp/rust-igd>`_ which implements `IGD protocol
<https://en.wikipedia.org/wiki/Internet_Gateway_Device_Protocol>`_. I added
`an asyncronous use example <https://github.com/sbstp/rust-igd/pull/33>`_.
Finally, I made a tiny `PR <https://github.com/cespare/vim-toml/pull/41>`_
into `vim-toml <https://github.com/cespare/vim-toml>`_ which is `TOML format
<https://github.com/toml-lang/toml>`_ highlighting for vim. Unfortunately the
PR did not get merged yet.
