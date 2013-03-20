---
title: How to program like an explorer (with Pry!).
permalink: blog/explore-with-pry.html
layout: post
credit: John Mair, Ryan King
fuzzydate: June 2012
---

<aside>I'm one of the core committers to the [Pry REPL](http://pry.github.com/). Talking about this at
work a few weeks ago, a colleague asked me: "Why should I use pry? I mean, I
know it's better, but what's it for?". This is a somewhat long-winded attempt to
answer that question.</aside>

Exploring is a large part of what programmers do during the time when we're not
typing out code. Attempting to discover the correct way to do something, or to
get a feeling for how something works is an essential part of the job, and for
many people it's also the most fun.

While these aren't the only examples, I classify the following as exploring:

1. Trying out new libraries.
2. Designing good APIs.
3. Debugging existing programs.
4. Exploring data sets.

There are several passive ways to solve these tasks. You can try Google, and see
if someone else has blogged about the problem, or try reading lots of existing
code or documentation in the hope that inspiration strikes. A more active
approach however is to write some code.

Exploring with code
-------------------

Writing code to explore problem domains is the best way to get tangible
feedback. If you're designing an API, but don't try using it, your API will not
be very good. If you're trying to learn a new library, but don't actually use
it, you'll not gain much. Finally, if you are trying to debug without writing
code; you are trying to emulate an entire computer with your mind, which is
pretty tiring.

The code you write while exploring will be different from the code you write
when building applications. Instead of the aim being an application you can
deploy, the aim is to improve your mental model. This in turn means that you
need to treat code differently: good exploratory code is designed to be written
quickly, run once, and to give you plenty of output that you can understand what
happened. It's expected to be buggy, and you're expected to just keep adding
hacks until it works.

<aside>Obviously, application code is optimized in completely the other
direction. It should be written thoroughly as it is run and read many times, it
should also be reasonably free of hacks, and output only a small amount of
critical information.</aside>

Programming in a REPL
---------------------

Read-evaluate-print loops are an invaluable tool for exploratory coding. They
have features that are so obvious to enable this that you probably didn't notice
them before:

1. Intermediate values generated are always displayed.
2. Running programs can be inspected in arbitrary detail.
3. You can always decide which line of code to execute next.

Contrast this to a normal development cycle of using a text-editor and then
running the program from the start. In that process the computer mindlessly
re-evaluates the entire program, dumps out a tiny amount of information at any
`print` statements you've inserted, and then exits, throwing away the entire
state.

Using Pry as a REPL
-------------------

Pry is a REPL designed with these features in mind. Taking inspiration from Smalltalk and
SLIME, it embraces the idea of exploratory programming and helps you to do it more
effectively. While it'd take me too long to cover [all of its
features](https://github.com/pry/pry/wiki), here are the two I consider most important:

<figure><h3>`ls`</h3>

<div class="highlight"><pre style="white-space: normal;"><code clas="ruby" style="white-space: normal">
[1] pry(main)&gt; ls <span class="bold"><span class="f4 underline">CodeRay</span></span>
<br/><span class="bold">constants</span>: CODERAY_PATH  <span class="f3">Duo</span>  <span class="f4">Encoders</span>  <span class="f3">FileType</span>  <span class="f3">GZip</span>  <span class="f4">Plugin</span>  <span class="f4">PluginHost</span>  <span class="f4">Scanners</span>  <span class="f3">Styles</span>  <span class="f3">TokenKinds</span>  <span class="f4">Tokens</span>  <span class="f4">TokensProxy</span>  VERSION  <span class="f4">WordList</span>
<br/><span class="bold">CodeRay.methods</span>: coderay_path  encode  encode_file  encode_tokens  encoder  get_scanner_options  highlight  highlight_file  scan  scan_file  scanner
</code></pre></div>
</figure>

If you've ever wondered what a given object can do, the
[`ls`](https://github.com/pry/pry/wiki/State-navigation#wiki-Ls) command will
help you. Additionally, and along with the
[`find-method`](https://github.com/pry/pry/wiki/State-navigation#wiki-Find_method)
command, it can also answer questions like "How do I use this API to achieve X?".

<figure><h3>`show-source`</h3>

<div class="highlight"><pre style="white-space: normal;"><code clas="ruby" style="white-space: normal">
[2] pry(main)&gt; $ binding.pry
<br/><span class="bold"><span class="f1">def</span></span> <span class="bold"><span class="f4">pry</span></span>(*args)
<br/>&nbsp;&nbsp;<span class="bold"><span class="f1">if</span></span> args.first.is_a?(<span class="bold"><span class="f4"><span class="underline">Hash</span></span></span>) || args.length == <span class="bold"><span class="f4">0</span></span>
<br/>&nbsp;&nbsp;&nbsp;&nbsp;args.unshift(<span class="bold"><span class="f6">self</span></span>)
<br/>&nbsp;&nbsp;<span class="bold"><span class="f1">end</span></span>
<br/>
<br/>&nbsp;&nbsp;<span class="bold"><span class="f4"><span class="underline">Pry</span></span></span>.start(*args)
<br/><span class="bold"><span class="f1">end</span></span>
</code></pre></div>
</figure>

The
[`show-source`](https://github.com/pry/pry/wiki/Source-browsing#wiki-Show_method)
command, aliased to `$`, allows you to see how something is implemented. This is
vital both for debugging, and also for exploring new libraries. The
[`edit-method`](https://github.com/pry/pry/wiki/Editor-integration#wiki-Edit_method)
command can be used in combination to fix bugs you spot on the fly.

<aside>You'll notice in both of the screenshots that Pry loves colours. They're
used in the `ls` command to distinguish different kinds of constant (classes are
blue, exceptions are pink, and those pending autoload are yellow); and anywhere that
code is displayed, including old prompts, to enhance readability.</aside>

Summary
-------

Acknowledging the existence of temporary exploratory code as distinct from
permanent application code has huge benefits. When you're comfortable with
writing code that will be immediately thrown away, you can iterate purposefully
to find solutions instead of relying on google and inspiration.

A REPL is the best tool to help you do this. Unlike your text editor which is
designed for writing permanent robust code, a REPL is designed to help you write
temporary code to quickly understand problem domains. Exploration tasks where
this is most helpful include learning new libraries, designing APIs, debugging,
data analysis, smoke testing, etc.

Pry is a world-class REPL. It's written in Ruby, and due to its extensive
wrapping of Ruby's introspection APIs excels for exploring in Ruby. Due to the
JVM's homogeneous object model and JRuby we're also beginning to see Pry used
as an exploration tool for non-ruby
projects<sup>[1](https://wiki.jenkins-ci.org/display/JENKINS/Pry+Plugin)</sup>.

And so that is why you should use [Pry](http://pry.github.com) :).
