---
layout: post
title:  "Practical Performance Engineering"
date:   2015-10-20 09:32:03
categories: perf perf_events profiling tutorial premature_optimization
---

Welcome!
===

Welcome to my little series on profiling applications in Linux with the superb
tracing and profiling tool `perf`!

There are a number of tutorials and articles about `perf` already, so why write
another one? I have spent considerable time with improving performance of
programs of different sizes and application domains, have tried a few profilers
along the way and found `perf` to be the one I like best. I feel indebted to
the good people who have invented it, and adding some more documentation is may
way of paying it forward.

Also, while `perf`'s documentation is pretty good and the ever increasing
number of tutorials cover a lot of ground, the environment that I am personally
working in is a little underrepresented:
  * C++ application too large to keep in my head completely
  * Many developers with different programming styles
  * Large, constantly evolving feature set
  * Maintenance presumably required for years
  * Performance not the primary concern
This poses some unique challenges to development in general and to performance
engineering in particular.

I could imagine that this is a fairly typical setting for developing an
enterprise application and something that quite a few developers find
themselves to be working with. Maybe I'm wrong and everybody else is working on
superscalable web applications these days, in which case I'm just scratching
my own itch here. Whatever.


Why optimize?
---

“Rules of Optimization:
Rule 1: Don't do it.
Rule 2 (for experts only): Don't do it yet.”

― Michael A. Jackson 

This quote by British computer scientist [Micheal A.
Jackson][https://en.wikipedia.org/wiki/Michael_A._Jackson] phrases an important
objection to program optimization into more light-hearted terms than Knuth's
"premature optimization is the root of all evil".

Both warn about optimizations having a cost that is often inappropriate for the
intended effect.

One issue is that performance itself is rarely considered a feature in itself.

Instead, when a user is asking for 'X' to be implemented, there seems to be a
silent agreement that 'X' should not be annoyingly slow to perform. The proof
is in the pudding and only after the implementation is finished users and
developers can find if they reached agreement on what 'annoyingly' is supposed
to mean. If users are not satisfied, the absence of performance will be felt
too painfully and you will get an infuriated bug report, rather than a new
request for 'X but faster, please'.

Another issue is that the process of changing some implementation of 'X' to one
that can do the same while using less resources has some upfront costs (such as
additional development time) and follow-up costs (for maintenance of a more
complex piece of code) that all parties would like to avoid.

Adding to that, [Amdahl's Law][https://en.wikipedia.org/wiki/Amdahl's_law]
applies and limits how effective an optimization can be. If the user only
spends 1% of her time waiting for 'X' to finish, this is the largest
improvement that any optimization will be able to achieve.

In order to make a decision on whether to optimize a piece of code, you
therefore should have some idea on how much resources you could reclaim this
way. How many seconds of waiting time can you spare your users? How much energy
can you avoid to consume? How much memory or bandwidth can you free for the
benefit of other applications? And is this worth the cost for implementation
and maintenance?

For an enterprise application, you could probably guesstimate that a person is
working ~6.5 million seconds a year and therefore each second of that work is
roughly worth ~0.01 €/$ to the employer of that person. How many users do you
have? How many seconds of their precious time does your optimization reclaim
per year? Again, is it worth your development cost?

As some guideline on what is reasonable, I try to relate changed lines of code
to achieved result in the following way:
 * If the improvement is N% for N<=10, then change only about 3 * N lines of
   non convoluted code, including comments.

This is arbitrary, but it helps me letting go of optimizations that seem
interesting to me but are too expensive to implement and test. It also
encourages me to come up with small and simple solutions for small
improvements.

For larger improvements, I have no such rule.

All of this is a little disheartening when you as a developer are enthusiastic
about performance and you actually like to optimize code. I am totally with you
here. Optimization in itself can be a lot of fun and the fact that your
achievements are measurable is very satisfying.

My understanding of the whole matter is just that you should take the time to
check where an optimization will have the most leverage, if implementation
effort is reasonable, and only then go for it. And this is where `perf` will
help you time and time again!

If your optimization is going to have little effect, just let it go - slow code
will come your way another day.

That concludes my initial disclaimer. The next few parts will give you a small
tour on `perf`'s features and some guidelines on setting it up properly on your
development machine.
