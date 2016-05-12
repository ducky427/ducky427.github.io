---
layout: post
title: "Dynamic children in a Reagent component"
date: 2016-05-11 15:45
disqus: y
categories: clojurescript reagent
---
[@mccraigmccraig](https://twitter.com/mccraigmccraig) asked on [clojurian's slack](http://clojurians.net/) why the following [Reagent](https://github.com/reagent-project/reagent) component causes a React warning:

{% highlight clojure %}
(defn an-integer
  [n]
  ^{:key n}
  [:li n])

(defn calling-component []
  [:div "Parent component"
   (for [n (range 0 10)]
     [an-integer n])])
{% endhighlight %}

whereas a similar looking component doesn't:

{% highlight clojure %}
(defn an-integer
  [n]
  [:li n])

(defn calling-component []
  [:div "Parent component"
   (for [n (range 0 10)]
     ^{:key n}
     [an-integer n])])

{% endhighlight %}

Initially my reaction was that maybe its Reagent which is causing this issue but a closer look at the documentation for this [topic](https://facebook.github.io/react/docs/multiple-components.html#dynamic-children) clarifies this exact scenario.

> The key should always be supplied directly to the components in the array, not to the container HTML child of each component in the array:

So in the Clojurescript context, the second code above is the correct way to do it.

And I think this makes sense. In the first example, `an-integer` is returning the `key` in what becomes React's render method whereas you want to get the `key` while you are creating the virtual dom without having to do call `render` as in the second example.

Aside, the following two Reagent components are exactly the same thing:

{% highlight clojure %}
(defn calling-component-1 []
  [:div "Parent component"
   (for [n (range 0 10)]
     ^{:key n}
     [:li n])])

{% endhighlight %}

{% highlight clojure %}
(defn calling-component-2 []
  [:div "Parent component"
   (for [n (range 0 10)]
     [:li {:key n} n])])

{% endhighlight %}

Reagent when creating the React component checks if the metadata has a key called `key` and if so it adds it to the components `props`. More [here](https://github.com/reagent-project/reagent/blob/master/src/reagent/impl/template.cljs#L269).

[Mike Thompson](https://twitter.com/wazound) has a great answer on [Stack Overflow](https://stackoverflow.com/questions/37164091/how-do-i-loop-through-a-subscribed-collection-in-re-frame-and-display-the-data-a/37186230#37186230) about the need for a `key` and different strategies to create children with/without keys.
