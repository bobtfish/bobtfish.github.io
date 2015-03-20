---
layout: post
title: "The starwars methodology"
date: 2013-10-05 00:25
comments: true
categories:
---

This seemingly isn't a well known term, and today I'm irritated by people
thinking I perform 'magic' by using grep, so here's a rant about what I call
the "*Star Wars*" (Tm) methodology of problem solving:

To put it quite simply:

> Use the Source Luke

Oftentimes grepping through the source code of the thing that's giving you
trouble (even if it's written in a language you don't speak) will turn up
gold.

Given a basic model of OO languages, it's easily possible to infer what's
going on (or what may be going on) most of the time given basic observations
and a few print statements, or even a file and line number.

Getting a full backtrace out of [most](https://metacpan.org/module/JJORE/App-Stacktrace-0.09/bin/perl-stacktrace) [dynamic](http://isotope11.com/blog/getting-a-ruby-backtrace-from-gnu-debugger) [languages](https://wiki.python.org/moin/DebuggingWithGdb) is real easy too, even in extreme cases..

Even if you don't know for sure, some source diving will at least
help give you more concrete ideas about how the code is built, which
will allow you to frame the problem better when you do peresent it
to someone with the relevant language and/or domain knowledge.

Even if your assumptions were totally wrong - at at least look more
initiative (and tried to go deeper) than your average Joe, so
the expert is more likely to want to help you, as you're already
demonstrated a provable desire to be taught to fish, rather than
just be thrown them.

&lt;/rant&gt;

