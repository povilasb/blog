.. title: Network Address Translation
.. slug: network-address-translation
.. date: 2019-01-28 21:16:55 UTC+02:00
.. tags: nat,networking
.. category:
.. link:
.. description: How NAT works?
.. type: text

There is a lot of resources on the Internet explaining how NAT works, but I
just want to have a post here that will lay the foundations for the subsequent
post I was meaning to write - `hole
punching <https://en.wikipedia.org/wiki/Hole_punching_%28networking%29>`_.
Network Address Translation (later simply NAT) is a method to connect multiple
LAN devices to the Internet via a single IP address. NAT is implemented by
our home routers. Technically speaking NAT changes source IP and source port
fields of every outgoing UDP/TCP packet and destination IP and port of every
incoming packet.

Example
-------

::

                   +---------------+
                   | Target Server |
                   |    1.1.1.1    |
                   +---------------+
                          ^
                          |                            Packet source IP and
                          |                          / port number are
      +---------------------------------------------+  overwritten by our router
      |                TCP/UDP Packet               |
      +-------------+----------+---------+----------+
      |    SRC_IP   | SRC_PORT | DST_IP  | DST_PORT |
      +-------------+----------+---------+----------+
      | 78.53.1.5   |   7000   | 1.1.1.1 |   80     |
      +-------------+----------+---------+----------+
                          |
                          |
                  +---------------+
                  |   78.53.1.5   | - Router IP assigned by our ISP
                  |               |
                  |     Router    |
                  |               |
                  | 192.168.1.254 | - Router IP on Local Area Network
                  +---------------+
                          ^
                          |                            Our machine on LAN
                          |                          / sends this packet to
      +---------------------------------------------+  the 1.1.1.1 server
      |                TCP/UDP Packet               |
      +-------------+----------+---------+----------+
      |    SRC_IP   | SRC_PORT | DST_IP  | DST_PORT |
      +-------------+----------+---------+----------+
      | 192.168.1.5 |   5000   | 1.1.1.1 |   80     |
      +-------------+----------+---------+----------+
                          ^
                          |
                          |
                    +-------------+
                    |  PC on LAN  |
                    | 192.168.1.5 |
                    +-------------+

NAT Table
---------

The fundamental way NAT works is pretty straightforward:

1. Router keeps a table of port mappings - NAT table. As new packets travel
   through the router this table is updated, e.g.

::

    +----------+-------------+----------+-------------+----------+----------+
    | Protocol |    SRC_IP   | SRC_PORT |    DST_IP   | DST_PORT | NEW_PORT |
    +----------+-------------+----------+-------------+----------+----------+
    |    UDP   | 192.168.1.5 |   5000   |   1.1.1.1   |   80     |  7000    |
    +----------+-------------+----------+-------------+----------+----------+

2. Before packet is sent over the Internet, it's source port will be replaced
   with `NEW_PORT`. So this is the port, the target server will see when the
   packet arrives.

3. When incoming packet is received, router finds a row in the NAT table whose
   `NEW_PORT` matches packet destination port. Then it forwards the packet
   to machine on LAN using `SRC_IP`:`SRC_PORT` information from NAT table.

::

                   +---------------+
                   | Target Server |
                   |    1.1.1.1    |
                   +---------------+
                          |
                          |
                          V
      +-----------------------------------------------+
      |                  TCP/UDP Packet               |
      +-------------+----------+-----------+----------+
      |    SRC_IP   | SRC_PORT | DST_IP    | DST_PORT |
      +-------------+----------+-----------+----------+
      |   1.1.1.1   |    80    | 78.53.1.5 |   7000   |
      +-------------+----------+-----------+----------+
                          |
                          |
                  +---------------+
                  |   78.53.1.5   | - Router IP assigned by our ISP
                  |               |
                  |     Router    |
                  |               |
                  | 192.168.1.254 | - Router IP on Local Area Network
                  +---------------+
                          |
                          |
                          V
      +-------------------------------------------------+
      |                  TCP/UDP Packet                 |
      +-------------+----------+-------------+----------+
      |    SRC_IP   | SRC_PORT |   DST_IP    | DST_PORT |
      +-------------+----------+-------------+----------+
      |   1.1.1.1   |    80    | 192.168.1.5 |   5000   |
      +-------------+----------+-------------+----------+
                          |
                          |
                          V
                    +-------------+
                    |  PC on LAN  |
                    | 192.168.1.5 |
                    +-------------+

NAT types
---------

Depending on how `NEW_PORT` is assigned NATs fall under 2 categories:
EIM (Endpoint Independent Mapping) and EDM (Endpoint Dependent Mapping).

Endpoint independent mapping NAT
================================

