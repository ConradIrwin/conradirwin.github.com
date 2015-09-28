---
title: Smooth animated HTML-5 icons
permalink: blog/smooth-animations.html
layout: post
fuzzydate: September 2015
style: "article{ -webkit-column-count: 1; -moz-column-count: 1; column-count: 1; }"
credit: Vivek Sodera & Rahul Vohra
---

Recently we added our first animated icon to Superhuman as part of the transition to the search interface:

<a style="display: block; text-align: center" href="../images/search.mp4">
    <video style="display: inline; width: 100%" autoplay="true" loop="true">
        <source src="../images/search.mp4" />
    </video>
</a>
<aside>NOTE: The animations in this video are running at 1/10th of normal speed.</aside>

There's clearly more work to be done here, but you get the idea: the icon animates to enforce the perceptual link between the user's click and its effect, while the text flicks right to draw the user's attention to the now visible textbox.

#Using &lt;canvas>

While the text transition can be achieved easily using translate, opacity and transition in CSS, making the icon morph from a magnifying glass to a cross requires significantly more work.

We did this by writing a simple program that can draw the animation into an HTML canvas. The advantage of this approach is that we can re-use d3's easing and scaling functions, and run the animation at any size or speed.

<aside>I've uploaded [the code](https://gist.github.com/ConradIrwin/e6b3d65beab92f1ce19d) behind this icon in case you're curious.</aside>

The disadvantage of using canvas is that we need javascript to compute a new frame every 16ms. In an ideal world, this would not be a problem as we can use `requestAnimationFrame` and rendering each frame takes significantly less than 16ms of CPU-time.

Unfortunately in our more realistic world, the browser is busy trying to render the search UI, or (even more challengingly) the user's Important inbox during the transition. As javascript is single-threaded, this means animation frames may be delayed by significant periods of time while the browser runs unrelated javascript code, or renders chunks of HTML. This causes lag and kills the delightful user experience.

# Accelerating canvas

There are a couple of ways to avoid the single-threaded nature of javascript. In this case the most pertinent is to offload the animation to the graphics card. Unfortunately graphics cards are not good at executing arbitrary javascript, and so we need to re-frame the problem in a manner that the GPU can handle well.

The idea is relatively simple, and borrowed from how pre-digital motion pictures work. We just paint a strip of ‘film’ and pass it in front of an opening really fast (at 60 frames per second). This can be done using a stepped CSS animation:

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

Here's the actual ten frames of the icon we're using right now. Hover over it
to see how it works in slow-motion:

<div class="window">
  <div class="slide"></div>
</div>

# canvas-animation-loader

We use Webpack to build the Superhuman front-end, and so we've created [canvas-animation-loader](https://github.com/ConradIrwin/canvas-animation-loader) to compile these canvas icons to SVGs using the excellent [canvas2svg](https://github.com/gliffy/canvas2svg) library.

This gives us the best of both worlds: We can develop animated icons interactively using canvas and test them at various speeds and scales; and when we're happy with the result webpack automatically builds the GPU-ready SVG versions that are used in production.

I'd love your feedback and comments, please get in touch [conrad@superhuman.com](mailto:conrad@superhuman.com). I'm particularly looking for a lead designer who can make Superhuman the most delightful email experience in the world.
