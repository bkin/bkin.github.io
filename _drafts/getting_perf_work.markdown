---
layout: post
title:  "Getting even more started with perf"
date:   2015-10-20 09:32:03
categories: perf perf_events profiling tutorial
---

This is a gist:
{% gist kirixik/cb4a367c904336536cdf %}

The [last blog]({% post_url 2015-10-20-getting_started_with_perf %}) did praise [perf] a lot
since it allows you to reason about the performance of your application in
detail. But once you leave the complexity of "foo() calls bar()" microbenchmarks
and target a real application, you may hit a roadblock, namely incomplete and/or
misleading call stacks that will make perf seem less useful.

[perf]: https://perf.wiki.kernel.org/

Don't give up right now! This is not really perf's fault, and in this post, I'm
going to show you how to cope.

To set the stage, we are going to profile this tiny application:

{% gist bkin/cb09787148ae7caf1ed9 %}

Yes, we are in Microbenchistan again, but for a reason.

This computes the length of collatz sequences in a convoluted way, pushes the
result into a vector and tells you what the longest sequence is. The
computation is done with three functions that call one another which should
give use nice and deep callstacks.

Since we are interested in measuring the application in the state it will be
deployed, we are doing an optimized release build without debug information:

{% highlight bash %}
$ g++ -s -O3 -std=c++11 -o collatz collatz.cpp
{% endhighlight %}

{% highlight bash %}
$ ./collatz 
Longest: 910107 takes 476 steps
$ perf stat ./collatz
Longest: 910107 takes 476 steps
failed to read counter stalled-cycles-backend

 Performance counter stats for './collatz':

        542.985311      task-clock (msec)         #    0.999 CPUs utilized          
                 4      context-switches          #    0.007 K/sec                  
                 0      cpu-migrations            #    0.000 K/sec                  
             1,599      page-faults               #    0.003 M/sec                  
     1,440,199,629      cycles                    #    2.652 GHz                    
       589,314,998      stalled-cycles-frontend   #   40.92% frontend cycles idle   
   <not supported>      stalled-cycles-backend   
     1,399,979,024      instructions              #    0.97  insns per cycle        
                                                  #    0.42  stalled cycles per insn
       394,622,993      branches                  #  726.766 M/sec                  
        44,313,508      branch-misses             #   11.23% of all branches        

       0.543290826 seconds time elapsed
{% endhighlight %}

So far so good, now let's try perf report to see where the time is spent:

{% highlight bash %}
$ perf record ./collatz
Longest: 910107 takes 476 steps
[ perf record: Woken up 1 times to write data ]
/proc/kcore requires CAP_SYS_RAWIO capability to access.
[ perf record: Captured and wrote 0.097 MB perf.data (2224 samples) ]
$ perf report|head -n20
/proc/kcore requires CAP_SYS_RAWIO capability to access.
no symbols found in /home/bki/spielplatz/perf_tutorial/getting_even_more_started/collatz, maybe install a debug package?
# To display the perf.data header info, please use --header/--header-only options.
#
#
# Total Lost Samples: 0
#
# Samples: 2K of event 'cycles'
# Event count (approx.): 1444465780
#
# Overhead  Command  Shared Object      Symbol                        
# ........  .......  .................  ..............................
#
    15.47%  collatz  collatz            [.] 0x0000000000000fd5        
     6.62%  collatz  collatz            [.] 0x0000000000000f19        
     6.51%  collatz  collatz            [.] 0x0000000000000fd0        
     6.03%  collatz  collatz            [.] 0x0000000000000fcc        
     5.53%  collatz  collatz            [.] 0x0000000000000f02        
     4.90%  collatz  collatz            [.] 0x0000000000000efd        
     4.82%  collatz  collatz            [.] 0x0000000000000fde        
     3.11%  collatz  collatz            [.] 0x0000000000000fe4        
     3.08%  collatz  collatz            [.] 0x0000000000000ef0        
{% endhighlight %}

No readable symbol names, but perf explains to us that we need debug info.

Adding debug info
---

This is very useful for understanding the profiling results. It also won't mess with the performance of your application. I default to build release versions with debug info like so:
{% highlight bash %}
$ g++ -g -O3 -std=c++11 -o collatz collatz.cpp
$ perf stat -n ./collatz
Longest: 910107 takes 476 steps

 Performance counter stats for './collatz':

       0.545175655 seconds time elapsed
{% endhighlight %}

There, '-g' and no effect on total runtime. (Btw: 'perf stat -n' does not use
any performance counters and serves as a glorified 'time' utility in this
case.)

Let's profile with call graph information this time:
{% highlight bash %}
$ perf record -g ./collatz
Longest: 910107 takes 476 steps
[ perf record: Woken up 1 times to write data ]
/proc/kcore requires CAP_SYS_RAWIO capability to access.
[ perf record: Captured and wrote 0.142 MB perf.data (2092 samples) ]
{% endhighlight %}

