---
title: A quick introduction to Teacup
permalink: blog/introducing-teacup.html
layout: post
fuzzydate: June 2012
---

[Teacup](https://github.com/rubymotion/teacup) is a library for
[RubyMotion](http://rubymotion.com/) that let's you create views and layouts in simple
declarative code. The simplest way to understand it is that it's "CSS for iOS", but that
would be to heavily underestimate what it can do. Perhaps a better analogy is "Interface
Builder for RubyMotion", as both tools solve the same problem.

Creating layouts
----------------

Once you've [added teacup to your app](https://github.com/rubymotion/teacup#installation),
you can get started immediately by creating views in your controller:

{% highlight ruby %}
class MyViewController < UIViewController
  attr_accessor :button

  layout do
    subview UILabel,
      text: "Hello World"

    self.button = subview(
      UIButton.buttonWithType(UIButtonTypeCustom),
      title: "Click me!"
    )
  end
end
{% endhighlight %}

Now, whenever `MyViewController` is shown, it will contain both a label and a button.
You'll also notice that I've stashed the button away on `self.button` so that I can refer
to it later in the controller. I won't need to use the label in my code, so I can just
rely on the view heirarchy to retain it as necessary.

<aside>The `subview` method is actually nestable, so if I wanted to I could put
`subview(MyMenuBar){ subview(MyButton1) }` to put a button into a menu. This can come in
handy when you need detailed structures, though in some cases it may be clearer to
encapsulate complicated structures in a `UIView` subclass. For this reason, Teacup's
`layout` method is available not only on `UIViewController` but also `UIView`.</aside>

Making them pretty
------------------

Unfortunately the code in the previous section had a bit of a flaw; when you run it the
label and the button don't actually appear. This is because Cocoa defaults all elements to
zero width and zero height, and is really easy to fix:

{% highlight ruby %}
class MyViewController < UIViewController
  attr_accessor :button

  layout do
    subview(UILabel,
      text: "Hello World",
      top: 60, left: 60,
      width: 200, height: 100,
      backgroundColor: UIColor.blueColor
    )

    self.button = subview(
      UIButton.buttonWithType(UIButtonTypeCustom),
      title: "Click me!",
      top: 200, left: 60,
      width: 200, height: 100
    )
  end
end
{% endhighlight %}

Again this hopefully stating the obvious, but I should now have a blue label a short
distance from the top of the screen and a button just underneath that. Both of them are
60px from the left and 200px wide so that they appear centered.

<aside>Any Cocoa view property that supports key-value coding can be set in this manner,
you can even set up the `delegate` of your button if you'd like. We also support setting
the properties of the view's layers, you just add add a sub-hash:
`layer: { cornerRadius: 10 }`.</aside>

Stylesheets
-----------

While the code in the previous section works just fine, it's somewhat ugly on two fronts.
The most obvious issue is that if I want to change the left margin I have to do that in
multiple places, but also my controller is being filled with irrelevant details. It really
doesn't matter what size the button has, from a controller's point of view, that's a
view-level concern.

To solve these problems teacup provides you with stylesheets:

{% highlight ruby %}
# app/controllers/my_view_controller.rb
class MyViewController < UIViewController
  attr_accessor :button

  stylesheet :iphone

  layout do
    subview(UILabel, :hello_world)
    self.button = subview(UIButton, :click_me)
  end
end
{% endhighlight %}

{% highlight ruby %}
# style/iphone.rb
Teacup::Stylesheet.new :iphone do
  style :widget,
    left: 60,
    width: 200,
    height: 100

  style :hello_world, extends: :widget,
    text: "Hello World",
    top: 60,
    backgroundColor: UIColor.blueColor

  style :click_me, extends: :widget,
    title: "Click me!",
    top: 200
end
{% endhighlight %}

This new code solves both of the code structure problems; the nitty-gritty of making the
view look right is no-longer polluting the controller, and the left-margin is only
specified in one place due to use of the `extends:` meta-property.

<aside>If you're coming from a web-development background it might surprise you that we
put the text into the stylesheet too. This actually makes sense because the text on the
button doesn't matter at all to the controller but it makes a lot of difference to how
the app looks. In the HTML world we're just stuck the wrong way around due to abusing a
markup language to build apps :).</aside>

Further reading
---------------

This blog post has just scratched the surface of what's available in teacup. To get full
information you should read the API docs for
[Stylesheets](http://rdoc.info/github/rubymotion/teacup/master/Teacup/Stylesheet) and for
the [layout function](http://rdoc.info/github/rubymotion/teacup/master/Teacup/Layout).

To get the example code created in this blog post you can clone [the git
repository](https://github.com/ConradIrwin/introducing-teacup). Other example uses of
teacup can be found in [the sample
app](https://github.com/rubymotion/teacup/tree/master/app) that's included with teacup
itself, and [the Commune app](https://github.com/ConradIrwin/Commune) which is where some
of the initial experiments were done.

If you are having problems or want to help us improve teacup, please join in on
[GitHub](https://github.com/rubymotion/teacup) or in
[#teacuprb](irc://irc.freenode.net#teacuprb) on Freenode.
