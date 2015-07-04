---
layout: post
title: "Neo4j unmanaged extension in Clojure"
date: 2014-01-08 18:30
disqus: y
categories: neo4j clojure
redirect_from: "/neo4j,/clojure/2014/01/08/neo4j-clojure/"
---
I needed to create an [Unmanaged Extension](http://docs.neo4j.org/chunked/milestone/server-unmanaged-extensions.html) in Neo4j but the prospect of using Java gave me shudders.

I have been programming in [Clojure](http://www.clojure.org) for a little while now. Given that Clojure is a JVM language, I decided to try to write the extension in that. Neo4j uses the [JAX-RS](https://en.wikipedia.org/wiki/Java_API_for_RESTful_Web_Services) API for supporting user supplied code. Luckily O'Reilly's excellent book on Clojure, [Clojure Programming](http://www.clojurebook.com/) had an example of a [JAX-RS class](https://github.com/clojurebook/ClojureProgramming/blob/master/ch09-annotations/src/main/clojure/com/clojurebook/annotations/jaxrs.clj). *I love it when a plan comes together*.

I modified that code to get what I needed. I hit across two issues:

* A nasty ``deftype`` namespace not loading [bug](http://dev.clojure.org/jira/browse/CLJ-1208) in Clojure. It seems to be an old one. But found a work around on [Stack Overflow](https://stackoverflow.com/questions/10953621/clojure-deftype-calling-function-in-the-same-namespace-throws-java-lang-illegal/).

* [Neo4j 2.0 documentation](http://api.neo4j.org/2.0.0/org/neo4j/graphdb/GraphDatabaseService.html) stated that a transaction is required only when the graph is being written to. Nope. I've highlighted it to them and as ever they've quickly fixed [it](https://github.com/neo4j/neo4j/pull/1801). I was trying to get a node by its ID and I was getting a ``NotInTransactionException``. Anyway its not needed for this exercise.

Here is my working code (I've tried to stay close to the Neo4j provided example; my only deviation being calling another function from the function which directly handles the ``GET``:

{% highlight clojure %}
(ns jaxtest.core
  (:import (java.nio.charset Charset)
           (javax.ws.rs Path PathParam Produces GET)
           (javax.ws.rs.core Context Response)))

(defn dummy [] "Hello World")

(definterface HelloWorld
  (hello [^org.neo4j.graphdb.GraphDatabaseService database ^long node-id]))

(deftype ^{Path "/helloworld"} HelloWorldResource []
  HelloWorld
  (^{GET true
     Produces ["text/plain"]
     Path "/{nodeId}"}
    hello
    [this ^{Context true} database ^{PathParam "nodeId"} node-id]
      (require 'jaxtest.core)
      (let  [result  (-> (str (dummy) ", nodeId=" node-id)
                         (.getBytes (Charset/forName "UTF-8")))]
        (-> (Response/ok)
            (.entity result)
            .build))))

{% endhighlight %}

So now I have Java, Scala (for Cypher) and Clojure (for my extension) all running in the same JVM.

JVM ftw! Woot!!
