.. title: Static C++ code analysis
.. slug: cpp-static-analysis
.. date: 2014-03-16 20:39:00 UTC+02:00
.. tags: cpp,qa
.. category:
.. link:
.. description:
.. type: text

====================
Static code analysis
====================

Static code analysis is the process of detecting errors and defects in
software's source code. Static analysis can be viewed as an automated code
review process.


Tools
=====

* scan-build from LLVM project [#f2]_.
* Cppcheck
* Flint [#f5]_
* Many more [#f3]_.


scan-build
----------

Let's see how to analyze c++ code with scan-build.
Again I will be using Debian 7 and cmake to build my c++ programs.


Get the clang package
+++++++++++++++++++++

LLVM provides scan-build in a debian package [#f1]_.

1. Add the apt key of LLVM repository

.. code-block:: bash

        $ wget -O - http://llvm.org/apt/llvm-snapshot.gpg.key|sudo apt-key add -

2. Add the LLVM repos to apt sources directory

.. code-block:: bash

        $ sudo echo "deb http://llvm.org/apt/wheezy/ llvm-toolchain-wheezy main"
          > /etc/apt/sources.list.d/llvm-clang.list
        $ sudo echo "deb-src http://llvm.org/apt/wheezy/ llvm-toolchain-wheezy main"
          >> /etc/apt/sources.list.d/llvm-clang.list

3. Get clang package

.. code-block:: bash

        $ sudo apt-get install clang-3.5

When this article was written latest clang version was clang-3.5. This package
contains scan-build script. Now we are ready to analyze our sources.


Code analysis
+++++++++++++

Code analysis is easy with scan-build:

.. code-block:: bash

        $ scan-build g++ main.cpp

To demonstrate better of what can static analysis do I've set up a git
`repo <https://github.com/povilasb-com/cpp-static-analysis>`_
with c++ project that has some sample bugs that scan-build might catch [#f4]_.

Let's download the sample:

.. code-block:: bash

        $ git clone https://github.com/povilasb-com/cpp-static-analysis

Now run the scan-build analyzer:

.. code-block:: bash

        $ cd cpp-static-analysis
        $ scan-build make

This should yield that some bugs were found::

        scan-build: 6 bugs found.
        scan-build: Run 'scan-view /tmp/scan-build-2014-03-16-200244-31706-1'
        to examine bug reports.

To see the report in HTML format enter:

.. code-block:: bash

        $ scan-view /tmp/scan-build-2014-03-16-200244-31706-1

You should see something like this:

.. image:: /images/scan_build_result.png


.. rubric:: References

.. [#f1] http://llvm.org/apt/
.. [#f2] http://clang-analyzer.llvm.org/scan-build.html
.. [#f3] http://en.wikipedia.org/wiki/List_of_tools_for_static_code_analysis
.. [#f4] https://github.com/povilasb-com/cpp-static-analysis
.. [#f5] https://github.com/facebook/flint
