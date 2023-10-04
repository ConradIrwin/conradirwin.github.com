---
title: What does = do in rust?
permalink: blog/rust-assignment.html
layout: post
fuzzydate: Oct 2023
credit: Mikayla Maki, Julia Risley, Kirill Bulatov
---

I have been learning rust for the last few months and, like many others, have found it hard to develop an intuition for when the borrow checker will disallow what I’m trying to do.

I had an insight recently that may be helpful to others who are switching to rust from other languages, and I wanted to share it.

In prior languages (for me, a hodgepodge of Go, Javascript and variants, Ruby, Python, and a bit of C), the = operator has usually had relatively straightforward semantics: `a = b` gives you a new copy of the value `b` and stores it in `a`. (With some hand-waving for whether `b` is a pointer under the hood or not).

In rust, this is *not* the case, and figuring out the difference made it easier for me to predict the outcome of borrow checking.

### Three assignments for beginners

In rust `=` does three similar (but different) things depending on the type of the expression on the right:

1. `a = b` “move”s `b` into `a` (it removes it from `b`, so you cannot use `b` again!) by default.
2. If `b` implements the `Copy` trait, then  `a = b` makes a “copy”  of `b` in `a`. (this is the case for most primitive types like numbers, as well as small usually-immutable custom types).
3. If `b` is a reference, `a = b` does a “reborrow” of `b` into `a` if `b` instead.

I should clarify that this is not limited to the `=` operator: passing a value to a function, or returning a value from a function acts the same way. It helps me to think of those as just syntactically hidden assignments under the hood.

(I also want to be clear that `b` can be any expression, not just a variable. It’s the type of the expression that matters).

### RAII-ge against the machine

Rust takes an opinionated approach to garbage collection (as you probably know), inspired by the [RAII](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization) principles developed for C++:
* Each value lives from the point it is created to the point it goes out of scope, at which point it is dropped.
* The compiler would like to (as far as possible) be able to statically determine when it is dropped.
This is why the “move” semantics for `=` exist. If you allowed people to make a copy of a value that owns something else, then you’d end up with 2 owners, and it would be even more difficult to statically determine when the value should be dropped. (Besides you get some wins about “fearless concurrency” or something).

The `Copy` trait is a way of opting out of this insanity. In exchange for promising the compiler that there’s nothing funny going on (your value doesn’t need any special care when being dropped, and it can be cheaply copied) you get to use `=` like a normal person.

### Reborrowing…

In my first few weeks of rust, my intuition with borrow checker errors was to try and add more explicit lifetimes. This was a bad intuition: at least in the code I work with, very few explicit lifetimes are needed.

The confusion arises because the `=` operator handles references specially:
* When you reborrow an immutable reference `a = b` you get another reference to referent of `b`. You are only allowed to do this if `a` is dropped before the referent of `b`.
* When you reborrow a mutable reference `a = b`, you take control of the referent `b` . You are only allowed to do this if `a` is dropped before the referent of `b`; *and* you cannot use `b` again until `a` is dropped.

There is a little [more nuance](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html) to it than this, but one of the things that catches me out is that rusts notion of “before” in these statements is fuzzy. Sometimes moving borrows to a different statement fixes the code even though the order of execution doesn’t change. If `b` is a mutable reference then `a(b, c(b))` is sometimes disallowed even though `let tmp = c(b); a(b, tmp)` is allowed.

### Pointing and laughing

Another thing I struggled with is to figure out how rusts “smart” pointers interact with all of this. What is an `Rc`, or an `Arc`, or a `Box` anyway, and how on earth do they work?

The answer is that they use `unsafe` under the hood, so they are built using primitives to which these rules do not apply. They can do whatever they want. (In particular, they are allowed to make copies of values that point to values).

Although I initially saw these types as maybe a bit of a code-smell, it turns out that there are many valid programs where the lifetime of a value is not statically determinable; and that is why these types exist. Use them!

But… beware the `=` operator.

If you have `a = b` and `b` is a smart pointer, the `=` is doing a move – that means you won’t be able to use `b` afterwards. This seems to defeat the point of having a reference count…

Because this is a common problem with all similar types, there is another trait `Clone` that smart pointers (among others) implement to get "clever" copy semantics. Instead write `a = b.clone()`.

Clone is a bit of a misnomer, because it is not actually “just” copying under the hood. For example an `Rc` is a reference counted pointer so `Rc::clone` increments the reference count, and returns you a new pointer to the `Rc`. When the returned pointer is dropped (which it will be, thanks RAII!) it will decrement the reference count.

### In summary

Rust is a complicated language built on top of some (comparatively) simple ideas. When you can see through the syntax to what’s going on under the hood, you’ll have a much better time.

At least for me (so far) the “aha” moment was that `=` is another example of this! In order to make your code look a little more sensible rust overloads one humble operator with three different operations, the downside is that you have to pay attention to discover which one you’re getting.

Until next time, when we'll have to mention that `=` can be used for pattern matching too (and it can be affected by `Deref` coercions... (and anything else I missed...!)).
