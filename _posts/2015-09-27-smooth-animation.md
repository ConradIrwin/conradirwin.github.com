---
title: Smooth animated HTML-5 icons
permalink: blog/smooth-animations.html
layout: post
fuzzydate: September 2015
style: "article{ -webkit-column-count: 1; -moz-column-count: 1; column-count: 1; }"
---

Recently I added our first animated icon to Superhuman as part of the transition to the search interface:

<a style="display: block; text-align: center" href="../images/search.mp4">
    <video style="display: inline; width: 100%" autoplay="true" loop="true">
        <source src="../images/search.mp4" />
    </video>
</a>
<aside>NOTE: The animations in this video are running at 1/10th of normal speed.</aside>

There's clearly more work to be done here, but you get the idea: the icon animates to enforce the perceptual link between the user's click and its effect, while the text flicks right to draw the user's attention to the now visible textbox.

#Using &lt;canvas>

While the text transition can be acheived easily using translate, opacity and transition in CSS, making the icon morph from a magnifying glass to a cross requires significantly more work.

The way I do this is to write a simple program that can draw the animation into an HTML canvas. The advantage of this approach is that I can re-use d3's easing and scaling functions, and run the animation at any size or speed.

<aside>I've uploaded [the code](https://gist.github.com/ConradIrwin/e6b3d65beab92f1ce19d) behind this icon in case you're curious.</aside>

The disadvantage of using canvas is that I need javascript to compute a new frame every 16ms. In an ideal world, this is no problem. I use requestAnimationFrame and drawing an icon takes a lot less than 16ms of CPU-time.

Unfortunately in our more realistic world, the browser is busy trying to render the search UI, or (even more challengingly) the user's Important inbox during the transition. As javascript is single-threaded, this means that my animation frames may be delayed by significant periods of time while the browser runs unrelated javascript code, or renders chunks of HTML. This causes lagginess and kills the delightful user experience.

# Accelerating canvas

There are a couple of ways to avoid the single-threaded nature of javascript. In this case the most pertinant is to offload the animation to the graphics card. Unfortunately graphics cards are not good at executing arbitrary javascript, and so we need to re-frame the problem in a manner that the GPU can handle well.

The idea is relatively simple, and borrowed from how pre-digital motion pictures work. We just paint a strip of ‘film’ and pass it in front of an opening really fast (at 60 frames per second). This can be done using a stepped CSS animation.

Here's the actual ten frames of the icon we're using right now, slowed down so you can see how it works:

<div>
    <style style="display: block; white-space: pre; font-family: monospace;">
    .window {
        width: 32px;
        overflow: hidden;
    }
    .slide {
        width: 352px;
        height: 32px;
        background-image: url(../images/10frames.svg);
        transform: translateX(0px);
        transition: transform 166ms steps(10);
    }
    .window:hover .slide {
        transform: translateX(-320px);
    }
    </style>
    <style>
        .window {
            overflow: visible !important;
            position: relative;
            height: 34px;
            margin: auto;
        }

        .window::after {
            display: block;
            outline: 3px solid aqua;
            content: " ";
            height: 32px;
            width: 32px;
            position: absolute;
            top: 0;
            left: 0;
        }
        .slide {
            transition: transform 1328ms steps(10) !important;
        }
    </style>
</div>

Here's the actual ten frames of the icon we're using right now, slowed down so
you can see how it works:

<div class="window">
  <div class="slide"></div>
</div>

# canvas-animation-loader

I'm a massive fan of Webpack, which we use to build the Superhuman front-end, and so I've written a custom loader canvas-animation-loader that lets me create these SVGs from canvas icons automatically.

This let's me design animations frame-by-frame at any speed I like, and then have them automatically built into Superhuman as GPU-accelerated assets at runtime.

It's based on the excellent canvas2svg project, so I can write, debug and source-control my animations using canvas, but when we do a production build they get converted into optimized SVGs which can be animated by the GPU.

I'd love your feedback and comments, please get in touch [conrad@superhuman.com](mailto:conrad@superhuman.com). I'm particularly looking for a lead designer who can make Superhuman the most delightful email client in the world.
