.. title: MNIST with scikit-learn
.. slug: mnist-with-scikit-learn
.. date: 2017-02-20 21:19:15 UTC+02:00
.. tags: machine learning, mnist, scikit-learn, python
.. category:
.. link:
.. description:
.. type: text

Solving `MNIST <http://yann.lecun.com/exdb/mnist/>`_ with `scikit-learn
<http://scikit-learn.org/>`_ is easy.

.. code-block:: python

    from sklearn import datasets
    from sklearn.neighbors import KNeighborsClassifier
    from sklearn.model_selection import train_test_split
    from sklearn.metrics import accuracy_score

    mnist = datasets.fetch_mldata('MNIST original', data_home='data')
    train_data, test_data, train_labels, test_labels = train_test_split(
        mnist.data, mnist.target)

    clf = KNeighborsClassifier()
    clf.fit(train_data, train_labels)

    predicted_labels = clf.predict(test_data)
    print('Prediction accuracy:', accuracy_score(test_labels, predicted_labels))

I get 97% accuracy which is great for such a simple implementation.

With `train_test_split()` I split the data into training and testing segments.
For simplicity I used K-nearest neighbour classifier but potentially others
could be used too (http://brianfarris.me/static/digit_recognizer.html).

Reusability
===========

`clf.fit()` takes some time to train the classifier.
But we can store the trained classifier on disk and reuse it multiple times:

.. code-block:: python

    from sklearn.externals import joblib

    clf = KNeighborsClassifier()
    clf.fit(train_data, train_labels)
    joblib.dump(clf, 'classifier_model.pkl')

    # ...

    clf = joblib.load('classifier_model.pkl')
    predicted_labels = clf.predict(test_data)

The serialized classifier takes up to 520 MB on disk.
