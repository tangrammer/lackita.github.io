---
layout: post
title: "Types of Tests"
date: 2014-08-18 12:01:09 -0400
comments: true
categories: tdd testing
---

Testing has become really overloaded, and I think this creates barriers to writing or changing tests. There was all that activity earlier this year following [DHH's indictment of TDD](http://david.heinemeierhansson.com/2014/tdd-is-dead-long-live-testing.html), including a [really interesting conversation between DHH, Kent Beck and martin Fowler](http://martinfowler.com/articles/is-tdd-dead/).

What inspired this post, though, was [Beck's snarky response to DHH](https://www.facebook.com/notes/kent-beck/rip-tdd/750840194948847) before the conversation. In it he lists eight separate bullet points on what he gets out of TDD, which is asking a lot of a few lines of code.  A major theme in software design is that each piece of code should only try to do one thing, we even do that in testing by encouraging one assertion per test, but somehow the idea was lost when we started conflating objectives.

I suggest we have clear separation on which purpose each test is trying to serve.  This isn't a radically new idea, acceptance and unit tests were distinct entities in [Extreme Programming: Explained (aff)](http://www.amazon.com/gp/product/0321278658/ref=as_li_qf_sp_asin_il_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=0321278658&linkCode=as2&tag=journeydevelo-20&linkId=BQMXJ3U3GQFCHOVC), but I think we should take this separation even further.

Acceptance Tests
----------------
As I mentioned before, this is the piece that is already treated somewhat separately.  Unfortunately, for a lot of shops, these can be indistinguishable from other kinds of tests.  When I work in codebases like this, I become afraid to delete superfluous tests on the off chance they're enforcing some important piece of business logic.

I've also found that this kind of test works best when passing through as high a level as possible.  If this happens at a lower level I become resistent to refactorings that might change that portion dramatically.

Documentation Tests
-------------------
Here's where I'm going to depart from the norm.  I'm completely behind having documentation be executable code, enforcing that it never gets out of date, but making them subordinate to other tests is just awkward.

When I'm trying to understand a new system, the last thing I want to do is pore over a bunch of low level tests to get an idea of what's going on.  We may be able to make our tests read like natural language, but that does nothing for how they're organized.

If tests serve as documentation, they should stand apart.  They should start with high level concepts and defer details until later.  Keeping the information up-to-date automatically is important, but not at the cost of organizing it in a coherent way.

Comprehension Tests
-------------------
Another place I often write tests is when I'm trying to understand a piece of code. I tend to think of this kind of test as completely expendable.  It's ok to leave these tests around, think of them as your notes on the system.  As soon as they get in the way, though, get rid of them, the system has changed too much for the notes to remain accurate.

Tools
-----
Some tools can be used to enforce this separation, even if that wasn't their original intent.
- [Cucumber](http://cukes.info) is a good option for acceptance tests, pushing down the implementation and emphasizing high level concerns.
- Something like [rspec](http://rspec.info) could be useful for documentation tests, as it is designed to be more expressive.
- Any tool will work for Comprehension tests, they're all about getting understanding quickly, so whatever tool is fastest.
