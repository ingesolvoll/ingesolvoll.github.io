---
layout: post
title:  "Clojure railways"
date:   2016-03-23 21:30:02 +0100
---

### The problem

If you ever made a non-trivial single page application (SPA) in JavaScript, you probably ran into some of the same problems as me, namely:

* Functions doing HTTP requests with callbacks aren't easily composable
* Taking error handling seriously pollutes your beautiful happy path code

So let's have a look at a typical JavaScript controller that retrieves data from the server:

{% highlight javascript %}
function loadCustomer(username) {
  $.get('mysite/customer/byusername/' + username).then(function(customer) {
    $.get('mysite/customer/' + customer.id + '/order').then(function(orders) {
        populateCustomerDashboard(customer, orders);
    });
  });
}
{% endhighlight %}

You could get quite far with this approach. But it gets messy once you would like to reuse this code chunk but with something else happening in the end, or maybe put the whole thing in a loop after retrieving a bunch of usernames from a server. Using callbacks forces us to explicitly wire things together, making composability possible but far from elegant and natural.

And we did not even get started on the error handling. Here's the same code, with error callbacks:

{% highlight javascript %}
function loadCustomer(username) {
  $.get('mysite/customer/byusername/' + username).then(function(customer) {
    $.get('mysite/customer/' + customer.id + '/order').then(function(orders) {
        populateCustomerDashboard(customer, orders);
    }).error(function(error) {
          // Some proper error handling here
        });
  }).error(function(error) {
    // Some proper error handling here
  });
}
{% endhighlight %}

Quite quickly, our quite simple logic is buried in deep nesting, duplication and noise. Seeing this, I started looking for solutions that would allow me to:

* Write my asynchronous logic in a clean and linear way, without nesting and callbacks
* Apply robust error handling transparently
* Leverage host language's built in features, avoiding custom made constructs

### Clojurescript and core.async to the rescue

I really like Clojure (script) and after working with it for a while, I got some ideas on how to apply these ideas here. Most of it should be pretty straightforward, as both F# and Clojure have a strong functional approach.

Clojure has the brilliant core.async library that gives us a completely different way of working with time in code. It uses channels for delivering messages between parts of your program. The rest of this post will require some basic understanding of core.async, I recommend this nice and friendly introduction !

We also need a library for doing HTTP. cljs-http leans heavily on core.async. It delivers HTTP responses on channels instead of using callbacks, and is just what we need.

### Tuning in on channels

With cljs-http and core.async, the examples above become:

{% highlight clojure %}
(defn load-customer [username]
  (go
    (let [customer (<! (http/get (str "mysite/customer/byusername/" username)))
          orders (<! (http/get (str "mysite/customer/" (:id customer) "/orders")))]
      (populate-customer-dashboard customer orders))))
{% endhighlight %}

Using core.async's go-blocks, we can write our HTTP-requests as a regular procedural code without callbacks, and have all responses available as variables in a flattened scope. Using <! inside a go-block, the library will "park" your program and continue with the next line after a value becomes available. Kind of like a breakpoint in your debugger.

This code still has major issues though.

First of all, a go block returns a channel, so the caller of this function needs to take from that channel to get our customer dashboard. This is ok to some extent, but it's a bit clunky to use channels and go-blocks all over the place. Ideally we would like to have our core logic as pure functions, and use channels to transparently connect them.

Also, the code has the same problem with error handling. Currently there isn't any, and it would be implemented in much the same way as the JavaScript version, with boring and noisy if-checks after receiving responses. And if an error actually occurs, propagating it through your functions will hurt your nice APIs.

### Railway-oriented programming

[This is excellent talk][rop-talk] by [Scott Wlaschin][scottw-twitter] introduces the term Railway oriented programming, using F#. It's basically a way of talking about monads that makes it understandable to most people. You should watch it first to get a much better understanding of the concept than you will ever get from me. The talk introduces some very good solutions for transparent error handling. It also does a good job leveraging the functional features of F# rather than making a custom framework.

It does however not deliver a very strong answer on the asynchronous part.

### Channel oriented programming

I could have re-created the same thing in Clojure that Scott did in F#, wrapping pure functions in dual-track adapters that fit nicely into each other in a chain. But I wanted to take it a step further, taking time into account. So I landed on letting the adapter function use channels as input and output. That way it doesn't matter if the inner function is asynchronous or not.

Usually when you make a functional chain that transforms data, you pass the output of one function into the next function. I found that when working at the abstraction level of HTTP calls and application state, it's hard to compose functions in the same way. Quite often you need to pass things like primary keys several steps down the chain.

My way of solving this is to use middleware style. Each function receives accumulated upstream results as a map, and adds its own result entry to the map before passing it downstream.

Let's start by looking at the final code and walk through the small framework needed afterwards:

{% highlight clojure %}
(defn populate-customer-dashboard [{:keys [customer orders]}]
  {:customer-name (:name customer)
   :order-list    (map make-order orders)})

(defn customer-info [{username :username}]
  (http/get (str "mysite/customer/byusername/" username)))

(defn orders [{customer :customer}]
  (http/get (str "mysite/customer/" (:id customer) "/orders")))

(defn order-chain [input-channel]
  (-> input-channel
      (=http= :customer customer-info)
      (=http= :orders orders)
      (=fn= :dashboard populate-customer-dashboard)))

(railway/wrap-rail order-chain {:username "steve"} error-handler)
{% endhighlight %}

[rop-talk]: https://fsharpforfunandprofit.com/rop/
[scottw-twitter]: https://twitter.com/scottwlaschin