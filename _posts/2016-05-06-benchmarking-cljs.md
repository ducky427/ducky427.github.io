---
layout: post
title: "Micro-benchmarking Clojurescipt code in a browser"
date: 2016-05-06 12:45
disqus: y
categories: clojurescript
---
I wanted to find out which of creating a partial function is faster in Clojurescript. For example for adding two numbers, which of the two is faster:

{% highlight clojure %}
(partial + 1)
{% endhighlight %}

or

{% highlight clojure %}
#(+ 1 %)
{% endhighlight %}

I couldn't find a library which does this kind of benchmarking in a organised way. Luckily I found [benchmark.js](https://benchmarkjs.com/) which is used in [jsPerf.com](jsPerf.com). Helpfully it even ran in the browser which was the environment I was looking to run my code in.

I am a somewhat infrequent contributor to [cljsjs/packages](http://cljsjs.github.io/), a community initiative to package up javascript libraries which can be consumed in Clojurescript easily and survive the advanced optimisation grinder Google Closure compiler.

So I packaged up, [benchmark.js](https://clojars.org/cljsjs/benchmark), [lodash](https://clojars.org/cljsjs/lodash) and [platform](https://clojars.org/cljsjs/platform) and these libraries are now available on [clojars](https://clojars.org).

I then created a very simple front-end using [Reagent](http://reagent-project.github.io/).

My experiment is [here](http://blog.ducky.io/cljs-benchmark) and all the associated code is [here](https://github.com/ducky427/cljs-benchmark). Anyone wanting to run their own experiment can just fork my work and play away.

The main bit of code is which is using the API of benchmark.js:

{% highlight clojure %}

(def Suite (.-Suite js/Benchmark))

(def f1 (partial + 1))

(def f2 #(+ 1 %))

(defonce app-state (reagent/atom {:fastest nil
                                  :running? false
                                  :results []}))

(defn run-bench
  []
  (let [suite (Suite.)]
    (swap! app-state (fn [xs]
                       (merge xs {:results []
                                  :running? true
                                  :fastest nil})))
    (.. suite
        (add "partial function (partial + 1)"
             (fn []
               (f1 1)))
        (add "anonymous function #(+ 1 %)"
             (fn []
               (f2 1)))
        (on "cycle" (fn [event]
                      (let [b    (-> event .-target)
                            stat (-> b .-stats)]
                        (swap! app-state update :results  conj [(.-name b) stat]))))
        (on "complete" (fn []
                         (this-as this
                           (let [fastest  (-> this
                                              (.filter "fastest")
                                              (.map "name")
                                              (aget 0))]
                             (swap! app-state merge {:fastest fastest
                                                     :running? false})))))
        (run #js {"async" true}))))

{% endhighlight %}

P.S. Oh and the benchmark timings for my experiment are (on my Macbook Pro):

| Browser                     |`(partial + 1)`   | `#(+ 1 %)`         |
|---------------------------- | ---------------- | ------------------ |
| Chrome (52.0.2726.0 canary) | 2.28072563822e-8 | 1.090017047466e-8  |
| Safari (9.1.1 (11601.6.14)) | 8.75162575482e-9 | 5.509248548052e-9  |
| Firefox (46.0.1)            | 7.10047287610e-9 | 9.138133943344e-10  |

In all browsers `#(+ 1 %)` is faster. But enough of micro-benchmarking.
