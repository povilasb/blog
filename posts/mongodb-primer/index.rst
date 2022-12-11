.. title: MongoDB Primer
.. slug: mongodb-primer
.. date: 2017-05-19 16:56:59 UTC+03:00
.. tags: mongodb,databases
.. category:
.. link:
.. description:
.. type: text

When I have some to build sample sample toy PoC applications I tend to use
MongoDB.
I guess because it's schemaless, eats any JSON I give, has easy query language
and good language support.

Concepts
========

Just like in MySQL I have a database which groups document collections together.
Document collection is like a table in MySQL. It groups multiple logically
related documents, e.g. people, items, payments, etc.
Although, each document must not have the same fields.
A document is simply JSON object. E.g. person document::

    {
        "name": "Povilas",
        "age": 26,
        "gender": "male",
        "interests": ["systems", "books", "coding"]
    }

Basic Commands
==============

Show all databases::

    > show dbs

Create a database::

    > use my_db

This command won't create DB immediately. Once you insert the first document
new database together with a table will be created.

Insert a document::

    > db.table.insertOne({"name": "Povilas", "age", 26, "gender": "male"})

Delete document::

    > db.table.deleteOne({"name": "Povilas"})

Show all documents::

    > db.table.find()

Delete all documents::

    > db.table.deleteMany({})
