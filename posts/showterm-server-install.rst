.. title: Showterm server installation (Debian, Nginx, PostgreSQL)
.. slug: showterm-server-installation
.. date: 2014-08-18 09:35 UTC+02:00
.. tags: linux,showterm.,li
.. category:
.. link:
.. description:
.. type: text

Showterm is a set of open source applications for terminal I/O recording and
playback. It consists of client and server applications. Both of them are
hosted on github. You can find easy instructions how to install showterm client
on http://showterm.io/. Unfortunately, there are no instructions how to install
the server software.

This tutorial demonstrates how to install showterm server on debian systems
running Nginx web server and PostgreSQL database.


.. note:: I've also successfully tested the server together with MySQL database.


About Showterm server
=====================

Showterm server is `Ruby on Rails <http://rubyonrails.org/>`_ application.
Showterm stores it's "videos" in a database. Specific database might be selected
by editind the *database.yml* config file. This should be straighforward for
Ruby on Rails developers. For others like me we'll see later how to do this.

.. raw:: html

    <iframe width="100%" height="480"
        src="http://show.povilasb.com/f1cba221048ef40ce8222#fast" ></iframe>


Setup PostgreSQL
================

1. Get the necessary Debian packages::

        $ sudo apt-get install postgresql-9.1 postgresql-server-dev-9.1

2. Create PostgreSQL user and database dedicated to showterm server::

        $ su postgres
        $ psql
        postgres=# create database showterm;
        postgres=# create user showterm with password 'showterm';
        postgres=# grant all privileges on database "showterm" to showterm;
        postgres=# \q

Above instructions create database "showterm", user "showterm" with password
"showterm".

3. Update PostgreSQL `authentification settings
   <http://www.postgresql.org/docs/9.1/static/auth-methods.html>`_ to allow
   Ruby on Rails to connect to *showterm* database.
   Authetification configs are located in
   */etc/postgresql/9.1/main/pg_hba.conf*. Append new entry::

        # TYPE  DATABASE        USER       ADDRESS        METHOD
        local   showterm        showterm                  md5

Now restart the computer to apply the new settings.


Install Showterm server
=======================

1. Get necessary Debian packages::

        $ # Add unstable repo, it has ruby 2.0 packages.
        $ sudo echo "deb ftp://ftp.debian.org/debian/ unstable main contrib non-free" /etc/apt/sources.list.d/unstable.list
        $ sudo apt-get update
        $ sudo apt-get install ruby2.0 ruby2.0-dev gcc libc6-dev nodejs

gcc and libc6-dev packages required to build some native ruby packages.

2. Get server software from github::

        $ git clone git@github.com:povilasb/showterm.io.git
        $ cd showterm.io
        $ git checkout debian

3. Install ruby Gems (this must be run in directory where you cloned Showterm)::

        $ gem2.0 install bundle
        $ bundle install

4. Setup PostgreSQL database *showterm*::

        $ rake2.0 db:setup

5. Run the Showterm application::

        $ rails serve

Now our application runs on port **3000** by default. Let's configure nginx
to redirect some domain to *http://localhost:3000*.


Setup Nginx
===========


1. Get Nginx package::

        $ sudo apt-get install nginx

2. Add virtual host for showterm. Create a new virtual host file. E.g.
   */etc/nginx/sites-available/show.example.com*::

        server
        {
                listen 80;
                server_name show.example.com;

                index index.html;
                location / {
                        proxy_pass http://localhost:3000;
                }
        }

Now enable this virtual host::

        $ ln -s /etc/nginx/sites-available/show.example.com /etc/nginx/sites-enabled/show.example.com

Reload Nginx configs::

        $ nginx -s reload

That's it. Now you should be able to open Showterm server application in
your Web browser with *http://show.example.com*.


Configure Showterm client
=========================

By default Showterm client sends it's recorded stream to http://showterm.io.
In order to make it send to your own server, simply append a line to
*~/.bashrc*::

        $ export SHOWTERM_SERVER=http://show.example.com

For this to work you have to restart the terminal.
