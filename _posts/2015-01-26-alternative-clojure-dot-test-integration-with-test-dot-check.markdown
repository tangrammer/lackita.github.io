---
layout: post
title: "Alternative clojure.test Integration With test.check"
date: 2015-01-26 07:32:40 -0500
comments: true
categories: clojure testing
---

I've enjoyed using [test.check](https://github.com/clojure/test.check) lately, but its integration with [clojure.test](https://clojure.github.io/clojure/clojure.test-api.html) doesn't fit with how I want to use it very well.  In this post we're going to explore a different approach, which I'm going to try to get into [test.chuck](https://github.com/gfredericks/test.chuck).

The Problem
-----------
To demonstrate my difficulty, consider these basic tests in `clojure.test`:

``` clojure
(deftest integer-facts
  (testing "positive"
    (is (> 1 0))
    (is (> 24 0)))
  (testing "negative"
    (is (< -1 0))
    (is (< -24 0))))
```

Instead of checking hard-coded integers, I would prefer `test.check` to do its magic.  To do this using `defspec` (the existing integration option), I would have to do something like this:

``` clojure
(defspec positive-integer-facts 100
  (prop/for-all [i gen/s-pos-int]
    (> i 0)))

(defspec negative-integer-facts 100
  (prop/for-all [i gen/s-neg-int]
    (< i 0)))
```

There's a lot of boilerplate here, and it's not communicating that these two properties are related.  Alternatively, I could join everything into one big condition:

``` clojure
(defspec integer-facts 100
  (prop/for-all [p gen/s-pos-int
                 n gen/s-neg-int]
    (and (> p 0)
         (< n 0))))
```

This is also unsatisfying, as I wouldn't be able to tell easily what failed.

Setting aside that issue for now, let's look at the output:

```
lein test checking.core-test
{:test-var "positive-integer-facts", :result true, :num-tests 100, :seed 1422280152456}
{:test-var "negative-integer-facts", :result true, :num-tests 100, :seed 1422280152465}

Ran 2 tests containing 2 assertions.
0 failures, 0 errors.
```

Unlike `deftest`, `defspec` prints a report even when it passes.  If I start using properties liberally my test output will quickly get too noisy.

The transition path is also difficult.  Look at the original `deftest` code and notice that moving to `defspec` feels like a complete rewrite, as opposed to an upgrade from hard-coded to generated values.

Also, consider the case where we're performing multiple assertions stepping through a stateful piece of code.

``` clojure
(deftest counter
  (testing "increasing"
    (let [c (atom 2)]
      (swap! c inc)
      (is (= @c (inc 2)))
      (swap! c inc)
      (is (> @c 0)))))
```

Moving this to `defspec` can be tricky:

``` clojure
(defspec increasing-counter 100
  (prop/for-all [i gen/s-pos-int]
    (let [c (atom i)]
      (swap! c inc)
      (let [single-inc @c]
        (swap! c inc)
        (and (= single-inc (inc i))
             (> @c 0))))))
```

The Alternative
---------------
What I'd really like to say is exactly the same thing as the original `deftest` code, with just a little bit of variation for the generated values:

``` clojure
(deftest integer-facts
  (checking "positive" [i gen/s-pos-int]
    (is (> i 0)))
  (checking "negative" [i gen/s-neg-int]
    (is (< i 0))))

(deftest counter
  (checking "increasing" [i gen/s-pos-int]
    (let [c (atom i)]
      (swap! c inc)
      (is (= @c (inc i)))
      (swap! c inc)
      (is (> @c 0)))))
```

Notice how simple it is to move from `testing` to `checking`.  With `defspec` the code screams generative testing and mentions the assertions as an aside.  With `checking` generative testing becomes the aside and the assertions becomes the focus.

Naive Implementation
--------------------
A first pass on `checking` looks something like this:

``` clojure
(defmacro checking [name bindings & body]
  `(testing ~name
     (tc/quick-check 100
       (prop/for-all ~bindings ~@body))))
```

This works because the `is` macro completely bypasses the `tc/quick-check` construct.  While this gets us limping along, there are a couple problems.

First, let's force a failure:

``` clojure
(deftest failure
  (checking "incorrect" [i gen/pos-int]
    (is (< i 50))))
```

Now look at the output for this failure:

``` clojure
FAIL in (failure) (core_test.clj:64)
incorrect
expected: (< i 50)
  actual: (not (< 55 50))

lein test :only checking.core-test/failure

FAIL in (failure) (core_test.clj:64)
incorrect
expected: (< i 50)
  actual: (not (< 52 50))

lein test :only checking.core-test/failure

