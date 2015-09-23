---
layout: post
title:  "How I spent two weeks hunting a memory leak in Ruby"
date:   2015-09-21 19:00:00
comments: true
excerpt: This is a story about hunting a memory leak. A long story, because I go into much detail…
---

_(This post was translated, the [original version](http://be9.ru/2015/09/12/memory-leak.html)
is in my [Russian blog](http://be9.ru).)_

# Foreword

This is a story about hunting a memory leak. A long story, because I go into much
detail.

Why describe my adventures? Not that I wanted to save all those
tiny code pieces and scripts only. It rather occurred to me that it
was _UNIX way_ which I had pursued. Every step was related to yet another small utility,
or a library, which solves its task well. And finally I succeeded.

Also it was an interesting journey! Sometimes I got in and out of bed with
thoughts about what happens to memory.

I am grateful to my colleagues in [Shuttlerock](https://www.shuttlerock.com/)
company which actually _worked_ (as in "being productive for customers"), while I
was playing memory detective. Grats to the company itself too!

# Introduction

So we found ourselves in an unpleasant situation: our Rails app on
[Heroku](http://heroku.com) platform had a memory leak. One can see it on this graph:

![Memory consumption by web processes](/assets/images/heroku_metrics_memleak.png)

Now, every deploy or restart makes memory line go up fast (see around
3&nbsp;AM, 8&nbsp;AM). That's totally normal, because Ruby is a dynamic language.
Some `require`s are still being fired, something loads up. When everything is
loaded, memory consumption should be quasi-constant. Requests are coming,
objects get created, the line goes up. Garbage collector does its job – it goes down.
The app should be able to stay forever in such a dynamic balance.

But in our case there's an apparent upward trend. A leak! When our Heroku 2X
dyno (we use [puma](http://puma.io) in cluster mode with 2 worker processes)
reaches 1 GB, it becomes slow as hell, unresponsive and needs to be restarted.

# Reproduce the thing locally

Thou shalt not make experiments on the production server. First thing I did
was downloading latest production database dump and running the app in production
environment on my local Linux server which resides in the closet (you should have one
too!). Our app is a SaaS product and has a special middleware which detects customer
sites by domain, therefore I had to introduce small changes to app code enabling me to make
requests like so: `curl http://localhost:3000/…`. Environment variables are
best for this purpose.

{% highlight ruby %}
if ENV['QUERY_SITE_ID']
  def find_site(request)
    Site.find_by_id_cached(request.query_parameters[:site_id])
  end
else
  # Private: Look up for site.
  #
  # Tries to find by domain, subdomain, or use canned site in test environment.
  #
  # Returns Site instance or falsey value.
  def find_site(request)
    host = request.host

    # …
  end
end
{% endhighlight %}

As you can see, if environment variable `QUERY_SITE_ID` is set, site ID gets
taken from query parameter `site_id`:

**curl http://localhost:3000/?site_id=123**

In most cases you'll have to specify `config.force_ssl = false`
in `config/environments/production.rb`, set `DEVISE_SECRET_KEY` variable
(if you use [devise](https://github.com/plataformatec/devise)), maybe something else.
The `curl` command must finally work.

So, server works locally, what's next? You have to supply incoming requests.
The wonderful [siege](https://www.joedog.org/siege-manual/) utility
allows to put load on web servers in different modes and record statistics.

I decided to not hit a single route, but rather collect real URLs used by web clients.
Easy-peasy: run `heroku logs -t | tee log/production.log` for some time,
then extract URLs from the log. I wrote a small utility which parsed log,
collected paths, `site_id` values reported by our middleware and saved everything
into urls.txt like this:

{% highlight text %}
http://localhost:3000/foo
http://localhost:3000/bar
http://localhost:3000/baz
{% endhighlight %}

You can create such a file by hand, or resort to awk, grep, sed.

Let's run siege:

`siege -v -c 15  --log=/tmp/siege.log -f urls.txt`

Here siege will create 15 parallel clients and use urls.txt as its input.

If you do everything right, you'll experience the memory leak. It can be seen
with `top` and `ps` utilities — the number to look for is called RSS (Resident Set Size).
To save myself from running them, though, I've added
[the following code](http://samsaffron.com/archive/2015/03/31/debugging-memory-leaks-in-ruby)
to the app:

{% highlight ruby %}
if ENV['MEMORY_REPORTING']
  Thread.new do
    while true
      pid = Process.pid
      rss = `ps -eo pid,rss | grep #{pid} | awk '{print $2}'`.to_i
      Rails.logger.info "MEMORY[#{pid}]: rss: #{rss}, live objects #{GC.stat[:heap_live_slots]}"

      sleep 5
    end
  end
end
{% endhighlight %}

Accurate records with increasing RSS started to appear in the log. More on
`GC.stat[:heap_live_slots]` later.

After first experiments I swiched puma into single mode, because it leaked too
and single process is easier to deal with.

# Hunting leak in Ruby code

Being certain in leak existence, I started playing the Sherlock Holmes part.

Some words need to be said about how MRI works with memory in general.
Objects are stored in a heap controlled by the interpreter. The heap consists of
separate pages, each being 16 Kb in size and every object using 40 bytes of memory.
When object is created, MRI searches for a free slot: if there's none, extra
page is added to the heap. Now, not every object fits into 40 bytes. If it needs more,
additional memory is allocated (via `malloc`).

Memory is freed automatically when garbage collector (GC) runs. Modern MRIs have quite
effective incremental GC with generations and two phases: minor and major.
Based on heuristic principle "Most objects die young", _minor GC_ tries to find
unneeded objects only among newly created ones. This allows to run _major GC_ less
often, and this guy performs classic [Mark-and-Sweep algorithm](http://c2.com/cgi/wiki?MarkAndSweep)
for **ALL** objects.

It must be noted that intricacies of different generations have nothing to do
with hunting memory leaks. There's one important thing: are all objects freed,
which are created while processing web request, or not? Actually, in web server
context all objects can be divided into three groups:

1. **Statics**. All loaded gems, esp. Rails, and app code. All this gets loaded once
in production environment and doesn't change.
2. **Slow dynamics**. There's a certain amount of long-lived objects, e.g.
cache of prepared SQL statements in ActiveRecord. This cache is per DB connection
and is max. 1000 statements by default. It will grow slowly, and total number of
objects will increase until cache reaches full size (2000 strings * number
of DB connections)
3. **Fast dynamics**. Objects created during request processing and response
generation. When response is ready, all these objects can be freed.

In third case, if any object doesn't get freed, you'll have a leak. Here's an illustration:

{% highlight ruby %}
class MyController < ApplicationController
  FOO = []

  def index
    FOO << "haha"
  end
end
{% endhighlight %}

Constants are not garbage-collected, therefore consecutive `MyController#index` calls
will lead to `FOO` inflation. Heap will grow to accomodate more and more
`"haha"` strings.

If there are no leaks, heap size will oscillate. The minimum size corresponds to
objects from groups 1 and 2 (see above). For example, in our app this size is slightly
above 500,000, while an empty app created with `rails new app` instantiates roughly
300,000 objects. Maximum heap
size depends on how often major GC runs. **But:** after every major GC the number of
objects is always back to the lowest. Leaks will lead to
low boundary increased over time. The `GC.stat[:heap_live_slots]` figure reflects
current heap size.

The most convenient way to explore GC-related things is using
[gc_tracer](https://github.com/ko1/gc_tracer) gem, built by Koichi Sasada,
the Ruby developer team member and the author of incremental GC implementation
in Ruby 2.1 and 2.2. Having added

{% highlight ruby %}
require 'rack/gc_tracer'
config.middleware.use Rack::GCTracerMiddleware, view_page_path: '/gc_tracer', filename: 'log/gc.log'
{% endhighlight %}

to `config/application.rb`, we get `log/gc.log` file which is filled by GC stats and also `getrusage` results
(the latter is useful because one of fields contains RSS figure which we are so interested in!).

Every line of this log has over 50 numbers, but here's some simple UNIX magic to the rescue. The
command I ran along with `puma` and `siege` was:

`tail -f log/gc.log | cut -f 2,4,7,8,9,15,16,19,21,22,25,26,27,28,29,30,36`

which gives this:

{% highlight text %}
1441967588854360        2450    998618  997652  966     2450    0       0       5151    665     581634  1052512 2490888 16777216        0       newobj 277196
1441967588879589        2450    998618  997652  966     2450    0       0       5152    665     598912  1052512 2490888 16777216        0       newobj 277196
1441967590064377        2450    998618  997618  999     2450    0       503688  5152    665     598912  1052512 2103264 16777216        0       newobj 277196
1441967590064684        2450    998618  997808  810     2450    0       0       5152    665     598912  1052512 2107280 16777216        0       newobj 277196
1441967590088032        2450    998618  997808  810     2450    0       0       5153    665     613199  1052512 2107280 16777216        0       newobj 277196
{% endhighlight %}

First number in line is timestamp in milliseconds. Second is the number of pages. Third – heap size (in objects). The 11th
(with values in 581634…613199 range) is the number of "old" objects, i.e. objects which are not inspected during minor GC run. The last number in line is RSS in kilobytes.

Still so many numbers! Let's plot them. We could load this log directly into Excel
(sorry, LibreOffice Calc), but that's not classy. Let's use [gnuplot](http://www.gnuplot.info/)
instead, which is able to plot directly from files.

Unfortunately, gnuplot doesn't support timestamps in milliseconds, so I had to
write a small script for timestamp conversion:

{% highlight bash %}
#!/bin/sh

cat $1 | awk \
'{if($1=="end_sweep"||FNR==1) {if(FNR>1) {$2=substr($2,1,length($2)-6)} else {$2=$2}; print $0}}' > $2
{% endhighlight %}

Along with turning milliseconds into seconds some extra information is discarded here.
gc_tracer generates data on all stages of GC but we are interested only in the final
data (first column contains "end_sweep").

This gnuplot script
{% highlight gnuplot %}
set xdata time
set timefmt '%s'
set format x '%H:%M'
set y2tics on
set y2label 'Kilobytes'
set ylabel 'Objects'
plot 'gc.log.22' using 2:25 with lines title columnhead, \
     '' using 2:36 with lines lc rgb 'blue' title columnhead axes x1y2
{% endhighlight  %}
gets us this picture:

![The number of "old" objects and RSS over time](/assets/images/olds_vs_rss.png)

Red curve (left scale) displays the number of "old" objects. It behaves
like there's no leak. Blue curve (right scale) is RSS, which never stops rising.

The conclusion is: **there are no leaks in Ruby code**. But I wasn't keen enough
to grasp that at the moment and spent more time pursuing a false path.

# False path: messing with heap dumps

Modern versions of MRI are equipped with powerful means for memory analysis.
For instance, you can enable object creation tracing, and for each newly created
object the location where it was instantiated (source file, line) will be saved
and accessible later:

{% highlight ruby %}
require 'objspace'
ObjectSpace.trace_object_allocations_start
{% endhighlight  %}

Now we can dump the heap into a file:

{% highlight ruby %}
File.open("my_dump.txt", "w") do |f|
  ObjectSpace.dump_all(output: f)
end
{% endhighlight %}

Location information will be dumped too, if it's available.
The dump itself is a [JSON Lines](http://jsonlines.org/) file: data for each
object are in JSON format and occupy one line in the file.

There are gems which make use of allocation tracing, like
[memory_profiler](https://github.com/SamSaffron/memory_profiler) and
[derailed](https://github.com/schneems/derailed_benchmarks).
I've decided to investigate what happens with the heap in our app as well.
Having had no luck with memory_profiler, I went for generating and analyzing dumps
myself.

First I wrote my own benchmark like
[derailed](https://github.com/schneems/derailed_benchmarks/blob/master/lib%2Fderailed_benchmarks%2Ftasks.rb#L58)
does, but later switched to generating dumps from live app, using
[rbtrace](https://github.com/tmm1/rbtrace).
If you have `gem 'rbtrace'` in your `Gemfile`, here's a way to generate dump:

{% highlight bash %}
rbtrace -p $PID -e 'Thread.new{GC.start;require "objspace";File.open("heapdump.jsonl", "w"){|f| ObjectSpace.dump_all(output: f) }}'
{% endhighlight %}

Now let's assume we have three dumps (1, 2, 3), generated at different moments in time.
How can we spot a leak? The [following scheme](http://blog.skylight.io/hunting-for-leaks-in-ruby/) has been suggested:
let's take dump 2 and remove all objects which are present in dump 1. Then let's also remove all objects which are **missing**
from dump 3. What's left is objects which were possibly leaked during the time between dump 1 and 2 creation.

I even wrote [my own utility](https://github.com/Shuttlerock/memory-dump-analyzer) for this
differential analysis procedure. It's implemented in… Clojure, because I like Clojure.

But everything I could find was the prepared SQL statements cache mentioned above.
Its contents, SQL strings, weren't leaked, they just lived long enough to appear in dump 3.

Finally I had to admit that there are no leaks in Ruby code and had to look for them
elsewhere.

# Learning jemalloc

I made following hypothesis: if there are no leaks on Ruby side, but memory is somehow still
leaking, there must be leaks in C code. It could be either some gem's native code or MRI itself.
In this case C heap must be growing.

But how to detect leaks in C code? I tried [valgrind](http://valgrind.org/) on Linux and
[leaks](https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man1/leaks.1.html)
on OS X. They haven't got me anything interesting enough. Still looking for something, I've stumbled upon
[jemalloc](http://www.canonware.com/jemalloc/).

Jemalloc is a custom implementation of `malloc`, `free`, and `realloc` that is
trying to be more effective than standard system implementation (not to count
FreeBSD, where jemalloc **is** the system implementation). It uses a bag full of tricks
to achieve this goal. There's an own page system with pages allocated via
`mmap`; the allocator uses independent "arenas" with thread affinity allowing
for less synchronization between threads. Allocated blocks are divided
into three size classes:  _small_ (< 3584 bytes), _large_ (< 4 Mb), and _huge_ –
each class is handled differently. But, most importantly, jemalloc has
statistics and profiling. Finally, MRI 2.2.0 supports jemalloc natively!
The `LD_PRELOAD` hack is not needed anymore (by the way, I couldn't make it work).

I quickly went into installing jemalloc and then MRI with jemalloc enabled. Not without
trouble. Ubuntu and Homebrew-provided jemalloc library is built without profiling.
Also you can't build certain Ruby gems with most recent jemalloc version 4.0.0,
released in August 2015. E.g. `pg` gem doesn't like C99 `<stdbool.h>` included into
`<jemalloc/jemalloc.h>`. But everything is smooth with jemalloc 3.6.0.  

You can follow [this instruction](http://groguelon.fr/post/106221222318/how-to-install-ruby-220-with-jemalloc-support)
to build MRI with rbenv, however at the end I opted for building it myself:

{% highlight bash %}
cd ruby-2.2.3
./configure --with-jemalloc --prefix=/home/od/.rbenv/versions/2.2.3-dbg --disable-install-doc
make
make install
{% endhighlight %}
You'll also need `--with-openssl-dir=/usr/local/opt/openssl` for Homebrew.

This Ruby works as normal. But it reacts to `MALLOC_CONF` environment variable:

{% highlight bash %}
export MALLOC_CONF='narenas:1,stats_print:true'
ruby myscript.rb
{% endhighlight %}

When Ruby is done, you'll get quite detailed memory allocation stats printed
to `stderr`. I used this setting to run `puma` overnight (with `siege`
attacking it). By morning RSS grew up to 2 gigs. Hitting Ctrl-C brought me to the stats
with allocated memory figure being close to same 2 gigs. Eureka!
The C code leak hypothesis has been confirmed!

# Profiling and more detailed stats

Next question was: where in C code are those 2 gigs allocated? Profiling helped
here. The jemalloc profiler stores addresses which called `malloc`, calculates
how much memory has been allocated from each address, and
stores everything to the dump. You can enable it with same `MALLOC_CONF`, specifying
`prof:true` flag. In this case final dump will be generated when the process exits.
Dump is to be analyzed with `pprof` program.

Unfortunately, `pprof` couldn't decode addresses:
{% highlight text %}
Total: 105.3 MB
43.5  41.3%  41.3%     43.5  41.3% 0x00007f384fd2b5f9
23.4  22.2%  63.6%     23.4  22.2% 0x00007f384fd2b773
...
{% endhighlight %}

I had to subtract the start of code segment (this information is printed
when the process exists) from these numbers and use `info symbol` command
in `gdb` (e.g. `info symbol 0x2b5f9`). The address appeared to belong to [objspace_xmalloc](https://github.com/ruby/ruby/blob/v2_2_3/gc.c#L7359) function
(it's declared `static`, maybe that's the reason for non-showing). A more representative
profile, with `puma` being hit by `siege` for 2 hours, showed that
this function allocated **97.9&nbsp;%** of total memory allocated. Now, the leak
has something to do with Ruby indeed!

Having become more certain in my search area, I've decided to investigate statistical
patterns of allocated blocks. Feeling not inclined to parse jemalloc textual
stats output, I wrote my own gem, [jemal](https://github.com/be9/jemal).
The main functionality is in `Jemal.stats` method which returns all
statistics of interest as one big hash.

What was left is adding a small piece of code into the app:

{% highlight ruby %}
if ENV['JEMALLOC_STATS']
  STDERR.puts "JEMALLOC_STATS enabled"

  require 'jemal'

  if Jemal.jemalloc_builtin?
    STDERR.puts "jemalloc found"
    Thread.new do
      first = true

      while true
        sleep 5
        stats = Jemal.stats
        stats[:ts] = Time.now.utc.to_i

        File.open(Rails.root + 'log/jemalloc.log', first ? 'w' : 'a') do |f|
          f.puts stats.to_json
        end

        first = false
      end
    end
  end
end
{% endhighlight %}

…and run `puma` and `siege` overnight again.

By morning `log/jemalloc.log` was big enough and could be analyzed.
[jq](https://stedolan.github.io/jq/) tool proved being extremely helpful. First I
decided to see how memory grows:

{% highlight bash %}
jq '.ts,.allocated' log/jemalloc.log | paste - - > allocated.txt
{% endhighlight %}
Look how UNIX way works here! `jq` parses JSON in every line and outputs
`ts` values and `allocated` values in turn:
{% highlight text %}
ts1
allocated1
ts2
allocated2
...
{% endhighlight %}
Then `paste - -` joins its input line by line, separating two values with TAB:
{% highlight text %}
ts1 allocated1
ts2 allocated2
...
{% endhighlight %}
Such a file can be fed to `gnuplot`:
{% highlight gnuplot %}
set xdata time
set timefmt '%s'
set format x '%H:%M'
plot 'allocated.txt' using 1:($2/1048576) with lines title 'Allocated'
{% endhighlight %}
![Allocated memory over time](/assets/images/allocated.png)

Linear growth! What's with block sizes?

{% highlight bash %}
jq '.ts,.allocated,.arenas[0].small.allocated,.arenas[0].large.allocated' \
   log/jemalloc.log | paste - - - - > allocated.txt
{% endhighlight %}
`jq` can extract data even from deeply embedded structures.
{% highlight gnuplot %}
set xdata time
set timefmt '%s'
set format x '%H:%M'
plot 'allocated.txt' using 1:($2/1048576) with lines title 'Allocated', \
     '' using 1:($3/1048576) with lines title 'Small', \
     '' using 1:($4/1048576) with lines title 'Large'
{% endhighlight %}

![Allocated memory per class over time](/assets/images/allocated_classes.png)

The plot testifies that it's _small_ objects who's being leaked. But that's not all: jemalloc
provides separate stats for different block sizes within one class!
In _small_ class, every allocated block has one of 28 fixed sizes (the requested size is simply rounded up):
8, 16, 32, 48, 64, 80, …, 256, 320, 384, …, 3584. Stats for every size are kept separately.
By staring at the log I noted some anomaly with size 320. Let's plot it as well:
{% highlight bash %}
jq '.ts,.allocated,.arenas[0].small.allocated,.arenas[0].large.allocated,.arenas[0].bins["320"].allocated' \
   log/jemalloc.log | paste - - - - - > allocated.txt
{% endhighlight %}
{% highlight gnuplot %}
set xdata time
set timefmt '%s'
set format x '%H:%M'
plot 'allocated.txt' using 1:($2/1048576) with lines title 'Allocated', \
     '' using 1:($3/1048576) with lines title 'Small', \
     '' using 1:($4/1048576) with lines title 'Large', \
     '' using 1:($5/1048576) with lines title 'Small (257..320)'     
{% endhighlight %}
![Allocated memory per class over time](/assets/images/alloc_320.png)

Wow! Memory is consumed by objects of one size. Everything else is just constant, it's
evident from the fact that lines are parallel. But what's up with size 320?
Apart from total memory allocated, jemalloc calculates 8 more indices, including
the number of allocations (basically, `malloc` calls) and deallocations.
Let's plot them too:
{% highlight bash %}
jq '.ts,.arenas[0].bins["320"].nmalloc,.arenas[0].bins["320"].ndalloc,.arenas[0].bins["256"].nmalloc,.arenas[0].bins["256"].ndalloc' \
   jemalloc.log | paste - - - - - > malloc_dalloc.txt
{% endhighlight %}
{% highlight gnuplot %}
set xdata time
set timefmt '%s'
set format x '%H:%M'
plot 'malloc_dalloc.txt' using 1:2 with lines title '320 nmalloc', \
     '' using 1:3 with lines title '320 ndalloc', \
     '' using 1:4 with lines title '256 nmalloc', \
     '' using 1:5 with lines title '256 ndalloc'
{% endhighlight %}

![The number of allocations and deallocations for 320- и 256-sized blocks over time](/assets/images/malloc_free.png)
Same indices for blocks with neighbor size, 256, are included for comparison.
It's apparent than blue and orange curves blended, which means that the number
of deallocations for 256-sized blocks is approx. equal to the number of allocations
(that's what we call healthy!).
Compare this to size 320, where the number of allocations (magenta curve) runs
away from the number of deallocations (green curve). Which finally proves the
memory leak existence.

# Where's the leak?

We've squeezed everything we could from stats. Now the leak source is to be found.

I had no better idea than adding some debug prints to `gc.c`:

{% highlight c %}
static void *
objspace_xmalloc(rb_objspace_t *objspace, size_t size)
{
    void *mem;
    size_t orig_size = size;

    size = objspace_malloc_prepare(objspace, size);

    TRY_WITH_GC(mem = malloc(size));
    size = objspace_malloc_size(objspace, mem, size);
    objspace_malloc_increase(objspace, mem, size, 0, MEMOP_TYPE_MALLOC);
    mem = objspace_malloc_fixup(objspace, mem, size);

    if (size == 320) {
        fprintf(stderr, "objspace_xmalloc: %zu => %p\n", orig_size, mem);
    }

    return mem;
}

static void
objspace_xfree(rb_objspace_t *objspace, void *ptr, size_t old_size)
{
#if CALC_EXACT_MALLOC_SIZE
    ptr = ((size_t *)ptr) - 1;
    old_size = ((size_t*)ptr)[0];
#endif
    old_size = objspace_malloc_size(objspace, ptr, old_size);

    if (old_size == 320) {
        fprintf(stderr, "objspace_xfree: %p\n", ptr);
    }

    free(ptr);

    objspace_malloc_increase(objspace, ptr, 0, old_size, MEMOP_TYPE_FREE);
}
{% endhighlight %}

Not wanting to drown in information, I started to run `curl` by hand instead of
using `siege`. I was interested in how many blocks get leaked during one request processing.
Yet another middleware was injected to the app:

{% highlight ruby %}
# lib/rack/memory_analyzer_middleware.rb
require 'jemal'

class MemoryAnalyzerMiddleware
  def initialize(app)
    @app = app
  end

  def call(env)
    GC.start
    st1 = Jemal.stats

    puts "-------------------------"

    res = @app.call(env)

    GC.start
    st2 = Jemal.stats

    bin1 = st1[:arenas][0][:bins][320]
    bin2 = st2[:arenas][0][:bins][320]

    puts "MAM BEFORE: #{stats_line(bin1)}"
    puts "MAM AFTER: #{stats_line(bin2)}"

    res
  end

  def stats_line(bin)
    delta = bin[:nmalloc]-bin[:ndalloc]
    "Allocated #{bin[:allocated]} Mallocs #{bin[:nmalloc]} Dallocs #{bin[:ndalloc]} M-D #{delta} Nreqs #{bin[:nrequests]}"
  end
end

# config/initializers/memanalyzer.rb
if ENV['MEMANALYZER']
  require 'rack/memory_analyzer_middleware'
  Rails.application.middleware.unshift MemoryAnalyzerMiddleware
end
{% endhighlight %}
I've used initializer (not `config/application.rb`) here to ensure that the middleware
is exactly at the top of middleware stack.

Having run `curl` a few times, I saw that `M-D` value increases by 80-90
every request. Allocations and deallocations also appeared on `stderr` (and in the
log by virtue of `tee`). Cutting last portion of log between
`-------------------------` and `MAM BEFORE ...`, I ran it through this
simple script:

{% highlight ruby %}
#!/usr/bin/env ruby
require 'pp'

fname = ARGV.first
blocks = {}

File.open(fname, 'r') do |f|
  f.each_line do |line|
    case line
    when /objspace_xmalloc: (\d+) => 0x([0-9a-f]+)/
      blocks[$2] = $1.to_i

    when /objspace_xfree: 0x([0-9a-f]+)/
      blocks.delete $1

    else
      puts line
    end
  end
end

pp blocks
{% endhighlight %}

Now here they are, the possible leak addresses:
{% highlight ruby %}
{"7fed15c08500"=>288,
 "7fed15c0c4c0"=>296,
 "7fed15c0d500"=>312,
 "7fed15c0f440"=>264,
 "7fed15c0f580"=>304,
 # 190 more lines
 "7fed195cee40"=>312,
 "7fed195cf840"=>312,
 "7fed195cfd40"=>312,
 "7fed195cffc0"=>312,
 "7fed195d0b00"=>312,
 "7fed195d0d80"=>312,
 "7fed195e1640"=>312}
{% endhighlight %}
There are quite many blocks with size 312. What does it mean? Nothing!
I wanted to look at the memory contents and used gdb to connect
to live process. So, taking some address and looking what's there:
{% highlight text %}
(gdb) x/312xb 0x7f1138a83340
0x7f1138a83340: 0x34    0xe1    0xde    0x39    0x11    0x7f    0x00    0x00
0x7f1138a83348: 0x97    0xe2    0xde    0x39    0x11    0x7f    0x00    0x00
0x7f1138a83350: 0xb8    0xe3    0xde    0x39    0x11    0x7f    0x00    0x00
0x7f1138a83358: 0xd9    0xe4    0xde    0x39    0x11    0x7f    0x00    0x00
0x7f1138a83360: 0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x7f1138a83368: 0xf4    0xe6    0xde    0x39    0x11    0x7f    0x00    0x00
0x7f1138a83370: 0x98    0xe8    0xde    0x39    0x11    0x7f    0x00    0x00
0x7f1138a83378: 0x3c    0xea    0xde    0x39    0x11    0x7f    0x00    0x00
0x7f1138a83380: 0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x7f1138a83388: 0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x7f1138a83390: 0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x7f1138a83398: 0xd6    0xef    0xde    0x39    0x11    0x7f    0x00    0x00
0x7f1138a833a0: 0xf7    0xf0    0xde    0x39    0x11    0x7f    0x00    0x00
0x7f1138a833a8: 0x2f    0xf2    0xde    0x39    0x11    0x7f    0x00    0x00
0x7f1138a833b0: 0xd9    0xf3    0xde    0x39    0x11    0x7f    0x00    0x00
0x7f1138a833b8: 0x04    0xf5    0xde    0x39    0x11    0x7f    0x00    0x00
{% endhighlight %}

Looks like a bunch of pointers. Intel is little-endian, so most significant
byte is at the end and first line represents `0x7f1139dee134` number
(the `0x7f113..` thing made me believe it's an address).
Helpful? Not much.

Then I wanted to see the backtraces of calls which allocated those blocks.
Lazy googling revealed the following code which works with gcc:

{% highlight c %}
#include <execinfo.h>
void print_trace(FILE *out, const char *file, int line)
{
    const size_t max_depth = 100;
    size_t stack_depth, i;
    void *stack_addrs[max_depth];
    char **stack_strings;

    stack_depth = backtrace(stack_addrs, max_depth);
    stack_strings = backtrace_symbols(stack_addrs, stack_depth);

    fprintf(out, "Call stack from %s:%d:\n", file, line);

    for (i = 1; i < stack_depth; i++) {
        fprintf(out, "    %s\n", stack_strings[i]);
    }
    free(stack_strings); // malloc()ed by backtrace_symbols
    fflush(out);
}
{% endhighlight %}

I've added the call of this function to `objspace_xmalloc`:

{% highlight c %}
if (size == 320) {
    fprintf(stderr, "objspace_xmalloc: %zu => %p\n", orig_size, mem);
    print_trace(stderr, __FILE__, __LINE__);
}
{% endhighlight %}

Then I repeated everything again, ran the possible leak detection script and started to look up
the found addresses in the log, as backtraces were neatly printed near them.

And what I saw was…

{% highlight text %}
objspace_xmalloc: 312 => 0x7fed195e1640
Call stack from gc.c:7394:
    puma 2.11.3 (tcp://0.0.0.0:3000) [shuttlerock](+0x46e7a) [0x7fed31a6fe7a]
    puma 2.11.3 (tcp://0.0.0.0:3000) [shuttlerock](ruby_xmalloc+0x2c) [0x7fed31a70077]
    /home/od/.rbenv/versions/2.2.3-dbg/lib/ruby/gems/2.2.0/extensions/x86_64-linux/2.2.0-static/redcarpet-3.3.0/redcarpet.so(+0x7a00) [0x7fed17df0a00]
{% endhighlight %}

More then a dozen addresses related to `redcarpet.so` showed up.
**Now that's who ate all our memory!!!** It was
[redcarpet](https://github.com/vmg/redcarpet) gem, a Markdown to HTML renderer.

# Verification and fix

It was easy after finding the offender. I ran 10000 renderings in the console – leak confirmed.
Ditched Rails, loaded the gem separately and repeated the same – the leak is still there!

The only place where redcarpet's native code allocated memory through Ruby interface
is in `rb_redcarpet_rbase_alloc` function which is basically a C constructor for
`Redcarpet::Render::Base`. The allocated memory wasn't freed
during garbage collection. Quick googling revealed
[an example of how to write such a constructor
correctly](http://tenderlovemaking.com/2010/12/11/writing-ruby-c-extensions-part-2.html) in tenderlove's blog.
And the fix was [simple](https://github.com/vmg/redcarpet/pull/516/files).

Bingo!

# Conclusion

1. Probably, it shouldn't have taken 2 weeks. I'd spend much less now.
2. False paths are getting you out of the way, and you have to get back. Aside
from dumps I've also spent some time trying to configure garbage collector with
environment variables. The only thing I could take with me was
`RUBY_GC_HEAP_INIT_SLOTS=1000000`. With this setting our app fits into the heap
and doesn't need more slots.
3. It seems that you can debug anything in our times. The number of helpful tools
and libraries is incredible. If you don't succeed, just try more.

# P.S.

Redcarpet guys still did nothing about my [pull request](https://github.com/vmg/redcarpet/pull/516)
as of now (Sep 21, 2015), but we've deployed the fix to production long ago.
Compare this leak-y picture (same as in the beginning of this post)

![Memory consumption with a leak](/assets/images/heroku_metrics_memleak.png)

with normal memory consumption graph:

![Memory consumption after leak fix](/assets/images/heroku_metrics_noleak.png)

Here's no trend at all. As it should be.
