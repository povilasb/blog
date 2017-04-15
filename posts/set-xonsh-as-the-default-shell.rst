.. title: Set xonsh as the default shell
.. slug: set-xonsh-as-the-default-shell
.. date: 2016-10-26 09:33:01 UTC+03:00
.. tags: linux,xonsh
.. category:
.. link:
.. description:
.. type: text

Before we can change the shell it has to be added to `/etc/shells`.

::

    $ which xonsh | xargs echo >> /etc/shells

Then we use the `chsh` which stands for change shell::

    $ chsh -s `which xonsh`
