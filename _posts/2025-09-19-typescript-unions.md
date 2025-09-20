---
title: What if typescript was right about union types...?
permalink: blog/typescript-unions.html
layout: post
fuzzydate: Sept 2025
---

One thing I love about Rust is the exhaustive pattern matching. The killer use case of this is preventing nil pointer deferences. You never have an invalid pointer, you represent that as an `Option<T>`.

That said, Rust's enums (on which this feature are based) feel very heavy.

The type signatures are too precise: it is rare to care about the difference between an `Option<Result<T, Error>>` and a `Result<Option<T>, Error>`); the syntax for destructing is very different from the syntax to check (`if user.is_some()`  does not resemble `if let Some(user) = user`); and unpacking an enum requires introducing a new local variable.

What if instead of `Option<T>` we had `T | None`?

Then both `Result<Option<T, Error>>` and `Option<Result<T, Error>>` could be `T | None | Error`. Instead of destructing pattern matching, we pull in typescript's ability to restrict the type based on the surrounding conditionals.

# More ergonomic syntax

{% highlight rust %}
// before
fn do_foo(user: Option<User>) -> Option<Foo>
  if let Some(user) = user {
    Some(user.foo())
  } else {
    None
  }
}

// after
fn do_foo(user: User | None) -> Foo | None
  if user is User {
    user.foo() // user has type User
  } else {
    None // user has type None
  }
}
{% endhighlight %}

# Unification

What if in a later refactoring we change the type of `user.foo()` to return an `Option<Foo>`?

In the old code we'd have to remove the explicit call to `Some`, or update the return type to be an `Option<Option<Foo>>`

In the new code, we don't have to change anything. The first branch returns `Foo | None` and the second branch returns `None`. Although the type is technically `(Foo | None) | None` the normal unification rules for the `|` operator apply, meaning that this type is indistinguishable from `Foo | None`.

This is almost always what you want, which is why the `?` operator in Rust is so useful.

# Disunity

Let's imagine a hypothetical case where I don't want this unification to happen.

{% highlight rust %}
// returns a anyhow::Error if cache lookup fails
fn cached<K, V>(key: K, f: impl Fn() -> V)
    -> Result<V, anyhow::Error> { ... }
// returns a anyhow::Error if expensive work fails
fn expensive_work() -> Result<usize, anyhow::Error>

let x = cached(key_1, || {
  expensive_work()
})
{% endhighlight %}

Hypothetically, I want to be able to tell the difference between the cache lookup failing and the expensive work failing. In the old world this is easy, the type is `Result<Result<T, anyhow::Error>, anyhow::Error>`, and I can match on that.

In the new world, the type is unified to just `T | anyhow::Error`.  To distinguish between the two error cases, we need a wrapper type that is guaranteed to be distinct from the error type.

{% highlight rust %}
type Some<T> {
  some: T
}
fn cached<K, V>(key: K, f: impl Fn() -> V)
    -> Some<V> | anyhow::Error { ... }
{% endhighlight %}

Now the return type is `Some<usize | anyhow::Error> | anyhow::Error`, so I can unpack the error cases as before:

{% highlight rust %}
match x {
  Some<usize> => { println!(x.some) } // value
  Some<anyhow::Error> => { println!(x.some) } // error from work
  anyhow::Error => { println!(x) } // error from cache layer
}
{% endhighlight %}

# Namespacing

One advantage of Rust's enums is they give you a new namespace. If we have these unions, then we have to define the underlying types directly:

{% highlight rust %}
// before
enum Event {
    PageLoad
    MouseDown(f32, f32),
    MouseUp(f32, f32),
}

// after
struct PageLoad;
struct MouseDown(f32, f32);
struct MouseUp(f32, f32);
type Event = PageLoad | MouseDown | MouseUp;
{% endhighlight %}

In practice, this advantage quickly disappears. Because rust doesn't let you refer to [enum variant types](https://github.com/rust-lang/lang-team/issues/122) directly, for any non-trivial enum you end up with a struct value corresponding to each arm.

It also is not used for the two most common enums! The prelude imports `Result` and `Option`, but also imports `Some`, `None`, `Ok`, and `Err` directly into your namespace so that you don't need to do `if let Option::Some(user) = user`.

If we did want to solve the problem of needing to update two parts of the file to add a new variant, we might be able to allow some new syntax to allow that (and re-use some old namespacing syntax)

{% highlight rust %}
mod event {
  enum Event =
    struct PageLoad |
    struct MouseDown(f32, f32) |
    struct MouseUp(f32, f32);
}
{% endhighlight %}

It's also worth noting, in the new world, you don't need `Option` or `Result`. They are `T | None` or `T | U`. You also don't need `Ok` in addition to `Some` (they're the same thing). You do still need `None` (and although I can invent cases where `Err` would be nice, it seems *very* uncommon). So you go from 6 exported names to 2 (just `None` and `Some`).

# Methods

Another thing you lose is the ability to define methods on an enum, because they don't exist as distinct types. Instead you'd need to create a wrapper type that has the methods you want to export:

{% highlight rust %}
pub struct Event {
  event: PageLoad | MouseDown | MouseUp
}

impl Event {
  fn x_position(&self) -> f32 | None {
    match self.event {
      PageLoad => None,
      event @ MouseDown => event.0,
      event @ MouseUp => event.0
    }
  }
}
{% endhighlight %}

# What else?

I'm sure there are more implications of this than I'd thought through, but it kind of seems like a nice balance.

And I probably wouldn't go as far as typescript and let you access any incidentally shared fields or call methods on these types directly: you need to restrict it to a concrete type first.
