.. title: SSH Agent Forwarding
.. slug: ssh-agent-forwarding
.. date: 2017-06-07 22:10:07 UTC+03:00
.. tags: linux,ssh
.. category:
.. link:
.. description:
.. type: text

Sometimes I want to use my SSH keys from remote server.
But for security reasons copying SSH keys is a bad idea.
Fortunately there's a better way to do this - ssh-agent daemon.
`ssh-agent` holds privates keys in memory and allows to authenticate over network.
When I `ssh` to remote server, I can forward `ssh-agent` connection.
Thus remote server will be using my local `ssh-agent` to authenticate connections.

However, every time I want to forward ssh agent, I can't remember how.

Actually it's as simple as::

    eval `ssh-agent -s`
    ssh-agent
    ssh -A povilas@example.com
