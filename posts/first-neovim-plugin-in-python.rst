.. title: First Neovim plugin in Python
.. slug: first-neovim-plugin-in-python
.. date: 2017-04-16 11:23:23 UTC+03:00
.. tags: neovim,neovim-plugin
.. category:
.. link:
.. description:
.. type: text

For some time I was (and still am) eager to write a Neovim plugin that
would help me draw ASCII diagrams.

Neovim provides msgpack based RPC API which allows to communicate with
editor over TCP/IP, Unix socket or pipe.
Thus writing plugins in any language is possible.
I used Python library implementing the API:
https://github.com/neovim/python-client.

Controlling Neovim Instance
===========================

There are couple of ways how we can connect Python based plugin with
Neovim.
The easiest way is to start Neovim with specified Unix socket::

     $ NVIM_LISTEN_ADDRESS=/tmp/nvim nvim

Now we can connect to Neovim instance from Python:

.. code-block:: python

    import neovim

    vim = neovim.attach('socket', path='/tmp/nvim')
    print(vim.current.line)
    vim.current.line = 'New line'

python-client API
=================

`Neovim python <https://github.com/neovim/python-client>`_ API is not documented.
While playing with it, I found some useful commands::

    vim.current.line
        Currently selected line content.

    vim.eval('line(".")')
        Currently selected line number. Line indexes start from 1.

    vim.eval('col(".")')
        Currently selected column number. Column indexes start from 1.

    vim.current.buffer
        Currently visible Neovim buffer.

    buffer.append(line: str, index: int=-1) -> None
        Inserts specified line to the buffer.

        index - index of a line after which new line will be inserted.
            0 - inserts first line.
            -1 - inserts last line.

Configure Neovim to Load Plugin
===============================

We can tell Neovim to automatically load our Python based plugin.
First of all our plugin must expose implement required API:

.. code-block:: python

    import neovim

    @neovim.plugin
    class TestPlugin(object):
        def __init__(self, vim) -> None:
            self.vim = vim

        @neovim.command('TestCommand')
        def alter_current_line(self) -> None:
            self.vim.current.line = 'New line'

This test plugin defines new Neovim command which simply overwrites current
line.

Now to make Neovim load the plugin:

1. Place it in `~/.config/nvim/rplugin/python3`.
2. Open Neovim and issue `:UpdateRemotePlugins` command.
3. Restart Neovim.

Now select some line, press `ESC` and type `:TestCommand`.
It should overwrite the line with "New Line".
