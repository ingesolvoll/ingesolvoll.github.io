---
layout: post
title:  "Clojure railways"
date:   2016-03-23 21:30:02 +0100
---

### The problem

If you ever made a non-trivial single page application (SPA) in the browser, you probably ran into some of the same problems as me, namely:

* Functions doing HTTP requests with callbacks aren't easily composable (among other things)
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

You could get quite far with this approach. But it gets messy once you would like to reuse this code chunk but with something else happening in the end, or maybe put the whole thing in a loop after retrieving a bunch of usernames from a server. Using callbacks forces us to explicitly wire things together, making composability very hard.

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

Clojure has the brilliant [core.async library][core-async] that gives us a completely different way of working with time in code. It uses channels for delivering messages between parts of your program. The rest of this post will require some basic understanding of core.async, [I recommend this nice and friendly introduction!][async-intro]

We also need a library for doing HTTP. The [cljs-http] library leans heavily on core.async. It delivers HTTP responses on channels instead of using callbacks, and is just what we need.

### Tuning in on channels

With cljs-http and core.async, the examples above become:

{% highlight clojure %}
(defn load-customer [username]
  (go
    (let [customer (:body (<! (http/get (str "mysite/customer/byusername/" username))))
          orders (:body (<! (http/get (str "mysite/customer/" (:id customer) "/orders"))))]
      (populate-customer-dashboard customer orders))))
{% endhighlight %}

Using core.async's `go`-blocks, we can write our HTTP-requests as a regular procedural code without callbacks, and have all responses available as variables in a flattened scope. 
Using `<!` inside a `go`-block, the library will "park" your program and continue with the next line after a value becomes available. Kind of like a breakpoint in your debugger. 
The `go`-block itself returns a channel, which will eventually contain the customer dashboard HTML.

This code still has major issues though.

First of all, a go block returns a channel, so the caller of this function needs to take from that channel to get our customer dashboard. 
This is ok to some extent, but it's a bit clunky to use channels all over the place. 
Ideally we would like to have our core logic as pure functions, and use channels to transparently connect them.

Also, the code has the same problem with error handling. Currently there isn't any, and it would be implemented in much the same way as the JavaScript version, with boring and noisy if-checks after receiving responses. 
And if an error actually occurs, propagating it through your functions will hurt your nice APIs.

### Railway-oriented programming

[This is excellent talk][rop-talk] by [Scott Wlaschin][scottw-twitter] introduces the term Railway oriented programming, using F#. 
It's basically a way of talking about monads that makes it understandable to most people. The idea is to take a pure function with regular inputs and outputs,
and wrap it in a function that can accept and return either success or failure. If a failure is received, we return the failure value. On success, we call our regular wrapped function.
This way, an error will shut down the whole chain of functions as they will all just pass through the same error return.
 
You should head over to Scott's site and spend some minutes reading or watching the talk to get a much better understanding of the concept than you will ever get from me. 

### Channel oriented programming

I'm not too strong on monads and things, but I *think* I understand the main point: **A monad is a box with something in it that can be unboxed or mapped over**.
That sounds like core.async channels to me, so I'll give it a try. 

Usually when you make a functional chain that transforms data, you pass the output of one function into the next. 
I found that when working at the abstraction level of HTTP calls and application state, it's hard to compose functions in the same way. 
Quite often you need to pass things like primary keys several steps down the chain.

My way of solving this is to use [middleware style][middleware-style]. 
Each function receives accumulated upstream results as a map, and adds its own result entry to the map before passing it downstream.

### Channel wrapper functions

`=fn=` and `=http=` are my wrapper functions so far. I chose names padded with `=`, to indicate 2 tracks in and out. Let's have a look at the simplest one first:

{% highlight clojure %}
(defn fn->rail [ctx key f]
  (try
    {:success (assoc ctx key (f ctx))}
    (catch js/Object e
      {:error {:type :general :msg e}})))

(defn =fn= [input-chan key f]
  (go
    (let [{:keys [success error]} (<! input-chan)]
      (if success
        (fn->rail success key f)
        {:error error}))))
{% endhighlight %}

A wrapper always starts out by waiting for the the value of its input channel. If the value has a success key, we pass that value into our wrapped function. If we receive an error key, we short circuit and return the same error key. Upon success of the wrapped function, we `assoc` the returned value into the shared result map.

