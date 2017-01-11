---
layout: klipse
title:  "JSX vs Clojurescript: the showdown"
date:   2017-01-05 19:00:00 +0100
---

### Better or different?
In this post i would like to show off some of the differences between using "plain" React and a Clojurescript wrapper like Reagent. Not an easy subject to cover in one blog post, so I'll have to focus on a simple example. The official [tutorial] on the ReactJS web pages seems like a good fit.

The app we're making is a Tic-tac-toe game. The most basic building block for this game will be the `square` component, representing each of the 9 squares on the board. The very minimal contents of this component showcases some of the primary differences between vanilla React and "The Clojurescript way". Here's the JSX version:

{% highlight js %}
function Square(props) {
  return (
    <button className="square" onClick={() => props.onClick()}>
      {props.value}
    </button>
  );
}
{% endhighlight %}

Component parameters are received as a map of `props`. In order to avoid noisy low level markup code, React uses a special HTML-like language called JSX to describe the layout. The result is much more concise code, at the cost of an additional build step. One of the pain points of JSX is that it breaks tooling. As you can see above, my blog platform does not have code highlighting support for JSX.

Now over to the Reagent version:

{% highlight clojure %}
(defn square [value on-click]
  [:button.square {:on-click on-click}
   value])
{% endhighlight %}

The most striking difference is the Hiccup syntax for representing markup. Hiccup is a Clojure standard that uses regular Clojure vectors and maps to represent tags and attributes. No special syntax or language features is required to manipulate this structure, regular Clojure functions will do just fine. Having the full Clojure standard library at your disposal for generating HTML is very powerful.

Another thing to notice is the component parameters. React puts them into a `props` map, whereas Reagent uses standard Clojure function parameters. **A Reagent component is just a function, component parameters are just function parameters.**

### State management
Reagent, being a wrapper around React, shares React's philosophy of centralized state and one-way data flow. In React, the way to trigger UI updates is to use `this.setState(nextState)`. Let's see an example from our tic-tac-toe game:

{% highlight js %}
handleClick(i) {
    if (this.state.squares[i]) { return; }
    const squares = this.state.squares.slice();
    squares[i] = this.state.xIsNext ? 'X' : 'O';
    this.setState({squares: squares,
                   xIsNext: !this.state.xIsNext});
  }
{% endhighlight %}

Before we show the equivalent click handler in Reagent, we need to cover the subject of `atoms`. An `atom` is a generic data container that's able to notify its surroundings when the data inside change. `this.setState(something)` in React becomes `(reset! state something)` in Reagent. State in React is tied to the implementation of components, in Reagent it is a standalone concept.

{% highlight clojure %}

(defn make-a-move [state i]
  (if (-> state :squares (get i))
    state
    (-> state
      (assoc-in [:squares i] (if (:x-is-next state) "X" "O"))
      (update-in [:x-is-next] not))))

(defn handle-click [i]
  (swap! state make-a-move i))
{% endhighlight %}

For the Reagent version above I chose to do it the Clojure way, putting the pure code in a separate function that does nothing but transforming the state map. All that's left for the click handler to do is to mutate the state using a pure function and `atom` semantics.



Before we get started, we need to load ReactJS and Reagent (Clojurescript wrapper for React) onto the page:
<pre><code class="language-klipse">
(require '[reagent.core :as r])
</code></pre>


<pre><code class="jsx" data-gist-id="ingesolvoll/101256657c664581832db5a231178cc7">
</code></pre>

<pre><code class="language-reagent" data-gist-id="https://gist.github.com/ingesolvoll/3107df3d65599496c42bbc086427c5b0">
</code></pre>


<style>
.board-row:after {
  clear: both;
  content: "";
  display: table;
}

.square {
  background: black;
  color: white;
  border: 1px solid #999;
  float: left;
  font-size: 24px;
  font-weight: bold;
  line-height: 34px;
  height: 34px;
  margin-right: -1px;
  margin-top: -1px;
  padding: 0;
  text-align: center;
  width: 34px;
}

.game {
  display: flex;
  flex-direction: row;
}

.game-info {
  margin-left: 20px;
}
</style>

[tutorial]: https://facebook.github.io/react/tutorial/tutorial.html
