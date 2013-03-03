---
title: Tracking down memory leaks in Ruby 1.9
permalink: blog/find-references.html
layout: post
fuzzydate: Mar 2013
---

Last weekend, for the first time in a while, we didn't deploy one of our services for several days. Unfortunately this revealed a problem: over time the service would get slower and slower as it started using more and more memory.

A memory leak is one of my least favourite bugs to track down, it requires a detailed knowledge of the entire codebase, strong intuition and a fair amount of luck to spot. So I started along the route of the lazy programmer looking for tools thate could help me solve the problem.

ObjectSpace.each_object
=======================

In the first instance I turned to ObjectSpace: 

{%highlight ruby%}
counts = Hash.new{ 0 }
enum = ObjectSpace.each_object
enum.each_with_object(counts) do |o, h|
  h[o.class] += 1
end
{% endhighlight %}

By dumping the counts into a file after each request and using the `diff` tool, I was able to determine which objects were leaking. In this case, it seemed to be our Controller objects along with various handlers and helpers that are mutually linked to them. Unfortunately, because we have several objects that collaborate and form a cycle, it wasn't obvious either which was the one that was actually being retained, nor what was retaining it.

Searching the internet for more cool tools to use led me to these [memtrack patches](http://www.softwareverify.com/ruby-custom-build-memtrack.php), unfortunately not only are the patches for ruby 1.8, they aren't even downloadable anymore. So it was time to set out on my own.

ObjectSpace.find_references
===========================

Ruby already knows which objects retain each other, it needs to for the mark-and-sweep garbage collector to work. Unfortunately it doesn't expose this to users. So my first step was to add a new function `find_references` to `ObjectSpace` that would expose this information in Ruby land.

{%highlight ruby%}
ObjectSpace.find_references(
  ObjectSpace.each_object(MainController).first
)
{% endhighlight %}

ObjectGraph#view!
=================

This works, but it turned out to be far too low level for manual use while debugging. After you've loaded up a few arrays like this, you've added so many new references that point to the objects you're trying to track down that you have to throw the whole process away and start again.

What I needed instead was some way to easily get a visual representation of what was retaining what. Luckily this can be built in pure ruby once `ObjectSpace.find_references` exists. Calculating the graph from one object all the way up to the roots of the garbage collection graph was then just a case of running a breadth-first search.

<figure>
Once the graph has been obtained, it's then just a case of putting it through graphviz so you can see it:

{%highlight ruby%}
require 'object_graph'
ObjectGraph.new(9).view!
{% endhighlight %}
</figure>

<div class="highlight">
<a href="../images/references.pdf"><img src="../images/references-preview.png"></a>
</div>

I know 9 is a bit of a silly example, but it led to a graph of a good size. If you want to run your own analysis of the graph, the edges are available as `ObjectGraph.new(9).edges`, but don't forget to throw away any references to existing graphs before making a new one!

Installation
============

My [patches](https://github.com/ConradIrwin/ruby/blob/find-references) add both [`ObjectSpace.find_references`](https://github.com/ConradIrwin/ruby/commit/af5266875503d58b9fd16a6748d71649e69af922) and  a new [`ObjectGraph`](https://github.com/ConradIrwin/ruby/blob/find-references/lib/object_graph.rb) standard library. If you want to use it you can either clone my repo and build the `find-references` branch yourself, or use one of the quick methods below:

Using [rvm](http://rvm.io) with a patch:
{%highlight bash%}
curl -L http://git.io/kRIgxw > references.patch
rvm install 1.9.3-p392-fr --patch ./references.patch
{% endhighlight %}

Using [rbenv](https://github.com/sstephenson/rbenv) with ruby-build:
{%highlight bash%}
curl -L http://git.io/4fQ9Jg > 1.9.3-p392-fr
rbenv install ./1.9.3-p392-fr
{% endhighlight %}

Using [chruby](https://github.com/postmodern/chruby) with ruby-build:
{%highlight bash%}
curl -L http://git.io/4fQ9Jg > 1.9.3-p392-fr
ruby-build ./1.9.3-p392-fr /opt/rubies/ruby-1.9.3-p392-fr
{% endhighlight %}

It should be reasonably easy to port these patches to ruby 2 if someone's keen — it might even be possible to make the garbage collector hack less nasty. Please let me know if you're interested :).
