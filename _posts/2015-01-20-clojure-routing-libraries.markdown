---
layout: post
title: "Clojure Routing Libraries"
date: 2015-01-20 11:14:25 -0500
comments: true
categories: clojure
---

There are a lot of different routing libraries available through clojure, and it's hard understand which one best fits your needs.  I find what I really wanted when I was researching them was an article comparing them.

There are a lot of libraries in the [Clojure Toolbox](http://www.clojure-toolbox.com), so to limit this article I'm only going to talk about libraries that have had a commit within the past 6 months.

Ring
====
First, I need to point out that everything's a layer on top of Ring.  As such, I'm going to establish some basic terminology:
* *Request*: Clojure map representing the HTTP request.
* *Response*: Clojure map representing the HTTP response.
* *Handlers*: Function translating requests into responses.
* *Middleware*: Functions that wrap around handlers.
I'm not going to replicate Ring's [documentation](https://github.com/ring-clojure/ring), but keep in mind that all of these frameworks are fundamentally just functions and maps.

As a simple example, suppose you want your server to respond to GET requests for /foo and /bar.  It would look something like this:

``` clojure
(defn handler [request]
  (cond (and (= (request :uri) "/foo") (= (request :request-method) :get)) "foo"
        (and (= (request :uri) "/bar") (= (request :request-method) :get)) "bar"))
```

Compojure
=========
As you can see in the previous example, a common pattern in ring handlers is to condition on the :request-method and :uri.  Compojure standardizes this process, allowing you to instead say this:

``` clojure
(defroutes handler
  (GET "/foo" [] "foo")
  (GET "/bar" [] "bar"))
```

Pedestal
========
Instead of using functions to define routing, pedestal prefers a data structure called a routing table.  It's still built on top of Ring, but ends up looking a little different.

``` clojure
(defroutes routes
  [[["/foo" {:get "foo"}]
    ["/bar" {:get "bar"}]]])
```

This will actually end up being expanded into a much more complicated data structure, but you get the idea.  I haven't had a huge amount of exposure to Pedestal, but a general principle seems to be to prefer data to functions wherever possible.

Twixt
=====
One of the cool things about how Ring is designed is it allows for a fairly arbitrary amount of layering in the middleware.  Twixt is an asset pipeline (css, javascript, etc) that's designed to complement other routing by intercepting requests to /assets/.

Conclusion
==========
When I started writing this article I thought the landscape was larger, but actually there are only a few active projects.  I think Compojure is more broadly used, but Pedestal has a lot of power I don't understand yet.
