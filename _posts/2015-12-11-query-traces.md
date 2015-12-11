---
layout: post
title:  "Query traces"
date:   2015-12-11
comments: true
---

What a wonderful time it is, if you work on performance of a Rails application!
There's a bunch of services ([New Relic](http://newrelic.com/),
[Skylight](https://www.skylight.io/)), a great deal of performance-related
gems, like [rack-mini-profiler](https://github.com/MiniProfiler/rack-mini-profiler)
and whatnot.

But sometimes you still have to get back to the basics. My last task at Shuttlerock
was to optimize one of REST API endpoints, reported by Skylight as heavy. By
the way, Skylight is perfect to get a 10,000 feet picture. What's most used?
What's most time- and memory-consuming? The service provides the answers. But later, when
you decide to actually fix something, you need some local tool.

Usually I use rack-mini-profiler in such situations which is nice enough to provide
a list of SQL requests, timings, and corresponding traces. But what to do with REST API?
Rack-mini-profiler apparently can't inject its JS code into JSON responses, can it?

But we still have the log. SQL statements are already there, what about traces?
By the way, it must be noted that traces have become very complex nowadays! It's not
just controllers and models as it was back in 2008. Now it's also
[concerns](http://api.rubyonrails.org/classes/ActiveSupport/Concern.html),
[decorators](https://github.com/drapergem/draper),
[service objects](http://blog.codeclimate.com/blog/2012/10/17/7-ways-to-decompose-fat-activerecord-models/),
[serializers](https://github.com/rails-api/active_model_serializers), and
god knows what else. Finding where the exact call is made has become tricky —
unless you use the [active_record_query_trace](https://github.com/ruckus/active-record-query-trace)
gem! It appends the trace just after an SQL statement in the log.

But there's more to it. Everyone uses memcached, and Shuttlerock is no exception.
Compared to SQL calls, caching has always been considered cheap. Well, it is.
My test on Heroku showed that cache access plus unmarshaling takes _3ms_ for an
ActiveRecord object stored in the cache. However, the REST endpoint I worked on made 20
cache requests when it could make just one. And _60ms_ is already pretty much!
This time would also increase if the dyno becomes more busy.

So, I needed traces for the cache requests too. And I made a gem for that:
[cache-query-trace](https://github.com/be9/cache-query-trace). The
[dalli](https://github.com/petergoldstein/dalli) gem,
a memcached client, does a good job logging all cache requests with the keys,
but if you use standard file cache in development (or memory cache in test environment),
be sure to add this line somewhere:

{% highlight javascript %}
Rails.cache.logger = Rails.logger
{% endhighlight %}

These tools allowed me to eliminate unneeded calls (with SQL, I converted
40 rather dumb calls into one, but more complex). To count calls in the log
I just used the `grep` utility. Viva command line!