Ok, 0.142 MB of profiling goodness which looks like this:

{% highlight bash %}
$ perf report|head -n40
/proc/kcore requires CAP_SYS_RAWIO capability to access.
# To display the perf.data header info, please use --header/--header-only options.
#
#
# Total Lost Samples: 0
#
# Samples: 2K of event 'cycles'
# Event count (approx.): 1459896714
#
# Overhead  Command  Shared Object      Symbol                                                                     
# ........  .......  .................  ...........................................................................
#
    96.39%  collatz  collatz            [.] collatz                                                                
            |
            ---collatz
               |          
               |--0.11%-- 0
               |          
               |--0.05%-- 0x9f0000003500
               |          
               |--0.05%-- 0x210000005f000000
               |          
               |--0.05%-- 0x3c0000006d0000
               |          
               |--0.05%-- 0x470000001300
               |          
               |--0.05%-- 0x32000000320000
               |          
               |--0.05%-- 0x8b0000005a000000
               |          
               |--0.05%-- 0x7400000043
               |          
               |--0.05%-- 0x7300000073
                --99.46%-- [...]

     2.91%  collatz  collatz            [.] main                                                                   
            |
            ---main
               |          
               |--1.80%-- 0xffffffff80e90000
               |          
{% endhighlight %}

Again, not very helpful. For each entry in the output, we should see a chain of
callers all the way down to main. Instead, we see a bunch of bogus memory
adresses instead of proper symbols.

This is due to the mode we have built the call chains. 'perf record -g' is a
synonym for 'perf record --call-graph fp' which uses frame pointers to find the
callers, but unfortunately, current compilers will default to omit them from
optimized code even when you do not explicitly compile with
'-fomit-frame-pointers'.

There are two ways around this. One way is to tell perf to use something else
for finding the callers, for example the dwarf debugging information:

{% highlight bash %}
$ perf record --call-graph dwarf ./collatz
Longest: 910107 takes 476 steps
[ perf record: Woken up 69 times to write data ]
/proc/kcore requires CAP_SYS_RAWIO capability to access.
[ perf record: Captured and wrote 17.207 MB perf.data (2142 samples) ]
{% endhighlight %}

That is a lot of profiling data for one second of sampling and the main issue I
have with this method. At least, we are getting more reasonable results this
way:

{% highlight bash %}
    95.90%  collatz  collatz            [.] collatz                                                                
            |
            ---collatz
               |          
               |--97.37%-- collatz
               |          |          
               |          |--96.40%-- collatz
               |          |          |          
               |          |          |--96.76%-- collatz
               |          |          |          |          
               |          |          |          |--96.10%-- collatz
               |          |          |          |          |          
               |          |          |          |          |--96.85%-- collatz
               |          |          |          |          |          |          
...
{% endhighlight %}

Nice, endless callstacks. We do not see calls to even/odd due to inlining, but
we can see collatz calling itself recursively (Btw: You can check for yourself
about the inlining either with 'perf annotate -s collatz' or with 'objdump -d
collatz').

If you happen to have a Haswell chip and a recent version of perf, you could
also use '--call-graph lbr' to make use of the 'Last Branch Record' feature
that these chips have. There is [kernel support][lwn_lbr] for this, but since
my machine is older, I can not say anything about how it would look like in our
little example here.

[lwn_lbr]: https://lwn.net/Articles/617097/

Another way is to recompile your application to preserve frame pointers:

{% highlight bash %}
$ g++ -g -O3 -fno-omit-frame-pointer -std=c++11 -o collatz collatz.cpp
$ perf stat -n ./collatz
Longest: 910107 takes 476 steps

 Performance counter stats for './collatz':

       0.532246306 seconds time elapsed

$ perf record -g ./collatz
Longest: 910107 takes 476 steps
[ perf record: Woken up 2 times to write data ]
/proc/kcore requires CAP_SYS_RAWIO capability to access.
[ perf record: Captured and wrote 0.488 MB perf.data (2133 samples) ]
{% endhighlight %}

Much less data written during profiling, no performance impact whatsoever[^1], and the same results:

[^1]: Well, technically there is some overhead since each function now has to save the frame pointer register EBP, set it up and restore it upon exit. It also takes away one of the registers. In practice, this does not add much to all the other things that collatz() or any typical function does and therefore does not take much away from performance.

{% highlight bash %}
    96.05%  collatz  collatz            [.] collatz                                                                
            |
            ---collatz
               |          
               |--96.26%-- collatz
               |          |          
               |          |--96.64%-- collatz
               |          |          |          
               |          |          |--96.15%-- collatz
               |          |          |          |          
               |          |          |          |--96.30%-- collatz
               |          |          |          |          |          
               |          |          |          |          |--96.34%-- collatz
               |          |          |          |          |          |          
...
{% endhighlight %}
