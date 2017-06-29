---
layout: klipse
title:  "Re-frame: side effects as data"
date:   2017-06-29 12:00:00 +0100
---

### Re-frame

I'm going to assume that you have some basic knowledge about [Re-frame],
the very popular lightweight application framework for [Reagent] applications. The architecture is very
similar to React Redux, in that you have *event handlers / reducers* that take the current version of your
state as an argument and returns a transformed version of the state.

### Data first
Clojure is all about data. It has a huge standard library that allows you to manipulate your immutable data structures
in every possible way.
This is a great tool for reducers when doing their data crunching.

### How about those nasty side effects?
Any serious app needs to do HTTP, navigation, cookies, local storage, and other kinds of effects that are not about manipulating
some data structure. It turns out that the re-frame developers did some very serious thinking about this. They came up with
the concept of *pluggable effects*. Instead of executing functions you will return data that tells some other party how to
get the job done.

Rather than trying to explain this in words, I will use the amazing KLIPSE
plugin to let you play around with these concepts yourself! Feel free to edit the code below to see what happens.
### Boot it up!
First, we need to require re-frame, and transitively reagent, into this page

<pre><code class="language-klipse">
(require '[re-frame.core :as re-frame])
</code></pre>

#### Introducing events

This is a typical re-frame event handler. We use `reg-event-fx` to get access to both incoming and outgoing side effects.


<pre><code class="language-klipse">

(re-frame/reg-event-fx
:show-message
    (fn [incoming-effects [event-key message]]
        (js/alert (str "I was DIRECTLY asked to print this: " message))))
nil
</code></pre>

As you can see, the handler shows the provided message in a native javascript `alert` and returns nil (meaning nothing changed and there is nothing more that the framework needs to take care of).
Click the button below to trigger the event and see the effect.

<pre><code class="language-reagent">
[:button
    {:on-click (fn [e] (re-frame/dispatch [:show-message "42"]))}
    "Click me to alert something!"]
</code></pre>

For many cases, this will be just fine. But in a more complex system with less visible side effects, you quickly lose track
of where you are when debugging. One thing that really helps in these cases is a full and complete overview of the code paths, after the fact.
If we execute our side effects inside the black boxes that functions are, we have no such overview.

### Effects as data

If we manage to express our side effects as data, this overview becomes trivial to make. To do this we need a couple of building
blocks. First out are pluggable effects. You are free to create your own types of side effects, using re-frame's `reg-fx` function.

<pre><code class="language-klipse">
(re-frame/reg-fx
    :alert
    (fn [message]
        (js/alert (str "I was INDIRECTLY asked to print this: " message))))
nil
</code></pre>

Nothing too fancy there. We are just introducing a level of indirection, letting the framework do the work of connecting
our data to the actual effect.

### Logging

To get the full trace that we want, we need a way to inspect the return value of this handler after it executed.

Re-frame uses [interceptors] for cross-cutting concerns like this. The one below runs after the event handler.
Through its `context` parameter it has access to all metadata, including the return value of the handler. The `effects` function
gives us the handler return value.

<pre><code class="language-klipse">

(require '[re-frame.interceptor :refer [->interceptor get-effect]])

(def debug-interceptor
  (->interceptor
    :id     :log-effects
    :after  (fn [context]
             (let [effects (get-effect context)
                   event (re-frame/get-coeffect context :event)]
               (if (seq effects)
                 (js/alert (str "Event " event " caused side effect " effects)))
               context))))
</code></pre>

### A "pure" side effecting handler

Now we have what we need to be able to inspect or test our side effect. We inject our interceptor into the event handler when
registering it.

<pre><code class="language-klipse">
(re-frame/reg-event-fx
    :show-message-with-indirection
    [debug-interceptor]
    (fn [_ [_ message]] {:alert message}))
nil
</code></pre>

And finally, here's a snippet that uses our new handler.

<pre><code class="language-reagent">
[:button
    {:on-click (fn [e] (re-frame/dispatch [:show-message-with-indirection "42"]))}
    "Click me to alert something!"]
</code></pre>

The difference between `(js/alert message)` and `{:alert message}` might seem insignificant. But having the trace of data
at your fingertips makes a world of a difference when you're stuck trying to figure out why your chain of HTTP requests
 has stalled. Unit testing also becomes trivial, as opposed to sniffing the presence of an alert box.

### Event ping pong

I'm not super happy about the callback style of re-frame. Every time you need to do something asynchronous like HTTP, you
need to name the event handler that should receive the callback. It very quickly turns into something that could be called
 "event ping pong" or even "callback hell". [keechma] is a very interesting alternative framework, the [pipelines] are particluarly interesting.
 But keechma does not have the same focus on effects as data.

I would love to see something like the keechma pipelines for re-frame, and I'm experimenting to see if I can find a nice
and practical solution.

[interceptors]: https://github.com/Day8/re-frame/blob/master/docs/Interceptors.md
[re-frame]: https://github.com/Day8/re-frame
[reagent]: https://reagent-project.github.io/
[keechma]: https://keechma.com/
[pipelines]: https://keechma.com/news/introducing-keechma-toolbox-part-1-pipelines/