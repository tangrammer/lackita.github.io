---
layout: post
title: "Refactoring to Something More Expressive"
date: 2014-09-27 21:35:32 -0400
comments: true
categories: clojure sicp
---

Another fun tidbit from going through SICP.

Exercise 2.2
------------
Consider the problem of representing line segments in a plane. Each segment is represented as a pair of points: a starting point and an ending point. Define a constructor make-segment and selectors start-segment and end-segment that define the representation of segments in terms of points. Furthermore, a point can be represented as a pair of numbers: the x coordinate and the y coordinate. Accordingly, specify a constructor make-point and selectors x-point and y-point that define this representation. Finally, using your selectors and constructors, define a procedure midpoint-segment that takes a line segment as argument and returns its midpoint (the point whose coordinates are the average of the coordinates of the endpoints).

First Approach
==============
My first approach was to use the constructors they described, modernizing the data structures slightly to make it easier to understand.

``` clojure
(defn make-point [x y]
  {:x x
   :y y})

(defn make-segment [start end]
  {:start start
   :end end})

(defn midpoint [segment]
  {:x (/ (+ (-> segment :start :x)
            (-> segment :end :x))
         2)
   :y (/ (+ (-> segment :start :y)
            (-> segment :end :y))
         2)})
```

With the data structures make-point and make-segment aren't incredibly useful.  I won't reference them again, but I ended up just defining the data structures directly and deleting the constructors.

The first duplication I eliminated was between the two averages, as the only thing that changed was the axis.

``` clojure
(defn midpoint [segment]
  (into {} (map (fn [axis]
                  {axis (/ (+ (-> segment :start axis)
                              (-> segment :end axis))
                           2)})
                [:x :y])))
```

Better, but I'm not sure things got easier to read, and there's still that duplicated structure between extracting the start and end points.

Something that will make this easier to understand is naming the averaging concept with a function.

``` clojure
(defn average [& args]
  (/ (apply + args)
     (count args)))

(defn midpoint [segment]
  (into {} (map (fn [axis]
                  {axis (average (-> segment :start axis)
				                 (-> segment :end axis))})
	            [:x :y])))
```

That's a little easier to understand, but still working on a pretty low level.  One thing about clojure is there are so few data types that often there's a higher level concept provided by the language.

``` clojure
(defn midpoint [segment]
  (merge-with average (segment :start)
                      (segment :end)))
```

Summary
=======
I was amazed at how clear and expressive the final representation was.  The intermediate refactorings helped me see what I was trying to do from a higher level, and this ultimately gave me the insight that I was actually merging the points together with an average.
