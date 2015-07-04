---
layout: post
title: "Extending Cypher or How I Learnt To Stop Worrying and Love Cypher"
date: 2014-10-03 18:30
disqus: y
categories: neo4j clojure
---
A while back I had asked if I there was an easy way to extend Cypher. Cypher is an excellent declarative way for querying a Neo4j database but the functions in it leave me wanting for me. I had this twitter conversation:

<blockquote class="twitter-tweet" lang="en"><p><a href="https://twitter.com/ducky427">@ducky427</a> <a href="https://twitter.com/neo4j">@neo4j</a> It&#39;s open source... you can do anything... here is XOR (before it was added for reals) <a href="https://t.co/eWg126b0rS">https://t.co/eWg126b0rS</a></p>&mdash; Max De Marzi (@maxdemarzi) <a href="https://twitter.com/maxdemarzi/status/451383432061669377">April 2, 2014</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

<blockquote class="twitter-tweet" lang="en"><p><a href="https://twitter.com/maxdemarzi">@maxdemarzi</a> <a href="https://twitter.com/neo4j">@neo4j</a> Thanks! Would I be able to package something like this up and throw in the plugins/ dir and have cypher recognise it?</p>&mdash; ducky (@ducky427) <a href="https://twitter.com/ducky427/status/451384054433480704">April 2, 2014</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

<blockquote class="twitter-tweet" lang="en"><p><a href="https://twitter.com/maxdemarzi">@maxdemarzi</a> <a href="https://twitter.com/neo4j">@neo4j</a> cheers! 👍</p>&mdash; ducky (@ducky427) <a href="https://twitter.com/ducky427/status/451384893936005120">April 2, 2014</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

![Challenge Accepted](/images/challenge_accepted.png)

Cypher is built using Scala. As I had only basic knowledge of the language, creating new functionality would have been challenging for me. So I reached for my current favourite language, Clojure. It runs on JVM. So it should have been possible to integrate that with Scala and cypher. Once I was in clojure, I realised that I can really leverage the power of dynamic programming to attach a rocket booster to cypher. By that I mean, I could very easily add really complex functions to cypher easily. And the whole contraption would be stuck together with programming equivalent of duct tape.

So here we go.

A great use case for the extension is adding Date/DateTime support which Neo4j lacks natively. I don't think its a deal breaker but some people have strong views aroung this, as voiced at the end of Jim Weber's excellent talk at Skills Matter called [Impossible is Nothing](https://skillsmatter.com/skillscasts/5719-impossible-is-nothing). A workaround is to store that data as [Unix Epochs](https://en.wikipedia.org/wiki/Unix_time), which are just long values. But it means if you want to add a date to a node, you first have to convert it to a long externally and then run your cypher query. So there 2 steps when there should only be 1.


This is my Clojure code which added two date functions to cypher: `str-to-date`, `date-to-str`.

{% highlight clojure %}
(ns cypher-ext.core
  (:require [clj-time.format :as tf]
            [clj-time.coerce :as tc]))


(defmulti func (fn [n args] n))

(defmethod func :str-to-date
  [_ [x fmt]]
  (let [fmt  (or fmt "yyyy-MM-dd")
        ptn  (tf/formatter fmt)
        d    (tf/parse ptn x)]
    (tc/to-long d)))


(defmethod func :date-to-str
  [_ [x fmt]]
  (let [d    (tc/from-long x)
        fmt  (or fmt "yyyy-MM-dd")
        ptn  (tf/formatter fmt)]
    (tf/unparse ptn d)))

(defn call
  [args]
  (func (keyword (first args))
        (rest args)))
{% endhighlight %}

The code for `cypher-ext` is also on [github](https://github.com/ducky427/cypher-ext).


 With my changes, you can do the following:

{% highlight text %}
CREATE (n { date : specialfunc(["str-to-date", "2013-01-01"]) });
+-------------------+
| No data returned. |
+-------------------+
Nodes created: 1
Properties set: 1
{% endhighlight %}

{% highlight text %}
MATCH (n) RETURN n;
+-----------------------------+
| n                           |
+-----------------------------+
| Node[0]{date:1356998400000} |
+-----------------------------+
1 row
{% endhighlight %}

{% highlight text %}
MATCH (n) RETURN specialfunc(["date-to-str", n.date]);
+--------------+
| date         |
+--------------+
| "2013-01-01" |
+--------------+
1 rows
{% endhighlight %}

So the date is parsed to a long by `str-to-date` function and a long is converted to date string by `date-to-str` function.


Obviously now to add more functions to cypher is really easy. All I need to do is add another `defmethod` in my Clojure code.

The changes I had to make to Cypher in Neo4j codebase are [here](https://github.com/ducky427/neo4j/commit/359294e8711550cf47cd553e09cb638a637cca3c). The meat of the change is in the `compute` method of `SpecialFunc` class:

{% highlight scala %}
package org.neo4j.cypher.internal.compiler.v2_1.commands.expressions

import org.neo4j.cypher.internal.compiler.v2_1._
import pipes.QueryState
import symbols._
import org.neo4j.cypher.internal.helpers.CollectionSupport

import scala.collection.JavaConverters._

import clojure.java.api.Clojure

case class SpecialFunc(inner: Expression)
  extends NullInNullOutExpression(inner)
  with CollectionSupport {

  def compute(value: Any, m: ExecutionContext)(implicit state: QueryState) = {
    val require = Clojure.`var`("clojure.core", "require");
    require.invoke(Clojure.read("cypher-ext.core"))
    val fn = Clojure.`var`("cypher-ext.core", "call")
    fn.invoke(makeTraversable(value).toSeq.asJava)
  }

  def rewrite(f: (Expression) => Expression) = f(SpecialFunc(inner.rewrite(f)))

  def arguments = Seq(inner)

  def calculateType(symbols: SymbolTable): CypherType = CTAny

  def symbolTableDependencies = inner.symbolTableDependencies
}
{% endhighlight %}


**Everything is Awesome!!**


<iframe width="560" height="315" src="//www.youtube.com/embed/StTqXEQ2l-Y" frameborder="0" allowfullscreen></iframe>