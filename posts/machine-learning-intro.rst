.. title: Machine learning in 3 lines of code
.. slug: machine-learning-intro
.. date: 2016-08-06 21:34 UTC-05:00
.. tags: python,ml
.. category:
.. link:
.. description:
.. type: text


This is a very light introduction into machine learning.
I will demonstrate how to solve one specific problem using
`scikit-learn <http://scikit-learn.org/>`_, a machine learning framework in
just 3 lines of code.

Problem
=======

I created a syntetic problem where data is very simple so we don't have
to struggle with distilling it.

Based on very simple rules we have labeled points on a coordinate plane.
If x > 0 and y < 0 then the point is labeled as *X*.
If x < 0 and y > 0 then the point is labeled as 'O'.

::

        o        ^ y
            o    |
          o      |
                 |
     ------------+-------------> x
                 |    x
                 |       x
                 |  x

Our task is to label any given point.

**Input**: (x, y)

**Output**: X or O

E.g.

**Input**: (1, -7)

**Output**: *X*

Manual solution
===============

In this case to implement the classification manually using if statements is
really easy:

.. code-block:: python

    def label_for(x, y):
        if x < 0 and y > 0:
            return 'o'
        if x > 0 and y < 0:
            return 'x'

        return '?'

Anyway let's compare with the machine learning implementation.

Machine learning solution
=========================

.. code-block:: python

    from sklearn.neighbors import KNeighborsClassifier

    import coords

    clf = KNeighborsClassifier()
    clf.fit(*training_data(coords.make_n_random(100)))
    print(clf.predict([[1, -7], [-1, 7]]))

Basically the last 3 lines of code create the classifier, train it with
sample data and predict the labels for the given sample coordinates.
The full solution code is in
https://github.com/povilasb/machine-learning/blob/master/labeled-coordinates/x_o.py.
If we run it, we get the output::

	['x' 'o']

Meaning (1, -7) was labeled as 'x' and (-1, 7) as 'o'.
Which is correct.

Let's take a deeper look how this actually works.

Obtain data
-----------

To make our classifier understand which coordinates have which labels we
must train it with the sample data.

We will not get training data from anywhere. Instead we will generate it.
We'll use `generate_n_random()` function from
`coords <https://github.com/povilasb/machine-learning/blob/master/labeled-coordinates/coords.py>`_
module to generate N coordinates that comply with my given rules.
Basically, this function returns a list of labeled coordinates:

.. code-block:: python

    [(1, -5, 'x'), (-4, 3, 'y'), ...]

Scrub data
----------

`coords.make_n_random()` generates labeled coordinates in a different format
than the `scikit-learn` classifier expects.
So we need to reformat our training data.

That's where we use `training_data()` function:

.. code-block:: python

    def training_data(coordinates):
        """Converts coordinates to classifier acceptable format."""
        return (
            [[x, y] for x, y, _ in coordinates],
            [label for _, _, label in coordinates]
        )

It separates coordinates and labels into two separate arrays.

Classify new data
-----------------

Once we have preprocessed data we can continue with training the model
and classifying new coordinates:

.. code-block:: python

    clf = KNeighborsClassifier()
    clf.fit(*training_data(coords.make_n_random(100)))
    print(clf.predict([[1, -7], [-1, 7]]))

`KNeighborsClassifier` is a python class that implements the
`k-nearest neighbors algorithm <https://en.wikipedia.org/wiki/K-nearest_neighbors_algorithm>`_.

`clf.fit()` trains the classifier with the given labeled data.

`clf.predict()` returns predicted labels for the specified coordinates.

Machine learning vs manual solution
===================================

In this case it's obvious that solving the problem with if statements
is way easier: you don't need to gather any data, scrub it, etc.

But what if our input data changes as time goes by?

Let's say now every coordinate where x > 0 and y > 0 is labeled as 's'::

        o        ^ y
            o    |    s
          o      |       s
                 | s
     ------------+-------------> x
                 |    x
                 |       x
                 |  x

In a machine learning-based implementation we don't need to change anything.
We just have to retrain the model with new data.

If we implemented the classification manually, we would have to program a new
rule:

.. code-block:: python

    def label_for(x, y):
        if x < 0 and y > 0:
            return 'o'
        if x > 0 and y < 0:
            return 'x'
    +   if x > 0 and y > 0:
    +       return 's'

        return '?'

Conclusions
===========

When we use a framework it might be really easy to solve problems using
machine learning.

The example problem was easy to implement using if statements.
But any input data changes require to adopt the algorithm.
Also, in real life scenarios problems are not that simple.
For example if we wanted to recognize digits in an image we should program
10 different cases for different digits.
Also, if the digit font changes, we would have to adopt code, etc.
Using machine learning all we need to do is to train our model with new data.
And this is way more scalable.

So machine learning helps to solve a lot of otherwise unsolvable problems.
And we don't really need to understand the maths behind it because
there are great tools that do the job for us.

Environment setup
=================

I used the `scikit-learn` framework with python 3.
It depends on a lot of other packages.
So if you dont have `scikit-learn` installed on your machine,
I created a `Docker container
<https://hub.docker.com/r/povilasb/scikit-learn/>`_.

Now all you have to do to run your python script in this environment is::

    $ docker run -it --rm=true -v `pwd`:/tmp/ml povilasb/scikit-learn python3 /tmp/ml/x_o.py

This command will download Docker image, create a container, run the
specified script in it and finally destroy it.
