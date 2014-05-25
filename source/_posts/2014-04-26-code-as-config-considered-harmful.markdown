---
layout: post
title: "Code as config considered harmful"
date: 2014-04-26 12:34
comments: true
published: false
categories: 
---

Configuration as code a (or the wider scoped infrastructure as code) is a devops mantra.

I'm entirely behind this - repeated / duplicated configuration is an evil thing
and taking steps to be able to keep your configs in revision control and template
repeated sections are extremely helpful.

So what do I mean by "code as config"?

DSLs! (Domain Specific Languages)

The idea of a DSL is to write something a more 'natural language' than as normal code.

This is also a great idea in a bunch of cases.

However DSLs can be (and are) overused when it's not appropriate.

Picking on one example to cover in depth (I'll illustrate plenty more at the end), I'm going
to pick on Puppetfile.

https://github.com/rodjek/librarian-puppet

https://github.com/adrienthebo/r10k

Puppetfile is an extremely simple DSL, it could easily just be a YAML or JSON file.

In fact, I think it _should_ be a YAML or JSON file instead.

Anything which wants to read Puppetfile either needs to be written in ruby, and re-implement the DSL
(and as r10k shows, this is pretty trivial https://github.com/adrienthebo/r10k/blob/master/lib/r10k/puppetfile.rb#L90)
, or needs to re-implement a subset of ruby enough to read Puppetfile.

This isn't an insurmountable problem, as noted the subset of ruby used is very very simple.

The larger issue comes if you wanted to _write_ that config out again.

I'd like to be able to make a Jenkins job which upgrades any modules I'm using for newer versions, and then runs the
tests in my code.

