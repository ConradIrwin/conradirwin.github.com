---
title: How to avoid hash-injection attacks in MongoDB.
permalink: blog/hash-injection.html
layout: post
fuzzydate: January 2014
---

[MongoDB](http://mongodb.com) is a popular document store. Its query API neatly
sidesteps SQL-injection attacks by not being SQL. Unfortunately when using a
framework like Rails, it's easy to build an app that's vulnerable to a
hash-injection attack.

Hash-injection is much less severe than SQL-injection, because an attacker is
more limited in the kind of changes he can make, but it can lead to
authentication-bypass, denial of service, or timing attacks.

I reported this to MongoDB as
[SECURITY-90](https://jira.mongodb.org/browse/SECURITY-90), and they recommend
fixing this by patching your application as described below. Alternatively you
can install my [mongoid-rails](https://github.com/ConradIrwin/mongoid-rails)
gem which automatically fixes this for you.

The Problem with Rails
----------------------

The underlying problem is that Rails query parameters are not strongly typed.
This was first noticed in early versions of Rails when people were writing
code that blindly passed parameters to the update function of a model.

{%highlight ruby%}
user.update(params[:user])
{% endhighlight %}

A devious attacker could add `?user[is_admin]=1` to the URL or the form, and
make themselves an admin. This problem eventually led to the adoption of
[strong parameters](https://github.com/rails/strong_parameters) in rails 4
which protects you against this kind of attack.

<aside>Rails is not the only framework with this problem. PHP has it, Node.js's
express web framework does the same, and I'm certain there are many others.</aside>

The Problem with MongoDB
------------------------

The rails problem is exacerbated by the MongoDB API. MongoDB lets you either
query by equality with a String or using a query operator nested in a Hash:

{%highlight ruby%}
User.where(email: params[:email])

User.where(email: {"$regex" => params[:search]})
{% endhighlight %}

Unfortunately, Rails doesn't guarantee that `params[:email]` is a String.
This means that a devious attacker can again change the structure of the
URL to get an unexpected result.

They can turn a look-up by String into a look-up by regex by passing a
parameter like `?email[$regex]=.*@google.com`.

The consequences
----------------

This is much less severe than an SQL injection attack would be, but it still has several dangerous consequences.

1. <b>Authentication Bypass</b>. If you're verifying API tokens using a query like `User.where(api_token: params[:api_token])`, an attacker could use this to access the API without having an API token by passing in a regex.
2. <b>Denial of Service</b>. If you have a large table with an index on `_id` and you do a query like `BlogPost.find(params[:id])`, an attacker can craft a query that forces MongoDB to do a full table scan.
3. <b>Data leakage</b>. An attacker who can force you to do a full table scan on your users table can use a regex search to find out which domains are signed up to your site by seeing how fast the query returns. This is an example of a [timing attack](https://en.wikipedia.org/wiki/Timing_attack)

The solution
------------

If you're using Rails 3 with strong parameters and Mongoid then the quick fix
is to use my [mongoid-rails](https://github.com/ConradIrwin/mongoid-rails) gem.
This protects you from hash-injection attacks both when querying and when
updating.

For everyone else, I'm afraid the only way to fix this is to manually call
`.to_s` on the parameters. You should find most instances of the problem if you
grep for `where.*params`, `find.*params`, `creates.*params` and
`update.*params`.

{%highlight ruby%}
# These examples are safe, because params[:event_id] and
# params[:auth_token] are coerced to strings.
Event.find(params[:event_id].to_s)
Account.where(auth_token: params[:auth_token].to_s).first
{% endhighlight %}
