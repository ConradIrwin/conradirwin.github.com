---
title: To garbage collect or not?
permalink: blog/garbage-collection.html
layout: post
fuzzydate: May 2025
---

As many programmers do, I harbor a secret desire to build a "better" programming
language.

One of the things I've come to realize (having used a few of them at this point)
is that your approach to recycling memory has far-reaching ramifications.

There are (roughly) three common approaches in use today:

* Fully managed. Think Go, Python, etc. You allocate memory when you need it, and the garbage collector deals with it for you.
* Mostly managed. Think Swift, Rust, etc. You use a language-provided system of smart pointers to do reference counting, and deallocation happens mostly invisibly (assuming you don't create cycles, or otherwise mess things up)
* Free for all. Think Zig, C, etc. You allocate, you deallocate. You can do whatever you want, and good luck!

It goes without saying that all these languages have tight integration with C, and so you can use C-style malloc/free in all of them. Similarly, all are complete programming languages, so you can build (and use) a garbage collector.

But in reality, none of these approaches composes very well, and so within a given language you're mostly stuck in a single paradigm.

If you're in a world where reference counting takes care of things, you must be careful to avoid cycles. In a fully garbage collected world (or a free-for-all), you can create cycles all you like.

On the flip side, reference counting lets you rely on finalizers because you can know they are run "at the right time". In garbage collected languages you can only use a finalizer "best-effort" because collection times are not guaranteed. (And in a world where you're manually calling free, you can be sure that you're also manually calling the finalizer).

Either way, if you write any non-trivial code, you need to know which paradigm it will be run under. (This is also true for things like bump allocation, if you have an array implementation that reallocs on every grow you're going to really mess your heap up).

From an ergonomics point of view, automatic garbage collection is very attractive. Memory is a finite resource, and its really nice to not have to think about free'ing things. It comes with some obvious downsides though, the primary one being that you typically need either a read or write barrier on every pointer access, and a background task using up CPU and memory bandwidth just to collect garbage periodically. (An old objection was long stop-the-world pauses, but modern garbage collectors do not have this problem).

From a purity perspective there's also a lot to like about the "free-for-all" approach. You made the garbage, it's your responsibility. That said, experience with C and C++ has shown that without tooling support it's quite hard to make this work in practice, because it's oh-so-easy to forget to free.

The middle ground is exactly that: you get good tooling that ensures that most of the time garbage collection is not something you need to think about; but you're left with a few sharp edges. Tracking down memory leaks in any application is not fun, and it takes (potentially unnecessary) brain cycles to ensure you're not accidentally creating them when writing code.

So, what's a budding language designer to do?

Well, it depends :D.

My bet is that as more code is being built with LLMs, we're going to see less languages taking the middle road. It's hard for an LLM to avoid creating a reference cycle (exactly as it is for humans) because it can't track the ownership very easily between different subsystems. In a garbage collected world, this doesn't matter; and in a world where deallocation is explicit, it usually happens "paired" with allocation so it's (relatively) easier for an LLM to add.

I also think that as the amount of RAM we have increases (and the number of CPU cores) the costs of garbage collection are reduced (except for those pesky barriers). As always, we software engineers can happily eat into the performance gains of the exceptional hardware we have today.

But I am curious if there's a better way...

I am interested in the practicality of building a concurrent GC without the need for read/write barriers. I am curious if linear typing can help with forgetting to free. I am interested in ideas where you mostly use reference counting, but you can do this at a course-grained level so you can still have reference cycles within a subset of nodes in your memory graph... and in general to know what else is out there.
