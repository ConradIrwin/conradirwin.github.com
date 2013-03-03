---
title: Tracking down memory leaks in Ruby
permalink: blog/find-references.html
layout: post
fuzzydate: Mar 2013
---

Memory leaks are my least favourite types of bug to track down. They require a detailed knowledge of the entire codebase, strong intuition and a fair amount of luck to spot. Unfortunately they do happen, and I spent a fair amount of last week tracking one down. To make it easier I wrote a patch for ruby that let me visualize portions of the memory graph.

ObjectSpace.each_object
=======================

Ruby already comes with `ObjectSpace` which contains a few methods for analyzing your program. The most useful is `ObjectSpace.each_object` which yields every single ruby object in your program.

{%highlight ruby%}
counts = Hash.new{ 0 }
enum = ObjectSpace.each_object
enum.each_with_object(counts) do |o, h|
  h[o.class] += 1
end
{% endhighlight %}

By dumping the counts into a file after each request and using `diff`, I was able to determine what kind of objects were leaking. In this case, it seemed to be our Controller objects along with various handlers and helpers that are mutually linked to them. As we have several objects that collaborate and form a cycle, it wasn't yet obvious which of them was actually being retained, let alone what was retaining it.

ObjectSpace.find_references
===========================

Ruby already keeps track of which objects retain each other so that the garbage collector can accurately mark all objects that are still in use. This information isn't made available at the Ruby level, so the first step was to add a `find_references` method to `ObjectSpace` that would tell me which objects are directly referencing the object I'm looking for.

{%highlight ruby%}
ObjectSpace.find_references(
  ObjectSpace.each_object(MainController).first
)
{% endhighlight %}

This works but it turns out to be far too fiddly for manual use while debugging. The main problem is that every time you run `find_references` you get back an Array that contains a lot of new references. After a few iterations of this, you have to throw everything away and start again.

ObjectGraph#view!
=================

A memory leak happens because you accidentally create a reference from a long-lived object to a short-lived object. Long-lived objects are things like classes and globals and their instance variables that should go on existing until the program exits, short-lived objects are those that should be garbage collected as soon as they're finished with.

`ObjectGraph#view!` renders the graph of which objects reference which objects using graphviz. It starts at the object that you are looking for, and recurses backwards until it's found everything. By inspecting the graph you can easily spot the transition from long-lived objects to short-lived objects, and it should be simple from there to find the relevant bug in the source code.

<figure>

{%highlight ruby%}
require 'object_graph'
ObjectGraph.new(9).view!
{% endhighlight %}
</figure>

<div class="highlight">
<a href="../images/references.pdf"><img src="../images/references-preview.png"></a>
</div>

While 9 is a silly example, because there are only long-lived objects in this graph, it's a reasonably complete demonstration. You'll notice that not only is each node labelled with the inspect output for the object, each edge is also labelled in a way that corresponds to how the reference was made. If you need programattic access to this, then `ObjectGraph#annotated_edges` contains all the same information, but in computer-readable form.

Installation
============

My [patches](https://github.com/ConradIrwin/ruby/commits/find-references) add both [`ObjectSpace.find_references`](https://github.com/ConradIrwin/ruby/commit/af5266875503d58b9fd16a6748d71649e69af922) and  a new [`ObjectGraph`](https://github.com/ConradIrwin/ruby/blob/find-references/lib/object_graph.rb) standard library. If you want to use them you can either clone my repository and build the `find-references` branch yourself, or use one of the quick methods below:

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
ruby-build ./1.9.3-p392-fr ~/.rubies/ruby-1.9.3-p392-fr
{% endhighlight %}

You'll also need to install graphviz if you want to see the output, try `brew install graphviz` or `apt-get install graphviz`.

It should be reasonably easy to port these patches to ruby 2 if someone's keen — it might even be possible to make the garbage collector hack less nasty. The other improvement I want to add is naming the edges that exist for local variables. Please let me know if you're interested :).
