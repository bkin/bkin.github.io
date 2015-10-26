---
layout: post
title:  "Practical Performance Engineering"
date:   2015-10-20 09:32:03
categories: perf perf_events profiling tutorial premature_optimization
---

Welcome to my little series on profiling applications in Linux with the superb
tracing and profiling tool `perf`. There are a number of tutorials and articles
about `perf` already, so why write another one? I have spent considerable time
with improving performance of programs of different sizes and application
domains, have tried a few profilers along the way and found `perf` to be the
best of those. I feel indepted to the good people who have invented it and
adding some more documentation is may way of paying it forward.

Also, while `perf`'s documentation is pretty good and the ever increasing
number of tutorials cover a lot of ground, the environment that I am personally
working in is a little underprepresented:
  * C++ application too large to keep in a single head completely
  * Large, constantly evolving feature set
  * Maintenance presumably required for a decades
  * Performance not the primary concern

I could imagine that this is fairly typical for an enterprise application and
something that quite a few developers find themselves to be working with. Maybe
I'm wrong and everybody else is working on superscalable web applications these
days, in which case I'm just scratiching my onw itch here. Whatever.

“Rules of Optimization:
Rule 1: Don't do it.
Rule 2 (for experts only): Don't do it yet.”

― Michael A. Jackson 

This cute quote by british computer scientist [Micheal A.
Jackson][https://en.wikipedia.org/wiki/Michael_A._Jackson] phrases an important
objection to program optimization into more light-hearted terms than the
Knuth's "premature optimization is the root of all evil".
