---
layout: post
title: "Simple Collision Detection"
date: 2015-03-07 22:22:44 -0500
comments: true
categories: java
---

I started looking at collision detection for a [side project I've been tinkering with lately](https://github.com/lackita/ZombieSimulator).  The basic problem I needed to solve was, given a bunch of dots of uniform size on a plane, how do you quickly determine if any of them are touching or overlapping.

Obviously there's the brute force solution of comparing every dot with every other dot, but a $O(n^2)$ solution will fall over when the number of dots start scaling up.  It really feels like a hash table should be possible here, but that kind of lookup only works well on discrete values, not the continuous ranges we're dealing with here.

If it were possible to convert into some small set of discrete values, we could use a hash table.  To do this, imagine dividing the plane up into equally sized squares.  Within the hash table, we could maintain a set of all the dots that are at least partially covered by the corresponding square.

Now, instead of having to compare against every dot on the plane, the problem has been reduced to just those candidates within one of these squares.  If we contrive the squares to have sides matching the diameter of the dots, then in the worst case the dot can exist in four squares.

Now, obviously there can be any number of dots within these squares, but I was trying to prevent dots from overlapping, which means there can be at most four dots in each square.  So there is a constant bound on the number of dots to compare against, and so the problem has been reduced to $O(n)$.
