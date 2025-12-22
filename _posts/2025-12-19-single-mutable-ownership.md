---
title: Single mutable ownership
permalink: blog/single-mutable-ownership.html
layout: post
fuzzydate: Dec 2025
---

The best feature of Rust is the single-mutable reference.

The way this is implemented in Rust is via the borrow checker. Each variable is
either owned, a shared (immutable) reference, or mutably borrowed.

What's interesting about references in Rust is they really do two things:

1. They track permissions (am I allowed to mutate this or not)
2. They enable pass-by-reference instead of pass-by-value.

In a hypothetical simpler language, the decision of whether to pass-by-reference
or not could be made by the compiler. In the 90% case, you want structs larger
than some small size (say 24bytes on a modern arm processor) to be passed by
reference and smaller things to be passed by value.

Languages like Javascript do something similar (distinguish "primitive" types,
that are always passed by value from "object" types that are always passed by
reference).

But those languages lose an important feature: if you have a reference to an
object, you're allowed to mutate it. There's no enforcement of single mutable
ownership.

Languages like Swift take a different path. Instead of passing a mutable
reference, you can use an `inout` parameter. Conceptually the in-out parameter
passes the entire value in, and then at the end of the call, passes the entire
(potentially modified) value out again.

The nice thing about that, is that it (on the surface) gets rid of the need for
mutable references. You can describe this semantic as desugaring a call: `fn
a(inout b)` is "rewritable" to `fn a(b) -> b`, and all calls are rewritten
similarly.

Another advantage of Rust's approach over Swift's approach is that you can
create structs that contain references to other structs. This comes up from
particularly when implementing Iterators; but is otherwise relatively rare.

I think this is solvable the same way. You could imagine `struct Foo { bar: inout Bar }`.
It has the same meaning: the `bar` field takes temporary ownership of the `Bar` value,
but it has to give it back at the end. It's less clear that we can get away with
the layout problems in this case. Struct (and Array) layout tends to matter more than
calling convention because the number of elements may be much higher.

That said, I'm tempted to wait and see. Even in Rust, arrays of references are
relatively rare, and so it may be a good simplification overall to just handle
this for you.
