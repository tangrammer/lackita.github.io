---
layout: post
title:  "TDD and Mathematics"
date:   2014-03-14 00:00:00
categories: [tdd, sicp, clojure]
comments: true
---

I've started going through
[SICP](http://mitpress.mit.edu/sicp/full-text/book/book.html) to try to learn
Clojure in a little more depth.  I'm quickly realizing, though, that since some
of the exercises are on the theme of "implement this mathematical equation",
it's difficult to do so using TDD.

Exercise 1.8
------------

Newton's method for cube roots is based on the fact that if y is an
approximation to the cube root of x, then a better approximation is given by the
value 

$$
\frac{\frac{x}{y^2} + 2y}{3}
$$

Use this formula to implement a cube-root procedure analogous to the square-root
procedure.

Approach
========

good-enough?
------------

Because this function provides approximations, not exact values, we first need
to build a function to tell us a value is good enough.  This will certainly be
needed in the tests, as our function isn't going to return exactly 2 when 8 is
given.  What's interesting is we'll also need it in the main code, to know if
we're close enough to stop.

Exact values will be easier to build, so we'll start with the usual base case: 0

``` clojure
(deftest good-enough?-test
  (good-enough? 0 0))

(defn good-enough? [guess n])
```

That's pretty braindead, so let's add an assertion

``` clojure
(deftest good-enough?-test
  (is (good-enough? 0 0)))

(defn good-enough? [guess n]
  true)
```

Now let's get rid of that hard-coded true:

``` clojure
(deftest good-enough?-test
  (is (good-enough? 0 0))
  (is (not (good-enough? 1 0))))

(defn good-enough? [guess n]
  (= guess n))
```

We're trying to find the cube root, not equality:

``` clojure
(deftest good-enough?-test
  (is (good-enough? 0 0))
  (is (not (good-enough? 1 0)))
  (is (good-enough? 2 8)))

(defn good-enough? [guess n]
  (= (math/expt guess 3) n))
```

And it usually won't be an exact match:

``` clojure
(deftest good-enough?-test
  (is (good-enough? 0 0))
  (is (not (good-enough? 1 0)))
  (is (good-enough? 2 8))
  (is (good-enough? 2.000001 8)))

(defn good-enough? [guess n]
  (< (- (math/expt guess 3) n) 0.0001))
```

But the tolerance is faulty, anything less than the target will pass:

``` clojure
(deftest good-enough?-test
  (is (good-enough? 0 0))
  (is (not (good-enough? 1 0)))
  (is (good-enough? 2 8))
  (is (good-enough? 2.000001 8))
  (is (not (good-enough? 0 8))))

(defn good-enough? [guess n]
  (< (math/abs (- (math/expt guess 3) n)) 0.0001))
```

next-guess
----------

Here's where we build the core of the algorithm, the logic to generate a new
guess.  The idea here is to build it up incrementally by making as much of the
expression as possible go to 0.

We'll start by making $\frac{x}{y^2}$ go to 0.  Ideally, we would also like to
make $2y$ go to 0, but that would cause a division by 0 error.

``` clojure
(deftest next-guess-test
  (is (= (next-guess 1 0) 2/3)))

(defn next-guess [guess n]
  2/3)
```

Now another value to triangulate the functionality:

``` clojure
(deftest next-guess-test
  (is (= (next-guess 1 0) 2/3))
  (is (= (next-guess 2 0) 4/3)))

(defn next-guess [guess n]
  (/ (* 2 guess) 3))
```

Let's start working on the next term.  Notice that by keeping the denominator of
this term to 1 we get to drive the numerator separately:

``` clojure
(deftest next-guess-test
  (is (= (next-guess 1 0) 2/3))
  (is (= (next-guess 2 0) 4/3))
  (is (= (next-guess 1 1) 1)))

(defn next-guess [guess n]
  (/ (+ n (* 2 guess))
	 3))
```

Finally we can finish off the equation:

``` clojure
(deftest next-guess-test
  (is (= (next-guess 1 0) 2/3))
  (is (= (next-guess 2 0) 4/3))
  (is (= (next-guess 1 1) 1))
  (is (= (next-guess 2 1) 17/12)))

(defn next-guess [guess n]
  (/ (+ (/ n (math/expt guess 2))
		(* 2 guess))
	 3))
```

cube-root
---------

Now that we have a reliable way to tell if our answer's good enough, and a way
to determine the next guess, we can start driving the main function:

``` clojure
(deftest cube-root-test
  (is (good-enough? (cube-root 1) 1)))

(defn cube-root [n]
  1)
```

We'll need a way to track our current guess, so this creates the need for an
extra argument:

``` clojure
(deftest cube-root-test
  (is (good-enough? (cube-root 1) 1))
  (is (good-enough? (cube-root 1 1) 1)))

(defn cube-root
  ([n] 1)
  ([guess n] 1))
```

Now we can refactor the duplication out of the two forms:

``` clojure
(defn cube-root
  ([n] (cube-root 1 n))
  ([guess n] guess))
```

The next test requires a little bit of algebraic manipulation.  I don't want to
get the next guess and start recursing in the same step, so I need to figure out
a guess that will require another step, but only one.

$$\frac{\frac{1}{y^2} + 2y}{3} = 1$$

$$\frac{1}{y^2} + 2y = 3$$

$$1 + 2y^3 = 3y^2$$

$$2y^3 - 3y^2 + 1 = 0$$

Plugging this into a polynomial solver yields -1/2, which will suit our
purposes:

``` clojure
(deftest cube-root-test
  (is (good-enough? (cube-root 1) 1))
  (is (good-enough? (cube-root 1 1) 1))
  (is (good-enough? (cube-root -1/2 1) 1)))

(defn cube-root
  ([n] (cube-root 1 n))
  ([guess n]
	 (if (good-enough? guess n)
	   guess
	   (next-guess guess n))))
```

Now we want to drive the recursion, so let's test something that requires more
than one guess:

``` clojure
(deftest cube-root-test
  (is (good-enough? (cube-root 1) 1))
  (is (good-enough? (cube-root 1 1) 1))
  (is (good-enough? (cube-root -1/2 1) 1))
  (is (good-enough? (cube-root 8) 8)))

(defn cube-root
  ([n] (cube-root 1 n))
  ([guess n]
	 (if (good-enough? guess n)
	   guess
	   (cube-root (next-guess guess n) n))))
```
