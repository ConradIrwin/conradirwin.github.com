---
title: Everything you ever wanted to know about constant lookup in Ruby
permalink: blog/constant-lookup.html
layout: post
fuzzydate: November 2012
credit: Martin Kleppmann, John Mair
style: "article{ -webkit-column-count: 1; -moz-column-count: 1; column-count: 1; }"
---

One of the best things about Ruby is that it just does what you mean. The downside of this
is that if you're dealing with a fiddly situation, it can be somewhat hard to work out
exactly what it will do.

This is particularly true of constant lookup. To access a constant in ruby, you just use
its name: `FooClass`. But how does that actually work?

<aside>Although the answer turns out to be reasonably straightforward, I've included a lot
of examples. It's easy to tell you that constant lookup searches for constants that are defined in
`Module.nesting`, `Module.nesting.first.ancestors`, and `Object.ancestors` if
Module.nesting.first is nil or a module, but that doesn't really help with understanding.</aside>

Module.nesting
==============

To a first approximation, Ruby looks for constants attached to modules and classes in the
surrounding lexical scope of your code, for example:

{% highlight ruby %}
module A
  module B; end
  module C
    module D
      B == A::B
    end
  end
end
{% endhighlight %}

It first looks for `A::C::D::B` (doesn't exist), then `A::C::B` (still doesn't exist), and
then finally `A::B` (which does).

Like almost everything in Ruby, the chain of namespaces that will be searched (known to
l33t hackers as the `cref`) is introspectable at runtime with `Module.nesting`:

{% highlight ruby %}
module A
  module C
    module D
      Module.nesting == [A::C::D, A::C, A]
    end
  end
end
{% endhighlight %}

If you've ever tried to take a short-cut when re-opening a module, you may have noticed
that constants from skipped namespaces aren't available. This is because the outer
namespaces are not added to `Module.nesting`.

{% highlight ruby %}
module A
  module B; end
end

module A::C
  B
end
# NameError: uninitialized constant A::C::B
{% endhighlight %}

In order to find `B`, `Module.nesting` would have to include `A`, but it doesn't. It only
includes `A::C`.

Ancestors
=========

If the constant cannot be found by looking at any of the modules in `Module.nesting`, Ruby
takes the currently open module or class, and looks at its ancestors.

{% highlight ruby %}
class A
  module B; end
end

class C < A
  B == A::B
end
{% endhighlight %}

The currently open class or module is the innermost `class` or `module` statement in the
code. A common misconception is that constant lookup uses `self.class`, which is not true.
(btw, it's not using the [default definee](http://yugui.jp/articles/846) either, just in
case you wondered):

{% highlight ruby %}
class A
  def get_c; C; end
end

class B < A
  module C; end
end

B.new.get_c
# NameError: uninitialized constant A::C
{% endhighlight %}

Object::
========

`Module.nesting == []` at the top level, and so constant lookup starts at the currently
open class and its ancestors. While there's no `class` or `module` statement that you can
see, it is taken for granted that at the top level of a ruby file the currently open class
is Object:

{% highlight ruby %}
class Object
  module C; end
end
C == Object::C
{% endhighlight %}

Although I've not explicitly said it yet, you've probably noticed that newly defined
constants also get defined on the currently open class. So constants you define at the top
level end up attached to `Object`.

{% highlight ruby %}
module C; end
Object::C == C
{% endhighlight %}

This in turn explains why top-level constants are available throughout your program.
Almost all classes in Ruby inherit from Object, so Object is almost always included in
the list of ancestors of the currently open class, and thus its constants are almost
always available.

That said, if you've ever used a `BasicObject`, and noticed that top-level constants are
missing, you now know why. Because `BasicObject` does not subclass `Object`, all of the
constants are not in the lookup chain:

{% highlight ruby %}
class Foo < BasicObject
  Kernel
end
# NameError: uninitialized constant Foo::Kernel
{% endhighlight %}

For cases like this, and anywhere else you want to be explicit, Ruby allows you to use
`::Kernel` to access `Object::Kernel`.

Ruby assumes that you will mix modules into something that inherits from `Object`. So if
the currently open module is a module, it will also add `Object.ancestors` to the lookup
chain so that top-level constants work as expected:

{% highlight ruby %}
module A; end
module B;
  A == Object::A
end
{% endhighlight %}

`class_eval`
============

As mentioned above, constant lookup uses the currently open class, as determined by
`class` and `module` statements. Importantly, if you pass a block into `class_eval` or
`module_eval` (or `instance_eval` or `define_method`), this won't change constant lookup.
It continues to use the constant lookup at the point the block was defined:

{% highlight ruby %}
class A
  module B; end
end

class C
  module B; end
  A.class_eval{ B } == C::B
end
{% endhighlight %}

<aside>This was not true in ruby 1.9.1, and is perhaps the main reason that that
particular version was quick to fade out of existence. Many DSLs rely on instance eval'ing
the programmer's code in internal contexts, and it was considered very confusing to break
access to the programmer's own constants.</aside>

Confusingly however, if you pass a String to these methods, then the String is evaluated
with `Module.nesting` containing just the class itself (for `class_eval`) or just the
singleton class of the object (for `instance_eval`).

{% highlight ruby %}
class A
  module B; end
end

class C
  module B; end
  A.class_eval("B") == A::B
end
{% endhighlight %}

Other gotchas
=============

Finally I want to point out that if you're in a singleton class of a class, you don't get
access to constants defined in the class itself:

{% highlight ruby %}
class A
  module B; end
end
class << A
  B
end
# NameError: uninitialized constant Class::B
{% endhighlight %}

This is because the ancestors of the singleton class of a class do not include the class
itself, they start at the `Class` class.

{% highlight ruby %}
class A
  module B; end
end
class << A; ancestors; end
[Class, Module, Object, Kernel, BasicObject]
{% endhighlight %}

In a similar vein, it may also worth noting that superclasses of things in
`Module.nesting` are ignored.  For example:

{% highlight ruby %}
class A
  module B; end
end

class C < A
  class D
    B
  end
end
# NameError: uninitialized constant C::D::B
{% endhighlight %}

Summary
=======

Constant lookup in ruby isn't actually that hard after-all. It just looks at the lexical
nesting of `class` and `module` statements. You can calculate the currently open class by
using the first value in `Module.nesting`, or defaulting to `Object` if that array is
empty.

Here's some code that accurately replicates Ruby's inbuilt constant lookup. You'll notice
that I have to use binding.eval with a String so that `Module.nesting` is taken from the
binding object and not the block :).

{% highlight ruby %}
class Binding
  def const(name)
    eval <<-CODE, __FILE__, __LINE__ + 1
      modules = Module.nesting + (Module.nesting.first || Object).ancestors
      modules += Object.ancestors if Module.nesting.first.class == Module
      found = nil

      modules.detect do |mod|
        found = mod.const_get(#{name.inspect}, false) rescue nil
      end
      found or const_missing(#{name.inspect})
    CODE
  end
end
{% endhighlight %}
