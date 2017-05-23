---
layout: post
title: "ClojureScript On Google Cloud Functions"
date: 2017-05-13 7:14:00 -0400
comments: true
categories: clojure
---

I wanted to try out [Google Cloud Functions](https://cloud.google.com/functions/),
and when I saw that it only supported JavaScript I thought "What a
perfect opportunity to try [ClojureScript](https://github.com/clojure/clojurescript)
as well. I could have imitated (https://github.com/MartinSahlen/cloud-functions-clojure)
and that would have been fine, but I started going through the
[Modern ClojureScript Tutorials](https://github.com/magomimmo/modern-cljs)
and got taken by [boot](https://github.com/boot-clj). (See how easy it
is for me to get lost down a rabbit hole?)

Ultimately, I'm only going to end up having two files:

```
.
├── build.boot
└── src
    └── cljs
        └── demo
            └── core.cljs
```
I did this in phases, figuring out one piece at a time. I'll be
walking you through that process, as I think it will make the
information easier to digest.

To begin with I just wanted to get a minimal ClojureScript project
going. Most of this part is lifted straight from
[Tutorial 1 of Modern ClojureScript](https://github.com/magomimmo/modern-cljs/blob/master/doc/second-edition/tutorial-01.md),
but even simpler because I don't need an html file. I'm not going to
repeat what's explained in the tutorial, he does a much better job of
describing what's going on.

``` clojure
;; core.cljs
(ns demo.core)

(defn greet [req res]
  (println "hey")
  (.json res (clj->js (:headers req))))

(enable-console-print!)
```

``` clojure
;; build.boot
(set-env!
 :source-paths #{"src/cljs"}
 :dependencies '[[adzerk/boot-cljs "1.7.228-2"]])

(require '[adzerk.boot-cljs :refer [cljs]])
```

Now to start making it possible for cloud functions to be able to run
this function. There's a command line tool that we'll get to later,
but for now let's focus on just being able to paste in some raw
javascript. First, for cloud functions to be able to call `greet` it
needs to be added to the export module. Once I figured out what I was
looking for, it was pretty easy to pull out of
`cloud-functions-clojure` and add to the end of `core.cljs`:

``` clojure
(set! (.-exports js/module) #js {:greet greet})
```

With that in place, I ran `boot cljs --optimizations simple target` to
generate the JavaScript. The nuance here is `--optimizations simple`,
without that, it will use the optimization `none` which will scatter
the code across several files. With the optimization, all the code I
need is generated in `target/main.js`. Then I headed over to Google's
cloud console, and set up a new function. Made the trigger an HTTP
trigger, pasted the contents of `main.js` into the inline editor and
set the function to execute to `greet`.

So I have valid javascript generated, but it's pretty annoying to be
pasting into a text box to deploy a new version. Next up, I used
the gcloud cli to deploy automatically. The command I want to use
is `gcloud beta functions deploy greet --stage-bucket demo_staging --trigger-http`,
but there are two problems right now with that: it expects the
javascript to be in the directory where the command is run and it
expects the file to be named `index.js`. The former I resolved by
changing the command to
`gcloud beta functions deploy greet --local-path target --stage-bucket demo_staging --trigger-http`.
For the latter I'll switch the build command to `boot cljs --optimizations simple --ids index target`.

Now to get this all a bit more operationalized. I'd like to be able to
just say `boot deploy -f greet`, but first I need to understand how to
create new tasks. `deftask` will allows me to build the command, and a
less documented function in boot called `dosh` allows me to execute
the gcloud command. I used `with-pre-wrap` to make it easier to pass
the fileset object down the stack, as well as ensure that a deploy
happens before any tasks chained with it are called. This is added to
the end of `build.boot`.

``` clojure
(deftask deploy [f function NAME str "Function to deploy"]
  (comp (cljs :optimizations :simple :ids #{"index"})
  (target)
  (with-pre-wrap fileset
    (dosh "gcloud" "beta" "functions" "deploy" function "--local-path" "target"
          "--stage-bucket" "demo_staging" "--trigger-http")
    fileset)))
```
