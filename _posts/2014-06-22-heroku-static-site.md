---
title: How to serve a static site from Heroku
permalink: blog/heroku-static-site.html
layout: post
fuzzydate: April 2014
---

Heroku provides an awesome hosting service for all your apps. But it turns out
that you can use it to serve static pages too.

There are a variety of tricks I've seen, but most of them seem to involve
complicated hackery, this solution uses ruby's builtin web-server: it's not going
to be the most scalable site in the world, but it's more than enough considering
that it's totally free :).

Solution
--------

Heroku is controlled by a [Procfile](http://ddollar.github.io/foreman/#PROCFILE), which tells it what to run when your app is
deployed. To serve a static site, this is what it should look like:

```bash
# ./Procfile

# Use ruby's builtin library tp run an HTTP server
# that listens on $PORT and serves the current directory.
web: ruby -run -e httpd -p $PORT .
```

And that's it. Just include that Procfile in your git repository then run:

```bash
heroku create <name-of-site>
git push heroku master
```

And your site will be live, you can visit it by running:

```bash
heroku open
```