And here's the special case for HTTP::
{% highlight clojure %}
(defn response->rail [ctx key {:keys [body success]}]
  (if success
    {:success (assoc ctx key body)}
    {:error {:type :http :msg (:error body)}}))

(defn =http= [input-chan key f]
  (go
    (let [{:keys [success error]} (<! input-chan)]
      (if success
        (response->rail success key (<! (f success)))
        {:error error}))))
{% endhighlight %}

Not very different really, just needs to unwrap the data from the HTTP response before putting it on the channel.

Finally, here's the last piece of the puzzle: The `wrap-rail` function.

{% highlight clojure %}
(defn wrap-rail [f input error-handler]
  (-> (go {:success input})
      f
      error-handler))
{% endhighlight %}

It doesn't do much, just puts the provided input map on a channel, passes it to the wrappped function chain and makes sure that the any errors are caught in the end.
 
### The final result

{% highlight clojure %}
(defn populate-customer-dashboard [{:keys [customer orders]}]
  ; Produce some nice HTML for customer dashboard
  )

(defn customer-info [{username :username}]
  (http/get (str "mysite/customer/byusername/" username)))

(defn orders [{customer :customer}]
  (http/get (str "mysite/customer/" (:id customer) "/orders")))

(defn order-chain [input-channel]
  (-> input-channel
      (=http= :customer customer-info)
      (=http= :orders orders)
      (=fn= :dashboard populate-customer-dashboard)))
      
(defn error-handler [input-chan]
  (go
    (if-let [{:keys [error]} (<! input-chan)]
        ; You probably want prettier error handling than this.
        (js/alert "Sorry, error occurred: " error))))
      

(wrap-rail order-chain {:username "steve"} error-handler)
{% endhighlight %}

The `order-chain` function will  return a channel with a value looking like this, if it succeeds:
{% highlight clojure %}
{:customer  {:name "John" :email "john@john.com"}
 :orders    [{:id 1 :date "2015-23-01"} {:id 2 :date "2014-11-11"}]
 :dashboard "<html>Nicely presented dashboard here<html>"}
{% endhighlight %}

Now, if you would like to use `order-chain` for something different and still keep the nice guarantees provided by the architecture, that's trivial.

{% highlight clojure %}
(defn username-search [{query :query}]
  (http/get "mysite/search" {:q query}))

(defn search-for-orders [input-channel]
  (-> input-channel
      (=http= :username username-search)
      order-chain))
      
(wrap-rail search-for-orders {:query "ste*"} error-handler)
{% endhighlight %}


There are, of course, some trade offs being made here, so a quick summary of pros and cons is in order:

### Pros

* Asynchronous code can be written as a simple procedure
* Error handling is transparent and predictable
* Short circuiting on error makes corrupted app states less likely.
* Easy to inspect the data flowing through the chain
* Functions are as pure as they can be
* [Time is not important][time], your operations are guaranteed to be executed in order.

### Cons

* Each function must accept a map as input, and know the shape of the data in it.
* When code in core.async channels blow up, debugging can be a pain

### Conclusion

This approach has proven to be quite efficient on my most recent project, the ability to freely reuse asynchronous
functions to chain together operations from the UI-layer is quite useful and a lot of fun. Some extra complexity was introduced
to make it happen, but the result is quite powerful.

I would consider this micro-architecture to be an experiment, it could be greatly improved or even completely replaced with something else.
That's the beauty of Clojure, the conciseness and functional style tends to make code easily replaceable. 
If you don't like something, delete it and put in the thing you want. 
I'm hoping to expand on this in future blog posts, looking at things like:
 
* Use of macros for smoother API and more functional flex and power
* Support for more flexibility, like iterations, structure of input/output
* Support for doing custom error handling where needed.

[time]: https://www.youtube.com/watch?v=mtlTHmM1u10
[middleware-style]: http://clojure-doc.org/articles/cookbooks/middleware.html
[core-async]: https://github.com/clojure/core.async
[async-intro]: http://www.braveclojure.com/core-async/
[cljs-http]: https://github.com/r0man/cljs-http
[rop-talk]: https://fsharpforfunandprofit.com/rop/
[scottw-twitter]: https://twitter.com/scottwlaschin