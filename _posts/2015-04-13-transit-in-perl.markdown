---
layout: post
title: "Transit in Perl"
date: 2015-04-13 07:16:46 -0400
comments: true
categories: transit,cognitect
---

I've just released the first version of [Data::Transit](http://search.cpan.org/~lackita/Data-Transit-0.8.04/lib/Data/Transit.pm), an implementation of the [transit format](https://github.com/cognitect/transit-format) in Perl. This is an early release, so be warned that there are still a lot of sharp edges.

Motivation
----------
Despite decades of web frameworks and cool new languages, there's still a large portion of the web built in Perl. It's one of the original server side scripting languages, and so we'll be dealing with it for many years to come. One of Transit's goals is to allow disparate languages to communicate with richer data; by building a Perl version, it becomes easier to build new subsystems in other languages.

Design Trade-offs
----------------
Because I find that Perl, Ruby and even Python look extremely similar when you get past cosmetic differences, I studied those implementations heavily. I was able to avoid some pitfalls, but there was sufficient difference in Perl hashes to make it difficult to solve one of the problems in the same way.

Similar to many other languages, Perl uses 1 and 0 to represent true and false instead of having actual booleans. In the Python implementation of the spec, special true and false types were introduced to avoid key collisions in hashes, but Perl can't use that approach. Only strings are allowed to exist as keys, which means my design choices are limited.

There are basically two options: accepting the possibility of key collisions or some sort of string-based encoding. While it's tempting to do the latter for completeness, that sacrifices the feel of dealing with types native to the language. Thinking about the common usage path, it's unlikely both true and 1 will be passed as keys to the same hash.  I generally prefer not to complicate the interface to a library to serve the needs of the last 5%, so I ended up choosing the former.

The other complication unique to Perl is that many libraries usually part of the base language (Date, UUID, etc.) are provided as CPAN libraries in Perl. If these types are part of the base package, they would have to be included as dependencies. This would explode the dependencies pulled in by the package for types that may not ever be used. As with hash keys, I chose the needs of the common path over a perfect implementation and left those portions out of the base implementation.

Future Work
-----------
As mentioned in the beginning of the article, this is a pretty early release. More extensive testing is required before it can be taken out of alpha. Most notably, Cognitect has a list of Seattle user group meetings they use as an unofficial benchmark for the format, which needs to be run with some profiling to ensure reasonable performance.

In the design trade-offs, I discussed choosing not to pull in a bunch of CPAN modules that may never be used. Many people may want to use these packages, though, so there should be a way to supply a set of extension distributions that can be easily included if desired. This will be largely driven by feedback, though, as I'm not a fan of writing a bunch of code nobody will use.