EIM NAT [#r1]_ (those abbreviations can get pretty arcane, hugh? :D) assigns
unique `NEW_PORT` for each unique `(Protocol, SRC_IP, SRC_PORT)` tuple. Let's
see an example.

1. Say we have an empty NAT table::

    +----------+-------------+----------+-------------+----------+----------+
    | Protocol |    SRC_IP   | SRC_PORT |    DST_IP   | DST_PORT | NEW_PORT |
    +----------+-------------+----------+-------------+----------+----------+

2. Then we send UDP packet to `1.1.1.1:80` . Network Address Translation kicks off
and does it's job::

    +----------+-------------+----------+-------------+----------+----------+
    | Protocol |    SRC_IP   | SRC_PORT |    DST_IP   | DST_PORT | NEW_PORT |
    +----------+-------------+----------+-------------+----------+----------+
    |    UDP   | 192.168.1.5 |   5000   |   1.1.1.1   |   80     |  7000    |
    +----------+-------------+----------+-------------+----------+----------+

3. We send another UDP packet with the same source IP and port (this is possible
if we use the same UDP socket), but this time to `8.8.8.8:80`. This time NAT
table already has an entry for source endpoint `192.168.1.5:5000`, so another
one won't be added::

    +----------+-------------+----------+-------------+----------+----------+
    | Protocol |    SRC_IP   | SRC_PORT |    DST_IP   | DST_PORT | NEW_PORT |
    +----------+-------------+----------+-------------+----------+----------+
    |    UDP   | 192.168.1.5 |   5000   |   1.1.1.1   |   80     |  7000    |
    +----------+-------------+----------+-------------+----------+----------+

4. We send yet another UDP packet, but this time we use different source IP
and/or port. EIM NAT will add a new port mapping in this case::

    +----------+-------------+----------+-------------+----------+----------+
    | Protocol |    SRC_IP   | SRC_PORT |    DST_IP   | DST_PORT | NEW_PORT |
    +----------+-------------+----------+-------------+----------+----------+
    |    UDP   | 192.168.1.5 |   5000   |   1.1.1.1   |   80     |  7000    |
    |    UDP   | 192.168.1.5 |   6000   |   1.1.1.1   |   80     |  8000    |
    +----------+-------------+----------+-------------+----------+----------+

Endpoint dependent mapping NAT
==============================

It is also known as symmetric NAT [#r2]_. EDM NAT assigns unique `NEW_PORT` for
each unique tuple `(Protocol, SRC_IP, SRC_PORT, DST_IP, DST_PORT)`. Let's
see an example.

1. Let's start with a NAT table containing a single entry::

    +----------+-------------+----------+-------------+----------+----------+
    | Protocol |    SRC_IP   | SRC_PORT |    DST_IP   | DST_PORT | NEW_PORT |
    +----------+-------------+----------+-------------+----------+----------+
    |    UDP   | 192.168.1.5 |   5000   |   1.1.1.1   |   80     |  7000    |
    +----------+-------------+----------+-------------+----------+----------+

2. We send another UDP packet with the same source IP and port to different target
server `8.8.8.8:80`. Since NAT table does not have an entry to this endpoing,
new port mapping will be added:::

    +----------+-------------+----------+-------------+----------+----------+
    | Protocol |    SRC_IP   | SRC_PORT |    DST_IP   | DST_PORT | NEW_PORT |
    +----------+-------------+----------+-------------+----------+----------+
    |    UDP   | 192.168.1.5 |   5000   |   1.1.1.1   |   80     |  7000    |
    |    UDP   | 192.168.1.5 |   5000   |   8.8.8.8   |   80     |  8000    |
    +----------+-------------+----------+-------------+----------+----------+

3. Say we send another packet, this time to the same IP address, but different
port. Even in this case new port mapping will be added::

    +----------+-------------+----------+-------------+----------+----------+
    | Protocol |    SRC_IP   | SRC_PORT |    DST_IP   | DST_PORT | NEW_PORT |
    +----------+-------------+----------+-------------+----------+----------+
    |    UDP   | 192.168.1.5 |   5000   |   1.1.1.1   |   80     |  7000    |
    |    UDP   | 192.168.1.5 |   5000   |   8.8.8.8   |   80     |  8000    |
    |    UDP   | 192.168.1.5 |   5000   |   8.8.8.8   |   443    |  9000    |
    +----------+-------------+----------+-------------+----------+----------+

NAT filtering types
-------------------

NATs can be grouped into different types based on how they filter incoming
packets [#r3]:

* full cone NAT
* port restricted cone NAT

Full cone NAT
=============

This is the least restrictive type of NAT. Full cone NAT allows any incoming
packet whose destination port matches some `NEW_PORT` in NAT table.

E.g. say we have such NAT table::

    +----------+-------------+----------+-------------+----------+----------+
    | Protocol |    SRC_IP   | SRC_PORT |    DST_IP   | DST_PORT | NEW_PORT |
    +----------+-------------+----------+-------------+----------+----------+
    |    UDP   | 192.168.1.5 |   5000   |   1.1.1.1   |   80     |  7000    |
    |    UDP   | 192.168.1.8 |   5000   |   8.8.8.8   |   80     |  8000    |
    +----------+-------------+----------+-------------+----------+----------+

Then the packet arrives to our router::

      +-----------------------------------------------+
      |                  UDP Packet                   |
      +-------------+----------+-----------+----------+
      |    SRC_IP   | SRC_PORT | DST_IP    | DST_PORT |
      +-------------+----------+-----------+----------+
      |   1.1.1.1   |    80    | 78.53.1.5 |   7000   |
      +-------------+----------+-----------+----------+
                          |
                          |
                          V
                  +---------------+
                  |   78.53.1.5   |
                  |               |
                  |     Router    |
                  |               |
                  | 192.168.1.254 |
                  +---------------+
                         |
                         | ----- Packet is forwarded to 192.168.1.5
                         V

The destination port of a packet (`7000`) is matched with `NEW_PORT` in
router's NAT table. Since we find an entry, the packet is forwarded.
Let's take another example. Say we have the same NAT table, but the incoming
packet is::

      +-----------------------------------------------+
      |                  UDP Packet                   |
      +-------------+----------+-----------+----------+
      |    SRC_IP   | SRC_PORT | DST_IP    | DST_PORT |
      +-------------+----------+-----------+----------+
      |   1.1.1.1   |    80    | 78.53.1.5 |   6000   |
      +-------------+----------+-----------+----------+
                          |
                          |
                          V
                  +---------------+
                  |   78.53.1.5   |
                  |               |
                  |     Router    |
                  |               |
                  | 192.168.1.254 |
                  +---------------+
                         |
                         X ----- Packet is dropped
                         V

This packet will be discarded since its destination port `6000` does not
exist in NAT under `NEW_PORT` column.

Finally, I'd like to give another example::

      +-----------------------------------------------+
      |                  UDP Packet                   |
      +-------------+----------+-----------+----------+
      |    SRC_IP   | SRC_PORT | DST_IP    | DST_PORT |
      +-------------+----------+-----------+----------+
      | 86.100.4.5  |  87634   | 78.53.1.5 |   8000   |
      +-------------+----------+-----------+----------+
                          |
                          |
                          V
                  +---------------+
                  |   78.53.1.5   |
                  |               |
                  |     Router    |
                  |               |
                  | 192.168.1.254 |
                  +---------------+
                         |
                         | ----- Packet is forwarded to 192.168.1.5
                         V

Notice how this packet is forwarded even though it's originating from a
random source machine. This is full cone NAT in a nutshell - it allows packets
go through as long as their destination port is inside NAT table.

Port restricted cone NAT
------------------------

This is the most restrictive NAT. It allows incoming packets only from
endpoints that we sent the packet to before. That means

1. destination port of a packet must match `NEW_PORT` in NAT table;
2. source IP and port of a packet must match destination IP and port of the
   same row in NAT table.

So say we have a NAT table::

    +----------+-------------+----------+-------------+----------+----------+
    | Protocol |    SRC_IP   | SRC_PORT |    DST_IP   | DST_PORT | NEW_PORT |
    +----------+-------------+----------+-------------+----------+----------+
    |    UDP   | 192.168.1.5 |   5000   |   1.1.1.1   |   80     |  7000    |
    +----------+-------------+----------+-------------+----------+----------+

Then UDP packet with `(SRC_IP=1.1.1.1, SRC_PORT=80, DST_IP=78.53.1.5, DST_PORT=7000)`
will be allowed, but these packets will be dropped:

* `(SRC_IP=1.1.1.1, SRC_PORT=80, DST_IP=78.53.1.5, DST_PORT=6000)`
* `(SRC_IP=1.1.1.1, SRC_PORT=443, DST_IP=78.53.1.5, DST_PORT=7000)`
* `(SRC_IP=8.8.8.8, SRC_PORT=80, DST_IP=78.53.1.5, DST_PORT=7000)`

Glossary
--------

endpoint - IP:port pair.

.. rubric:: References

.. [#r1] https://docs.rs/p2p/0.6.0/p2p/#endpoint-independent-mapping-eim
.. [#r2] https://www.think-like-a-computer.com/2011/09/19/symmetric-nat/
.. [#r3] https://www.think-like-a-computer.com/2011/09/16/types-of-nat
