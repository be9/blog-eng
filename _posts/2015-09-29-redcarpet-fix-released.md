---
layout: post
title:  "Redcarpet fix released"
date:   2015-09-29 00:00:00
comments: true
---

Finally [Redcarpet 3.3.3](https://rubygems.org/gems/redcarpet/versions/3.3.3)
has been released. This version includes [my fix]({% post_url 2015-09-21-memory-leak %}).
If you use Redcarpet, update the version spec in your `Gemfile`:

{% highlight ruby %}
gem 'redcarpet', '~> 3.3.3'
{% endhighlight %}
…and forget about leaks!
