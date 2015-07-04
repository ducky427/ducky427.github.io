---
layout: post
title: "Python and Neo4j performance"
date: 2012-04-13 10:48
disqus: y
categories: python neo4j performance
---

I have been playing aroung with [Neo4j v1.6](http://www.neo4j.org/)  in the embedded mode with Python. The [Neo4j library](http://pypi.python.org/pypi/neo4j-embedded/) for python uses JPype.
I was quite unhappy with the the raw speed. So I decided to decided to test the following code in Python, or more specifically CPython.

{% highlight python %}
import csv, sys, datetime
from neo4j import GraphDatabase

if __name__ == "__main__":
    start = datetime.datetime.now()
    db = GraphDatabase("test")
    reader = csv.reader(open(sys.argv[1]))
    header = reader.next()

    with db.transaction:
        for row in reader:
            # do something with row
            node = db.node(**dict(zip(header, row)))

    print "Closing....."
    db.shutdown()

    end = datetime.datetime.now()
    print "Time taken: %s" % (end - start, )
{% endhighlight %}
The code is pretty basic. It takes a csv file with headers and creates nodes out of that data. On a sample file, this code took ~415 seconds to run.

I could have used the batch insertion API provided by Michael Hunger [here](https://github.com/jexp/batch-import). But it's not appropriate if you are manipulating the data from the csv file.

Then I rewrote the code to make use of the native Neo4j library through [Jython](http://jython.org/). Anecdotally I've heard that Jython is slower than CPython but I wanted to test it for this particular program. I had allocated the same Java max heap size for both the programs.

{% highlight python %}
import sys, datetime, csv
from org.neo4j.kernel import EmbeddedGraphDatabase

if __name__ == "__main__":
    start = datetime.datetime.now()
    db = EmbeddedGraphDatabase("test-jy")

    reader = csv.reader(open(sys.argv[1]))
    header = reader.next()

    tx = db.beginTx()

    try:
        for row in reader:
            # do something with row.
            node = db.createNode()
            for k, v in zip(header, row):
                node.setProperty(k, v)
        tx.success()
    except:
        tx.failure()

    tx.finish()

    print "Closing....."
    db.shutdown()

    end = datetime.datetime.now()
    print "Time taken: %s" % (end - start, )
{% endhighlight %}

On the same sample file as before, this took ~130 seconds to run. I got a over 3x performance improvement by shifting to jython. That's a major speed improvement!

So I decided to use jython for the component which does the bulk data import and cpython in other places.

Your mileage may vary.
