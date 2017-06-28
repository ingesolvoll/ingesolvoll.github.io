---
layout: klipse
title:  "Tell me what, not how!"
date:   2017-06-05 19:00:00 +0100
---

### Re-frame

Re-frame is a very generic and lightweight application framework for Clojurescript.

<pre><code class="language-klipse">
(require '[re-frame.core :as re-frame])
</code></pre>

<pre><code class="language-klipse">

(re-frame/reg-event-fx :show-message (fn [_ [_ message]]
                                    (js/alert (str "I was DIRECTLY asked to print this: " message))))
nil
</code></pre>

<pre><code class="language-reagent">
[:button
    {:on-click (fn [e] (re-frame/dispatch [:show-message "42"]))}
    "Click me to alert something!"]
</code></pre>


<pre><code class="language-klipse">
(re-frame/reg-fx :notify (fn [message] (js/alert (str "I was INDIRECTLY asked to print this: " message))))
nil
</code></pre>


<pre><code class="language-klipse">

(require '[re-frame.interceptor :refer [->interceptor get-effect]])

(def debug-interceptor
  (->interceptor
    :id :log-effects
    :after (fn [context]
             (let [effects (get-effect context)
                   event (re-frame/get-coeffect context :event)]
               (if (seq effects)
                 (js/alert (str "Event " event " caused side effect " effects)))
               context))))
</code></pre>

<pre><code class="language-klipse">
(re-frame/reg-event-fx
    :show-message-with-indirection
    [debug-interceptor]
    (fn [_ [_ message]] {:notify message}))
nil
</code></pre>



<pre><code class="language-reagent">
[:button
    {:on-click (fn [e] (re-frame/dispatch [:show-message-with-indirection "42"]))}
    "Click me to alert something!"]
</code></pre>