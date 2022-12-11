.. title: Notes on UTF-8
.. slug: notes-on-utf-8
.. date: 2017-07-19 16:22:43 UTC+03:00
.. tags: utf-8, strings
.. category:
.. link:
.. description:
.. type: text

UTF-8 is the most popular, to my mind, character encoding invented by
Ken Thomson (co-inventor of Unix and Go language) and Rob Pike (co-inventor
of Go language).
If you use Rust and Python, strings are encoded in UTF-8 by default.
Anyway, I got interested in `Boyer-Moore string search algorithm
<https://en.wikipedia.org/wiki/Boyer%E2%80%93Moore_string_search_algorithm>`_.
And of course I wanted to implement it for UTF-8 which is most practical.
Before I could do that effectively, I had to understand UTF-8 internals.

Description
===========

UTF-8 is pretty well described in `RFC 3629
<https://tools.ietf.org/html/rfc3629>`_.
UTF-8 character size varies between 1-4 bytes. The exact size depends
on a character, but personally I think that more common characters are encoded
with less bytes. E.g. ASCII symbols are encoded with 1 byte, cyrillic,
Lithuanian - 2 bytes, Japanese - 3, etc.

The maximum size of UTF-8 character code is 21 bits. Other 11 bits are
used as metadata. This table sums up the encoding format::

    Char. number range  |        UTF-8 octet sequence
       (hexadecimal)    |              (binary)
    --------------------+---------------------------------------------
    0000 0000-0000 007F | 0xxxxxxx
    0000 0080-0000 07FF | 110xxxxx 10xxxxxx
    0000 0800-0000 FFFF | 1110xxxx 10xxxxxx 10xxxxxx
    0001 0000-0010 FFFF | 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx

where `x` are bits used to encode character code.

Example
=======

Let's take character `č` and encode it to UTF-8. It's encoded with 2 bytes
`0xC48D` and the character code is `0x10D`.
We can test this with Rust:

.. code-block:: rust

    let chars = vec![0xC4, 0x8D];
    for c in String::from_utf8(chars).unwrap().chars() {
        println!("{} 0x{:X}", c, c as u32);
    }

The output is::

    č 0x10D

String operations
=================

UTF-8 strings can be easily concatenated.

Character at position `N` access complexity is `O(N)`.
That's why Rust strings don't have index operator `str[]` because
usually this operator means `O(1)` complexity.

String search is the same as in ASCII strings. Basically, you can do byte
to byte search. Thus Boyer-Moore search should also work with UTF-8 strings
without bigger problems.

Iterating UTF-8 strings is a bit more complicated than ASCII. You have
to calculate how many bytes each character takes up.
