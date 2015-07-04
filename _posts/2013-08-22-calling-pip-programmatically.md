---
layout: post
title: "Calling pip programmatically"
date: 2013-08-22 17:45
disqus: y
categories: python
---
I was trying to find an API for [pip](http://www.pip-installer.org/) but I wasn't able to find a one. So started to look at the code and I realised that I could just emulate what the pip command was doing. It turned out to be really easy.

{% highlight python %}

import pip

pip.main(['install', 'django'])

{% endhighlight %}

This is equivalent to

{% highlight bash %}
pip install django
{% endhighlight %}

That was easy!