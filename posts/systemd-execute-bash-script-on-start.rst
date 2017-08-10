.. title: systemd: execute bash script on start
.. slug: systemd-execute-bash-script-on-start
.. date: 2017-08-10 14:09:19 UTC+03:00
.. tags: linux,systemd
.. category:
.. link:
.. description:
.. type: text

`systemd` is a not so new Linux init system. It's default on Debian systems.
This is the very first process to start when Linux boots - it's PID is 1::

    $ ps aux
    USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
    root         1  0.0  0.1 139092  6876 ?        Ss   14:03   0:01 /sbin/init

    $ file /sbin/init
    /sbin/init: symbolic link to /lib/systemd/systemd


I've been running systemd for a while.
But thanks to it's System V init script compatibility I've never actually
written any scripts for it.

I wanted to start a simple bash script after linux booted and I decided it's
time to use systemd to take care of business.

Unit file
=========

Unit files describe the services `systemd` manages. Very simple unit file
that starts a bash script could be::

    [Unit]
    Description=Starts some bash script

    [Service]
    WorkingDirectory=/home/povilas/
    Type=forking
    ExecStart=/bin/bash my_script.sh
    KillMode=process

    [Install]
    WantedBy=multi-user.target

`WantedBy` option in the `Install` section means that this unit is wanted by
`multi-user.target` unit. Meaning this unit becomes a dependency of
`multi-user.target` unit.

Install unit file
=================

Save the unit file to `/etc/systemd/system/my_unit.service` and enable it with::

    # systemctl enable my_unit.service

Next time the OS boots, bash script will be executed.
