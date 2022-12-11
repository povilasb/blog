.. title: C++ unit tests with googletest
.. slug: cpp-googletest-intro
.. date: 2014-03-02 20:43 UTC+02:00
.. tags: cpp,testing
.. category:
.. link:
.. description:
.. type: text

==========================
Unit tests with googletest
==========================

Unit tests let programmer know that his software works as expected. If you i
have ever done heavy development or you are planning to do so, you would
definitely see the obvious advantages of unit testing your software:

* you can easily check if software works on different platforms;
* unit tests might replace or at least supplement the samples;
* you can assert if your code still works after refactoring;
* unit tests can also be used to test and track performance;
* tests can be run automatically and periodically;
* unit tests make you try module/library API before integrating it to the other
  code - this way design issues can be noticed sooner;
* unit tests inspire confidence;
* and many more advantages...

One of the many testing frameworks for C++ if googletest. Let's see how to
write unit tests for your modules.


Setup environment
=================

We'll use googletest, CMake and debian 7.3 system. Get the prerequisites:

.. code-block:: bash

        $ sudo apt-get install cmake unzip

Download the latest googletest version from
`here <https://code.google.com/p/googletest/downloads/list>`_ (by the time I
wrote this article it was v1.7.0).

Unpack the package:

.. code-block:: bash

        $ unzip gtest-1.7.0.zip


Create sample project
=====================

You can get a sample from github:

.. code-block:: bash

        $ git clone https://github.com/povilasb-com/cpp-googletest-intro.git


Sample project directory tree::

        .
        |__ googletest-cmake
          |__ lib
          | |__ googletest -> ~/dev-tools/gtest-1.7.0
          |__ src
          | |__ lib1.cpp
          | |__ lib1.hpp
          |__ test
          | |__ lib1_test.cpp
          |__ CMakeLists.txt
          |__ Makefile


Create directory hierarchy
--------------------------

.. code-block:: bash
        :linenos:

        $ mkdir googletest-cmake
        $ cd googletest-cmake
        $ mkdir lib src test


Create a symbolic link to google testing framework directory
------------------------------------------------------------

I have google test located in my home dir dev-tools/gtest-1.7.0, so:

.. code-block:: bash

        $ ln -s ~/dev-tools/gtest-1.7.0 lib/googletest


Write sample functions that will be tested
------------------------------------------

cpp-googletest-cmake/src/lib1.hpp:

.. code-block:: c++
        :linenos:

        #ifndef _LIB1_H
        #define _LIB1_H 1

        int sum(int n1, int n2);

        int mul(int n1, int n2);

        #endif

cpp-googletest-cmake/src/lib1.cpp:

.. code-block:: c++
        :linenos:

        #include "lib1.hpp"

        int
        sum(int n1, int n2)
        {
                return n1 + n2;
        }

        int
        mul(int n1, int n2)
        {
                return n1 * n2;
        }


Create build scripts
--------------------

googletest-cmake/src/CMakeLists.txt:

.. code-block:: cmake
        :linenos:

        cmake_minimum_required (VERSION 2.6)
        project (Lib1 CXX)

        set (CMAKE_CXX_FLAGS "-ggdb")

        set (SRC_DIR "${CMAKE_CURRENT_SOURCE_DIR}/src")

        include_directories ("${SRC_DIR}")

        file (GLOB_RECURSE SRC_FILES "${SRC_DIR}/*.cpp")

        # Compiles static lib that will be linked with tests.
        set (LIB_NAME "lib1")
        add_library ("${LIB_NAME}" STATIC ${SRC_FILES})

        # Include googletest.
        set (GTEST_DIR "${CMAKE_CURRENT_SOURCE_DIR}/lib/googletest")
        add_subdirectory (${GTEST_DIR})
        include_directories ("${GTEST_DIR}/include")

        # Build tests executable.
        set (TEST_EXEC "${LIB_NAME}_test")
        set (TEST_SRC_DIR "${CMAKE_CURRENT_SOURCE_DIR}/test")
        file (GLOB_RECURSE TEST_SRC_FILES "${TEST_SRC_DIR}/*.cpp")

        add_executable ("${TEST_EXEC}" ${TEST_SRC_FILES})
        target_link_libraries ("${TEST_EXEC}" "${LIB_NAME}" "gtest" "gtest_main")

CMake does the main build process, but we also use a helper Makefile that
runs cmake and the compiled tests executable.

googletes-cmake/src/Makefile:

.. code-block:: make
        :linenos:

        BUILD_DIR = build

        all: test test-run

        test: $(BUILD_DIR)
                cd $(BUILD_DIR); cmake $(CURDIR); make

        test-run:
                $(BUILD_DIR)/lib1_test

        $(BUILD_DIR):
                mkdir -p $@

        clean:
                rm -rf $(BUILD_DIR)

        .PHONY: all test test-run clean


Create test suite
-----------------

googletes-cmake/test/lib1_test.cpp:

.. code-block:: c++
        :linenos:

        #include <gtest/gtest.h>

        #include "lib1.hpp"


        TEST(lib1, sum)
        {
                int s = sum(10, 15);
                ASSERT_TRUE(s == 25);
        }

        TEST(lib1, mul)
        {
                int m = mul(10, 15);
                ASSERT_TRUE(m == 150);
        }

This is the simplest form of the unit tests. It uses ASSERT_TRUE macro which
checks if the specified expression is true. If it is not the test execution is
terminated.

There are more convenience macros that you can use to assert expected values:
https://code.google.com/p/googletest/wiki/V1_7_Primer.


Run tests
=========

Simply type *make* in project directory. It invokes the helper Makefile that
runs CMake which builds the tests executable and then Makefile runs this
executable.


Successfull execution results
-----------------------------

.. image:: /images/gtest_ok.png


Example of tests failure
------------------------

.. image:: /images/gtest_fail.png
