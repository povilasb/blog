.. title: Linux network routing table
.. slug: linux-network-routing-table
.. date: 2017-08-28 16:35:53 UTC+03:00
.. tags: linux,networking
.. category:
.. link:
.. description:
.. type: text

Our computers can have multiple network devices: Ethernet, WiFi cards,
virtual network devices, etc.
On Linux when the network packets arrive to one of these devices, these
packets are handled by device drivers and then put to networking stack.
Which then forwards packets to appropriate application to handle.

When the packets are sent, they are put to networking stack, e.g. via
BSD sockets API.
Then Linux networking stack uses a routing table to decide which network
interface the packet will be sent to.

Routing table can be viewed with::

    $ route -n
    Kernel IP routing table
    Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
    0.0.0.0         192.168.1.1     0.0.0.0         UG    600    0        0 wlp3s0
    0.0.0.0         0.0.0.0         0.0.0.0         U     1002   0        0 enp0s25
    10.0.0.0        0.0.0.0         255.255.255.0   U     0      0        0 tun0
    192.168.1.0     0.0.0.0         255.255.255.0   U     600    0        0 wlp3s0

or ::

    $ ip route show
    default via 192.168.1.1 dev wlp3s0 proto static metric 600
    default dev enp0s25 scope link metric 1002 linkdown
    10.0.0.0/24 dev tun0 proto kernel scope link src 10.0.0.1
    192.168.1.0/24 dev wlp3s0 proto kernel scope link src 192.168.1.125 metric 600

Although `route` command is deprecated [#f1]_, I still prefer it because of
nicer output.

Example
=======

Let's take the output of `route -n`.
Now what happens when we execute `ping 8.8.8.8`?

Well, Linux kernel

1. matches address `8.8.8.8` with `0.0.0.0` (all addresses),
2. looks up the gateway which is `192.168.1.1`,
3. matches gateway address with `192.168.1.0`,
4. looks up that destination `192.168.1.0` does not have gateway meaning
   that the network is directly connected somehow,
5. does NAT (network address translation),
6. writes packet to `wlp3s0` interface,
7. device driver sends packet to WiFi card.

Understanding routing table
===========================

Again, let's look at `route -n` output.

`Genmask` column is a mask applied (bitwise AND) to IP packet destination
address.
After `Genmask` is applied, `Destination` column is used to match packet
destination.
If the matched destination has `Gateway`, packet is forwarded there,
otherwise it's written to network interface described by column `Iface`.

When packet destination address matches with multiple `Destination` entries,
the one with the longest prefix wins.
For example address `192.168.1.100` matches `0.0.0.0/0` and `192.168.1.0/24`.
But `192.168.1.0` prefix is 24 bits long, so this rule takes priority.

`Flags` column holds some metainfo about routes: `U` - route is up, `G` route
has gateway.

`Ref` and `Metric` columns are not used in Linux kernel.

Manipulating routing table
==========================

We can use `ip` command to add, delete, modify routes.
E.g. we can redirect packets to desired network interfaces::

    # ip route add 8.8.8.8 via 10.0.0.2

This makes all packets with destination `8.8.8.8` be written to
:doc:`tun0 interface <linux-virtual-network-devices>`.

To delete route issue command::

    # ip route del 8.8.8.8 via 10.0.0.2

We can test which `Destination` entry will be matched for given address::

    $ ip route get 10.0.0.2
    10.0.0.2 dev tun0 src 10.0.0.1

    $ ip route get 8.8.8.8
    8.8.8.8 via 192.168.1.1 dev wlp3s0 src 192.168.1.125

    $ ip route get 192.168.1.100
    192.168.1.100 dev wlp3s0 src 192.168.1.125

The difference between the last two routes is that NAT will be executed
for packets destined to `8.8.8.8`, while packets with destination
`192.168.1.100` will remain intact.

.. rubric:: References

.. [#f1] http://xmodulo.com/linux-tcpip-networking-net-tools-iproute2.html
.. [#f2] https://www.cyberciti.biz/faq/what-is-a-routing-table/
