---
layout: post
title: "Installing python-glpk on OS X 10.8"
date: 2013-07-01 11:00
disqus: y
categories: python
---
I have been using [PuLP](https://code.google.com/p/pulp-or/) for linear optimisation. By default, on OS X, the library used the packaged [COIN](http://www.coin-or.org/) command line program. This tends to be slow if you are many optimisations as I was doing. Luckily PuLP can handle other optimisers as well like [GLPK](http://www.gnu.org/software/glpk/glpk.html) or paid ones like [CPLEX](http://www.cplex.com/) or [GUROBI](http://www.gurobi.com/).

So I decided to install GLPK on my computer. This proved harded than I initially imagined.

Installing GLPK [Homebrew](http://brew.sh/) and swig:

{% highlight bash %}
brew install homebrew/science/glpk
brew install swig
{% endhighlight %}

Now to install python bindings for GLPK. I tried out two binding for python (more details [here](http://en.wikibooks.org/wiki/GLPK/Python)):

* Python-GLPK
* PyGLPK

I didn't have any success with PyGLPK. So I stuck with Python-GLPK.

My instructions:

Firstly install 'ply'

{% highlight bash %}
pip install ply
{% endhighlight %}

Then download the code and unzip it.

{% highlight bash %}
wget http://www.dcc.fc.up.pt/~jpp/code/python-glpk/python-glpk_0.4.43.orig.tar.gz
tar xf python-glpk_0.4.43.orig.tar.gz
cd python-glpk-0.4.43/src/
{% endhighlight %}

In swig/Makefile replace line number 1 to
{% highlight bash%}
PYVERS := 'python2.7'
{% endhighlight %}

In folder 'src', run
{% highlight bash%}
make
{% endhighlight %}

I got these errors and I ignored them.

{% highlight bash%}
make -C swig all
swig -python  glpkpi.i
./glpk.h:916: Warning 314: 'in' is a python keyword, renaming to '_in'
sed -i 's/:in /:_in /g' glpkpi.py
sed: 1: "glpkpi.py": extra characters at the end of g command
make[1]: *** [glpkpi.py] Error 1
make: *** [all] Error 2
{% endhighlight %}

Now run:

{% highlight bash%}
cp swig/glpkpi.py python/glpkpi.py
python setup.py install
{% endhighlight %}

This should install the library. To test if the install worked, go to the examples directory

{% highlight bash%}
cd ../examples
python test.py
{% endhighlight %}
