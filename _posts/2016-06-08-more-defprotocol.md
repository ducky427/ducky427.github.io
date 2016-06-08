---
layout: post
title: "More about protocols in ClojureScript"
date: 2016-06-08 17:29
disqus: y
categories: clojurescript
---

> TL;DR - Using a protocol via `apply` in performance sensitive code is not a good idea.

Building on the last blog post on [`deftype` and `defrecord`](/clojurescript/2016/05/12/deftype-defmethod-cljs/), I decided to investigate more about `defprotocol`. Whereas `deftype` and `defrecord` can be thought of as "bags of data", `defprotocol` defines beaviour for those.

A great post on the problem `deftype`, `defrecord` and `defprotocol` solves is [Solving the Expression Problem with Clojure 1.2
](https://www.ibm.com/developerworks/library/j-clojure-protocols/) by [Stuart Sierra](https://twitter.com/stuartsierra).

To get some deeper understanding of implementation of a protocol in ClojureSript, [Raphael Boukara](https://twitter.com/raphaelboukara) has a nice [post](http://blog.klipse.tech/clojurescript/2016/04/09/clojurescript-protocols-secret.html). This has some of the same details as my last blog post.

In the [comments section](http://blog.klipse.tech/clojurescript/2016/04/09/clojurescript-protocols-secret.html#comment-2653882504), [toxi](https://twitter.com/toxi) comment's that

> It's maybe worth pointing out that if you define a protocol with multiple arities that CLJS will compile to a drastically more involved & slower code, causing 1 extra loop to copy arguments (never understood why) and 2 extra dispatch functions per each invocation (one for varargs and aritity selection, the other as you described). So if performance matters, define protocols in such a way that each protocol fn only uses a single arity...

hmmm.... I decided to investigate this claim.

## Down the rabbit hole

Code below creates a simple protocol and deftype. We'll first examine the javascript generated and then benchmark it.

{% highlight clojure %}
(defprotocol P
  (foo [this a])
  (bar-me [this a] [this a b]))

(deftype Foo []
  P
  (foo [this a] 1)
  (bar-me [this a] 1)
  (bar-me [this a b] 1))
{% endhighlight %}

It generates the following javascript (feel free to skip this section):

{% highlight javascript %}
/**
 * @interface
 */
bench.core.P = function() {};

bench.core.foo = (function bench$core$foo(this$, a) {
    if ((!((this$ == null))) && (!((this$.bench$core$P$foo$arity$2 == null)))) {
        return this$.bench$core$P$foo$arity$2(this$, a);
    } else {
        var x__19136__auto__ = (((this$ == null)) ? null : this$);
        var m__19137__auto__ = (bench.core.foo[goog.typeOf(x__19136__auto__)]);
        if (!((m__19137__auto__ == null))) {
            return (m__19137__auto__.cljs$core$IFn$_invoke$arity$2 ? m__19137__auto__.cljs$core$IFn$_invoke$arity$2(this$, a) : m__19137__auto__.call(null, this$, a));
        } else {
            var m__19137__auto____$1 = (bench.core.foo["_"]);
            if (!((m__19137__auto____$1 == null))) {
                return (m__19137__auto____$1.cljs$core$IFn$_invoke$arity$2 ? m__19137__auto____$1.cljs$core$IFn$_invoke$arity$2(this$, a) : m__19137__auto____$1.call(null, this$, a));
            } else {
                throw cljs.core.missing_protocol("P.foo", this$);
            }
        }
    }
});

bench.core.bar_me = (function bench$core$bar_me(var_args) {
    var args28030 = [];
    var len__19548__auto___28033 = arguments.length;
    var i__19549__auto___28034 = (0);
    while (true) {
        if ((i__19549__auto___28034 < len__19548__auto___28033)) {
            args28030.push((arguments[i__19549__auto___28034]));

            var G__28035 = (i__19549__auto___28034 + (1));
            i__19549__auto___28034 = G__28035;
            continue;
        } else {}
        break;
    }

    var G__28032 = args28030.length;
    switch (G__28032) {
        case 2:
            return bench.core.bar_me.cljs$core$IFn$_invoke$arity$2((arguments[(0)]), (arguments[(1)]));

            break;
        case 3:
            return bench.core.bar_me.cljs$core$IFn$_invoke$arity$3((arguments[(0)]), (arguments[(1)]), (arguments[(2)]));

            break;
        default:
            throw (new Error([cljs.core.str("Invalid arity: "), cljs.core.str(args28030.length)].join('')));

    }
});

bench.core.bar_me.cljs$core$IFn$_invoke$arity$2 = (function(this$, a) {
    if ((!((this$ == null))) && (!((this$.bench$core$P$bar_me$arity$2 == null)))) {
        return this$.bench$core$P$bar_me$arity$2(this$, a);
    } else {
        var x__19136__auto__ = (((this$ == null)) ? null : this$);
        var m__19137__auto__ = (bench.core.bar_me[goog.typeOf(x__19136__auto__)]);
        if (!((m__19137__auto__ == null))) {
            return (m__19137__auto__.cljs$core$IFn$_invoke$arity$2 ? m__19137__auto__.cljs$core$IFn$_invoke$arity$2(this$, a) : m__19137__auto__.call(null, this$, a));
        } else {
            var m__19137__auto____$1 = (bench.core.bar_me["_"]);
            if (!((m__19137__auto____$1 == null))) {
                return (m__19137__auto____$1.cljs$core$IFn$_invoke$arity$2 ? m__19137__auto____$1.cljs$core$IFn$_invoke$arity$2(this$, a) : m__19137__auto____$1.call(null, this$, a));
            } else {
                throw cljs.core.missing_protocol("P.bar-me", this$);
            }
        }
    }
});

bench.core.bar_me.cljs$core$IFn$_invoke$arity$3 = (function(this$, a, b) {
    if ((!((this$ == null))) && (!((this$.bench$core$P$bar_me$arity$3 == null)))) {
        return this$.bench$core$P$bar_me$arity$3(this$, a, b);
    } else {
        var x__19136__auto__ = (((this$ == null)) ? null : this$);
        var m__19137__auto__ = (bench.core.bar_me[goog.typeOf(x__19136__auto__)]);
        if (!((m__19137__auto__ == null))) {
            return (m__19137__auto__.cljs$core$IFn$_invoke$arity$3 ? m__19137__auto__.cljs$core$IFn$_invoke$arity$3(this$, a, b) : m__19137__auto__.call(null, this$, a, b));
        } else {
            var m__19137__auto____$1 = (bench.core.bar_me["_"]);
            if (!((m__19137__auto____$1 == null))) {
                return (m__19137__auto____$1.cljs$core$IFn$_invoke$arity$3 ? m__19137__auto____$1.cljs$core$IFn$_invoke$arity$3(this$, a, b) : m__19137__auto____$1.call(null, this$, a, b));
            } else {
                throw cljs.core.missing_protocol("P.bar-me", this$);
            }
        }
    }
});

bench.core.bar_me.cljs$lang$maxFixedArity = 3;



/**
 * @constructor
 * @implements {bench.core.P}
 */
bench.core.Foo = (function() {})
bench.core.Foo.prototype.bench$core$P$ = true;

bench.core.Foo.prototype.bench$core$P$foo$arity$2 = (function(this$, a) {
    var self__ = this;
    var this$__$1 = this;
    return (1);
});

bench.core.Foo.prototype.bench$core$P$bar_me$arity$2 = (function(this$, a) {
    var self__ = this;
    var this$__$1 = this;
    return (1);
});

bench.core.Foo.prototype.bench$core$P$bar_me$arity$3 = (function(this$, a, b) {
    var self__ = this;
    var this$__$1 = this;
    return (1);
});

{% endhighlight %}

Its quite a bit of generated code, but the important javascript functions relating to the protocol `P` are:

1. `bar_me` - which examines the number of arguments given to the function given and then decides to call the appropriate arity function.
1. `bar_me.cljs$core$IFn$_invoke$arity$2` - `bar-me` 2 arity js generated function (1 arity for the detype and another for parameter `a`).
1. `bar_me.cljs$core$IFn$_invoke$arity$3` - `bar-me` 3 arity js generated function.

Also `(bar-me (Foo.) 1)` generates `bar_me.cljs$core$IFn$_invoke$arity$2((Foo.),(1));`.

 Whoaa! ClojureScript compiler already knows the number of appropriate arguments to the `bar-me` function. So it emits a direct call to the function 2 from above without going through function 1. This is counter to the claim made by toxi.

The only circumstance where the claim is valid is when the compiler does not know the number of arguments passed i.e. the number of arguments is dynamic. This happens if `bar-me` is used via `apply`. So the combination of apply and protocols is going to be slow. How slow, lets find out below.

## Benchmarks

Before I benchmark the protocols, I decided to set the base case to be - by explictly attaching a function to `Foo`'s prototype.

{% highlight clojure %}
(set! (.. Foo -prototype -baz)
      (fn [a]
        (this-as this a)))
{% endhighlight %}

This adds a method called `baz` to `Foo` data type and it has similar behaviour to the `foo` function in the `P` protocol.

I tested the performance of the following:

{% highlight clojure %}

(def f1 #(.baz foo-type 1)) ;; Manual
(def f2 #(foo foo-type 1)) ;; Simple protocol
(def f3 #(bar-me foo-type 1)) ;; bar-me 1 arity
(def f4 #(bar-me foo-type 1 2)) ;; bar-me 2 arity
(def f5 #(apply foo foo-type [1])) ;; Simple protocol via apply
(def f6 #(apply bar-me foo-type [1])) ;; bar-me 1 arity via apply
(def f7 #(apply bar-me foo-type [1 2])) ;; bar-me 2 arity via apply

{% endhighlight %}

All benchmarks were run on Mac OS X in Safari Technical Preview [Version 9.1.1 (11601.6.17, 11602.1.33)] browser.

| Type                        | Mean Timing           | Performance |
|---------------------------- | --------------------- | ----------- |
| Manual                      | 9.17459080317946E-10  | x           |
| Simple protocol - `foo`     | 5.80548203721976E-09  | 6.3x        |
| `bar-me` 1 arity            | 5.92361523817485E-09  | 6.4x        |
| `bar-me` 2 arity            | 5.80063496123134E-09  | 6.3x        |
| Simple protocol via apply   | 2.02819105288781E-07  | 221x        |
| `bar-me` 1 arity via apply  | 2.77527422532859E-07  | 300x        |
| `bar-me` 2 arity via apply  | 4.10717624610265E-07  | 448x        |


As suspected calling multi-arity protocols is no different from calling simple protocols if done directly. But boy, is `apply` expensive for protocols.

Also the timing difference between calling `baz` and `foo` gives us cost of using protocols. Its the cost associated with checking if the datatype is not null and checking if the appropriate method exists on the datatype.

Till next time!
