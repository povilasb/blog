.. title: Boyer-Moore string search
.. slug: boyer-moore-string-search
.. date: 2017-08-10 10:05:28 UTC+03:00
.. tags: algorithms,string-search
.. category:
.. link:
.. description:
.. type: text

Boyer-Moore is a string search algorithm.
The first time I heard about it was on FreeBSD mailing list when someone
was explaining "why GNU grep is fast". [#f2]_
Although, it was invented in 1977 [#f3]_, turns out this algorithm is
considered one of the most efficient and very widely used even nowadays. [#f1]_

So I thought I have to at least take a brief overview of this classic algorithm.

Overview
========

Similarly to `naive string search
<https://en.wikipedia.org/wiki/String_searching_algorithm#Na.C3.AFve_string_search>`_.
the algorithm scans text from left to right searching for the given pattern.
Although, the pattern is matched from right to left::

    Text:    X X X A B C X X X D B C
    Pattern:       D B C - - -> pattern shift
                  <- - - pattern matching

What makes Boyer-Moore algorithm faster than naive string search is the
heuristics it uses to shift pattern right. Rather than shifting one symbol
at a time, this algorithm uses some heuristics to skip multiple symbols.
We can optionally choose which heuristics to use. In order to go fastest,
we should use them all.

Each heuristic calculates how many positions we can skip. Then we choose the
maximum number of skips and repeat the matching.

At first glance the algorithm looks pretty easy. But it has multiple
cases which might make it a little cofusing.

Bad character heuristic
=======================

It states that when text does not match the pattern at a given position, we
can shift the pattern right until it does. [#f4]_

::

       0 1 2 3 4 5 6 7
    T: X X X A B C A C
    P:     B C A C
           0 1 2 3

In this case the algorithm starts matching the pattern at text position 5
and pattern position 3. The mismatch occurs at `text_pos` 4 and `pattern_pos` 2.
The mismatched symbol is 'B'. 'B' exists in the `pattern_pos` 0.
Thus we can skip 2 symbols: `2 - 0 = 2`::

       0 1 2 3 4 5 6 7
    T: X X X A B C A C
    P:         B C A C
               0 1 2 3

In case the pattern does not have the mismatched symbol, we can move it
past that symbol::

       0 1 2 3 4 5 6 7 8 9
    T: X X X A B C A C D C
    P:     D C A C

Mismatched symbol is `B` which does not exist in the pattern.
Thus the pattern is shifted to::

       0 1 2 3 4 5 6 7 8 9
    T: X X X A B C A C D C
    P:           D C A C

Preprocessing
-------------

When mismatch happens we could search for a matching symbol traversing
the pattern backwards. That would take O(n) comparisons.
Instead we can achieve O(1) with a lookup table.
This table is constructed once for the given pattern and it holds the last
position for a given character.  E.g.:

.. code-block:: python

    pattern = 'abca'
    table['a'] = 3
    table['b'] = 1
    table['c'] = 2


.. code-block:: python

    def preproc_bad_character(pattern: str) -> list:
        last_char_pos = [-1] * 256
        for i, c in enumerate(pattern):
            last_char_pos[ord(c)] = i
        return last_char_pos

Good suffix heuristic
=====================

This heuristic is nicely explained in `this video
<https://www.youtube.com/watch?v=lkL6RkQvpMM>`_.

It heuristic has 3 cases:

1. the matching pattern suffix exists in another pattern place::

       0 1 2 3 4 5 6 7 8 9
    T: X X X A B C A C D C
    P: A B C D B C

   `BC` matches the text and another `BC` is in the pattern at position `1`.
   Thus we could shift the pattern by `3` positions right::

       0 1 2 3 4 5 6 7 8 9
    T: X X X A B C A C D C
    P:       A B C D B C

2. the matching pattern suffix is also the prefix of a pattern::

       0 1 2 3 4 5 6 7 8 9
    T: X X X A B C A C D C
    P: B C D A B C

   In this case we can safely shift the pattern by `4` positions::

       0 1 2 3 4 5 6 7 8 9
    T: X X X A B C A C D C
    P:         B C D A B C

3. When none of the above cases are satisfied the pattern is shifted past
   the matched text part::

       0 1 2 3 4 5 6 7 8 9
    T: X X X A B C A C D C
    P:     D A B C

       0 1 2 3 4 5 6 7 8 9
    T: X X X A B C A C D C
    P:             D A B C

This heuristic also has a preprocessing step to construct a lookup table.
The algorithm is a bit more complicated and better explained in [#f6]_ [#f5]_.


.. rubric:: References

.. [#f1] http://www-igm.univ-mlv.fr/~lecroq/string/node14.html
.. [#f2] https://lists.freebsd.org/pipermail/freebsd-current/2010-August/019310.html
.. [#f3] https://en.wikipedia.org/wiki/Boyer%E2%80%93Moore_string_search_algorithm
.. [#f4] http://www.geeksforgeeks.org/pattern-searching-set-7-boyer-moore-algorithm-bad-character-heuristic/
.. [#f5] http://www.inf.fh-flensburg.de/lang/algorithmen/pattern/bmen.htm
.. [#f6] http://www.geeksforgeeks.org/boyer-moore-algorithm-good-suffix-heuristic/
