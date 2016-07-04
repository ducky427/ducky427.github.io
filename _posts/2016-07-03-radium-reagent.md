---
layout: post
title: "Using Radium with Reagent"
date: 2016-07-04 10:15
disqus: y
categories: clojurescript reagent
---
I came across [this](https://stackoverflow.com/questions/32355688/why-radium-doesnt-work-with-reagent-clojurescript/38183679#38183679) StackOverflow question regarding [Radium]() with [Reagent](https://github.com/reagent-project/reagent).

> Radium is a set of tools to manage inline styles on React elements. It gives you powerful styling capabilities without CSS.
> Inspired by [React: CSS in JS](https://speakerdeck.com/vjeux/react-css-in-js) by [vjeux](https://twitter.com/Vjeux).

First thing I had to do with was to package up Radium for [cljsjs/packages](https://github.com/cljsjs/packages). By now I have enough experience in packaging up React components using [Webpack](https://webpack.github.io/) for UMD consumption. This now lives as [cljsjs.radium](https://github.com/cljsjs/packages/tree/master/radium).

Next, I had to figure out how to use Radium from ClojureScript. In the examples, Radium seemed to be heavily using [ES2016](https://medium.com/google-developers/exploring-es7-decorators-76ecb65fb841#.qkx97s9k8) decorators. Luckily they also had support for `React.createClass` based versions.

Below is a simple example of using Radium with Reagent

{% highlight clojure %}
(ns rera.core
  (:require [reagent.core :as reagent]
            [cljsjs.radium]))

(def Radium js/Radium)

(def styles {:base {:color "#fff"
                    ":hover" {:background "#0A8DFF"}}
             :primary {:background "#0074D9"}
             :warning {:background "#FF4136"}})

(defn button
  [data]
  (let [kind (keyword (:kind data))]
    [:button
     {:style (clj->js [(:base styles)
                       (kind styles)])}
     (:children data)]))

(def btn (Radium. (reagent/reactify-component button)))

(def rbtn (reagent/adapt-react-class btn))

(defn hello-world
  []
  [:div
   [rbtn {:kind :primary} "Hello Primary"]
   [rbtn {:kind :warning} "Hello Warning"]])

(reagent/render-component
 [hello-world]
 (. js/document (getElementById "app")))
{% endhighlight %}

This example shows how easy is to use the `hover` css property in a React component. To do with with plain React or Reagent, isn't trivial without resorting to a CSS file.

Till next time!