...
```

We're seeing a failure for every attempt when `tc/quick-check` tries to narrow down to the smallest failure.  All we really want to know about is the result of this search.

The other problem is that `tc/quick-check` only traces when the final sexp is false:

``` clojure
(deftest unsearched-failure
  (checking "incorrect" [i gen/pos-int]
    (is (< i 50))
    (is (= i i))))
```

Which becomes obvious in the test output:

``` clojure
FAIL in (unsearched-failure) (core_test.clj:68)
incorrect
expected: (< i 50)
  actual: (not (< 57 50))

lein test :only checking.core-test/unsearched-failure

FAIL in (unsearched-failure) (core_test.clj:68)
incorrect
expected: (< i 50)
  actual: (not (< 61 50))

...

lein test :only checking.core-test/unsearched-failure

FAIL in (unsearched-failure) (core_test.clj:68)
incorrect
expected: (< i 50)
  actual: (not (< 90 50))
```

Intercepting Reporting
----------------------
We're seeing the failure, but getting a lot of noise as well.  What we really need here is an alternative test-reporting framework that can be nested within the checking macro.  Fortunately, `clojure.test` is designed to replace that framework by overriding the report multimethod.  Let's look at what one of these reports looks like:

``` clojure
user> (let [result (atom nil)] (with-redefs [clojure.test/report #(reset! result %)]
        (is (= 1 2))) @result)
{:message nil, :actual (not (= 1 2)), :expected (= 1 2), :type :fail,
 :file "form-init7960148245055492106.clj", :line 1}

user> (let [result (atom nil)] (with-redefs [clojure.test/report #(reset! result %)]
        (is (= 1 1))) @result)
{:type :pass, :expected (= 1 1), :actual (#<core$_EQ_ clojure.core$_EQ_@39877a52> 1 1),
 :message nil}

user> (let [result (atom nil)] (with-redefs [clojure.test/report #(reset! result %)]
        (is (/ 1 0))) @result)
{:message nil, :actual #<ArithmeticException java.lang.ArithmeticException: Divide by zero>,
 :expected (/ 1 0), :type :error, :file "Numbers.java", :line 156}
```

So the condition `quick-check` is looking for is actually if :type is not :pass.  If we could catch that investigations would work correctly:

``` clojure
(defmacro checking [name bindings & body]
  `(testing ~name
     (is (:result (tc/quick-check 100
                    (prop/for-all ~bindings
                      (let [pass# (atom true)]
                        (with-redefs [report #(swap! pass#
                                                     (fn [r#] (and r# (= (:type %) :pass))))]
                          ~@body)
                        @pass#)))))))
```

Output:

```
FAIL in (searched-failure) (core_test.clj:63)
incorrect
expected: (:result (clojure.test.check/quick-check 100 (clojure.test.check.properties/for-all [i gen/pos-int] (clojure.core/let [pass__616__auto__ (clojure.core/atom true)] (clojure.core/with-redefs [clojure.test/report (fn* [p1__615__617__auto__] (clojure.core/swap! pass__616__auto__ (clojure.core/fn [r__618__auto__] (clojure.core/and r__618__auto__ (clojure.core/= (:type p1__615__617__auto__) :pass)))))] (is (< i 50))) (clojure.core/deref pass__616__auto__)))))
  actual: false
```

Tracking Reports
----------------
This guarantees that the investigation happens, but we've lost the individual assertions in the process.  What we need is a way to get at only those reports which were generated in the final failing execution.  There's not a mechanism for passing information out of a failure, so we'll have to use a closure with some state to simulate the effect:

``` clojure
(defmacro checking [name bindings & body]
  `(testing ~name
     (let [final-reports# (atom [])]
       (tc/quick-check 100
        (prop/for-all ~bindings
          (let [reports# (atom [])]
            (with-redefs [report #(swap! reports# conj %)]
              ~@body)
            (let [pass# (every? #(= (:type %) :pass) @reports#)]
              (when (or (not pass#) (empty? @final-reports#))
                (reset! final-reports# @reports#))
              pass#))))
       (doseq [r# @final-reports#]
         (report r#)))))
```

And the output:

```
lein test :only checking.core-test/unsearched-failure

FAIL in (unsearched-failure) (core_test.clj:68)
incorrect
expected: (< i 50)
  actual: (not (< 50 50))
```

Including Results
-----------------
So things are functioning correctly, and the failures are easy to read.  In more complex scenarios the value may have been transformed heavily before the assertion is made, in which case we want to present the `tc/quick-check` return value.

``` clojure
(defmacro checking [name bindings & body]
  `(testing ~name
     (let [final-reports# (atom [])]
       (let [result# (tc/quick-check 100
                       (prop/for-all ~bindings
                         (let [reports# (atom [])]
                           (with-redefs [report #(swap! reports# conj %)]
                             ~@body)
                           (let [pass# (every? #(= (:type %) :pass) @reports#)]
                             (when (or (not pass#) (empty? @final-reports#))
                               (reset! final-reports# @reports#))
                             pass#))))]
         (is (:result result#) result#))
       (doseq [r# @final-reports#]
         (report r#)))))
```

Output:

```
lein test :only checking.core-test/unsearched-failure

FAIL in (unsearched-failure) (core_test.clj:67)
incorrect
{:result false, :seed 1422311111994, :failing-size 58, :num-tests 59, :fail [55],
 :shrunk {:total-nodes-visited 23, :depth 3, :result false, :smallest [50]}}
expected: (:result result__618__auto__)
  actual: false

lein test :only checking.core-test/unsearched-failure

FAIL in (unsearched-failure) (core_test.clj:68)
incorrect
expected: (< i 50)
  actual: (not (< 50 50))
```

Cleaning Up
-----------
I'm happy with how things are reported now, but the code is a pretty big mess.  Decomposing the logic should help with that:

``` clojure
(defn report-when-failing [result]
  (is (:result result) result))

(defmacro capture-reports [body]
  `(let [reports# (atom [])]
     (with-redefs [report #(swap! reports# conj %)]
       ~@body)
     @reports#))

(defn pass? [reports]
  (every? #(= (:type %) :pass) reports))

(defn report-needed? [reports final-reports]
  (or (not (pass? reports)) (empty? final-reports)))

(defn save-to-final-reports [reports final-reports]
  (when (report-needed? reports @final-reports)
    (reset! final-reports reports)))

(defmacro checking [name bindings & body]
  `(testing ~name
     (let [final-reports# (atom [])]
       (report-when-failing (tc/quick-check 100
                              (prop/for-all ~bindings
                                (let [reports# (capture-reports ~body)]
                                  (save-to-final-reports reports# final-reports#)
                                  (pass? reports#)))))
       (doseq [r# @final-reports#]
         (report r#)))))
```

Number of Tests
---------------
One final touch is that the number of tests really shouldn't be hard-coded.  This adds to the `checking` footprint slightly, but that seems worth it to simplify our ability to control the number of tests being run.

``` clojure
(defn report-when-failing [result]
  (is (:result result) result))

(defmacro capture-reports [body]
  `(let [reports# (atom [])]
     (with-redefs [report #(swap! reports# conj %)]
       ~@body)
     @reports#))

(defn pass? [reports]
  (every? #(= (:type %) :pass) reports))

(defn report-needed? [reports final-reports]
  (or (not (pass? reports)) (empty? final-reports)))

(defn save-to-final-reports [reports final-reports]
  (when (report-needed? reports @final-reports)
    (reset! final-reports reports)))

(defmacro checking [name tests bindings & body]
  `(testing ~name
     (let [final-reports# (atom [])]
       (report-when-failing (tc/quick-check ~tests
                              (prop/for-all ~bindings
                                (let [reports# (capture-reports ~body)]
                                  (save-to-final-reports reports# final-reports#)
                                  (pass? reports#)))))
       (doseq [r# @final-reports#]
         (report r#)))))
```

Update
======
The macro has been accepted to [test.chuck](https://github.com/gfredericks/test.chuck) in release 0.1.12.  [Gary Fredericks](https://github.com/gfredericks) helped me work out some concurrency problems with the above implementation.

First, `with-redefs` rebinds things globally, which means that we can end up saving to the wrong atom.  It's much better to use `binding`:

``` clojure
(defmacro capture-reports [body]
  `(let [reports# (atom [])]
     (binding [report #(swap! reports# conj %)]
       ~@body)
     @reports#))
```

Second, using `reset!` in `save-to-final-reports` causes a race condition between checking the condition and assigning to the atom.  To get around this we can rearrange the arguments of `save-to-final-reports` and call `swap!` instead:

``` clojure
(defn save-to-final-reports [final-reports reports]
  (if (report-needed? reports final-reports)
    reports
    final-reports))

(defmacro checking
  [name tests bindings & body]
  `(testing ~name
     (let [final-reports# (atom [])]
       (report-when-failing (tc/quick-check ~tests
                              (prop/for-all ~bindings
                                (let [reports# (capture-reports ~body)]
                                  (swap! final-reports# save-to-final-reports reports#)
                                  (pass? reports#)))))
       (doseq [r# @final-reports#]
         (report r#)))))
```
