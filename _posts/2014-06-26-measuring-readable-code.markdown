---
layout: post
title: "Measuring Readable Code"
date: 2014-06-26 07:29:34 -0400
comments: true
categories: [readability, craftsmanship]
---

Ever wonder how readable people find your code?  Do you wish there was a more
objective way to measure this?

Now there is.  Introducing 
[Readable Code](http://readability-rating.colinwilliams.name), a site targeted
at measuring the time it takes to understand a piece of code.

How It Works
------------

* Visit the site
* Read the sample presented
* When you understand what the sample is doing, click "I Understand This"

The time it took to understand the sample will be recorded, and you will be
taken to a page averaging all of these times.

Call For Help
-------------

As you may be able to tell, the site is currently more of a proof of concept
than a valid experiment.  Try it out and let me know what you think in the
comments of this post.

Also, I'd like to add some sort of qualitative measure, something to gauge the
*level* of understanding, rather than just the time, let me know in the
comments.  I've had a couple of ideas, but most of them add more complexity to
the design than I'm happy with.

Motivation
----------

There are a lot of people in the software development community which are loudly
asserting contradicting advice on what makes code readable.  I'm not sure what
the answer is in a lot of these cases, and am tired of these people arguing
without objective data.  My hope is that, by providing a place for these ideas
to be measured, we can finally get an objective answer.