.. title: Linux virtual network devices
.. slug: linux-virtual-network-devices
.. date: 2017-08-22 21:28:32 UTC+03:00
.. tags: linux,tun,tap,networking,python
.. category:
.. link:
.. description:
.. type: text

Linux allows us to create virtual network devices and control them
programmaticaly. We can read and produce raw IP or Ethernet packets.
Such devices are called TUN or TAP and often referred to as TUN/TAP.
TUN device is used to manipulate IP packets, TAP - Ethernet [#f1]_.

TUN/TAP has a lot of uses: we can inspect, modify, generate, etc. network
packets. OpenVPN uses TUN/TAP to route all packets through proxy server.
Thus we can use TUN/TAP to create misc VPN services, e.g. IP over DNS [#f2]_
[#f3]_.

Overview
========

::

    +------+              +------------+          +------+
    | eth0 |  <-------->  | Networking | <------> | tun0 |
    +------+              |    stack   |          +------+
                          +------------+             ^
                                                     |
                                                     |
                                                     V
                                              +-------------+
                                              | Application |
                                              +-------------+

`tun0` is virtual network device interface. It acts just like a regular
interface. Except we can hook to it and control it from userspace application.

When application writes packets to `tun0`, they will be put to networking
stack and treated as if they came from a regular NIC.
When packets arrive to networking stack with destination address that is
routed to `tun0`, they will be forwarded to userspace application.

Create TUN device from CLI
==========================

We can use `ip` CLI command to setup the TUN device::

    # ip tuntap add mode tun tun0
    # ip addr add 10.0.0.1/24 dev tun0
    # ip link set tun0 up

Then the resulting routing table looks like::

    # route -n
    Kernel IP routing table
    Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
    0.0.0.0         192.168.0.1     0.0.0.0         UG    600    0        0 wlp3s0
    10.0.0.0        0.0.0.0         255.255.255.0   U     0      0        0 tun0

Which means that the packets with destination `10.0.0.X` will be forwarded
to `tun0` interface.

Create TUN device with Python
=============================

The syscalls used by `ip tuntap` command might be called programmatically from
any language including Python.
To create TUN device we need to open `/dev/net/tun` and call `ioctl()
<https://docs.python.org/3/library/fcntl.html#fcntl.ioctl>`_ with specific
parameters:

.. code-block:: python

    import os
    from fcntl import ioctl
    import struct

    TUNSETIFF = 0x400454ca
    IFF_TUN   = 0x0001
    IFF_NO_PI = 0x1000

    ftun = os.open("/dev/net/tun", os.O_RDWR)
    ioctl(ftun, TUNSETIFF, struct.pack("16sH", b"tun0", IFF_TUN | IFF_NO_PI))

However, this is pretty low level and suits my learning needs very well.
Although, we can definitely find some python libraries for this [#f5]_ [#f6]_.

Also, we can programmatically set up routes using `Netlink
<https://en.wikipedia.org/wiki/Netlink>`_ based protocols.
Fortunately there's a python package `pyroute2
<https://pypi.python.org/pypi/pyroute2>`_ implementing Netlink.
For example we can assign an IP address to TUN interface and bring up:

.. code-block:: python

    # pip install pyroute2==0.4.19
    from pyroute2 import IPRoute

    ip = IPRoute()
    idx = ip.link_lookup(ifname='tun0')[0]
    ip.addr('add', index=idx, address='10.0.0.1', prefixlen=24)
    ip.link('set', index=idx, state='up')
    ip.close()

Controlling TUN/TAP device
==========================

Once we have created and enabled `tun0` interface we can receive and send
raw IP or Ethernet packets, respectively.

Receiving packets
-----------------

Let's start sending ICMP requests to `10.0.0.4` which gets routed to our
TUN device::

    $ route -n
    Kernel IP routing table
    Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
    0.0.0.0         192.168.0.1     0.0.0.0         UG    600    0        0 wlp3s0
    10.0.0.0        0.0.0.0         255.255.255.0   U     0      0        0 tun0
    192.168.0.0     0.0.0.0         255.255.255.0   U     600    0        0 wlp3s0

    $ ping 10.0.0.4

Then we can receive those packets with a simple read:

.. code-block:: python

    import os

    while True:
        raw_packet = os.read(ftun, 1500) # we get ftun descriptor by opening /dev/net/tun
        print(raw_packet)

The output is something like::

    b'E\x00\x00T\xef]@\x00@\x017G\n\x00\x00\x01\n\x00\x00\x04\x08\x00M\xef%\xc2\x00\x05As\x9dY\x00\x00\x00\x00\xe1\xa9\x05\x00\x00\x00\x00\x00\x10\x11\x12\x13\x14\x15\x16\x17\x18\x19\x1a\x1b\x1c\x1d\x1e\x1f !"#$%&\'()*+,-./01234567

which is a raw IP packet with ICMP packet as data.

By the way, seems like Linux kernel is sending `SSDP
<https://en.wikipedia.org/wiki/Simple_Service_Discovery_Protocol>`_ packets
to the TUN interface. So don't get suprised to see some unexpected traffic.

Sending packets
---------------

To send raw IP packets we write them to TUN interface:

.. code-block:: python

    import os

    icmp_req = b'E\x00\x00(\x00\x00\x00\x00@\x01`\xc2\n\x00\x00\x04\x08\x08'\
        '\x08\x08\x08\x00\x0f\xaa\x00{\x00\x01\x00\x00\x00\x00\x00\x00\x00\x00test'
    os.write(ftun, icmp_req)

By the way, we can use `pypacker <https://github.com/mike01/pypacker>`_ to
construct and parse raw packets.

Prerequisites
=============

* To use TUN/TAP devices python scripts must be run with root permissions.
* To forward packets from TUN/TAP to other interfaces (`eth0`), packet forwarding
  must be enabled::

    # echo 1 > /proc/sys/net/ipv4/ip_forward
    # iptables -P FORWARD ACCEPT

* To properly route outgoing packets NAT must be enabled::

  # iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -j MASQUERADE

.. rubric:: References

.. [#f1] https://www.kernel.org/doc/Documentation/networking/tuntap.txt
.. [#f2] http://code.kryo.se/iodine/
.. [#f3] http://cs.brown.edu/courses/cs168/s11/handouts/dtun.pdf
.. [#f4] http://backreference.org/2010/03/26/tuntap-interface-tutorial/
.. [#f5] https://github.com/montag451/pytun
.. [#f6] https://github.com/Gawen/pytun
