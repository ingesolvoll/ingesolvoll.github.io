---
layout: post
title:  "JSX vs Clojurescript: the showdown"
date:   2017-06-22 19:00:00 +0100
---

> > What's the point in using ClojureScript with Om/Reagent/Rum/Quiescent instead of plain ReactJS? Is it better or is it just different?

This excellent question was asked on [reddit] some time ago. Many commented, most of them seem to not have read the question. I would like to try to answer it in this post. It will be superficial, but hopefully entertaining and informative!

We'll be comparing ReactJS/JSX (JavaScript) with [Reagent] (ClojureScript ReactJS wrapper). I'll use the official ReactJS [tutorial] as my example, exploring the differences.

**I've tried to keep the code as equal as possible between the languages.**  

### 1. A trivial component
The app we're making is a Tic-tac-toe game. The most basic building block for this game will be the `square` component, representing one clickable square on the board.

**JSX**
{% highlight js %}
const Square = (props) =>
<button className="square" onClick={props.onClick}>
  {props.value}
</button>
{% endhighlight %}

**Reagent**

{% highlight clojure %}
(defn square [value on-click]
  [:button.square {:on-click on-click}
   value])
{% endhighlight %}

Clojure is without doubt a more concise language than JS. It also has a very neat standard for representing markup, called **Hiccup**. Here's the complete Hiccup syntax guide:

`[:tagname#elementId.cssClassName {:some "attribute"} child-content-here]`

What you see is immutable Clojure data structures. Your markup is literally data that will be converted to React components later on. But right now it's data, which is **a much bigger deal than you might think.**

In the opposite corner our React square has some proprietary properties:

* A custom way of conveying component parameters (`props`)
* Renames the CSS property `class` to `className`
* Special syntax `<div>{props.theMessage}</div>` for putting dynamic content in markup
* Embedding HTML-like markup inside JS (JSX)

Some people love JSX, others don't. The code formatter on this blog clearly doesn't. I won't get into that discussion, there's a more interesting point to make: **JSX produces function calls, Reagent produces data**. More on that later.

### 2. Event listeners and state

When you click on a square, our handler is called. I modified the official example slightly, putting the side effect free code in a [pure function].

**Javascript**
{% highlight js %}

function makeMove(state, i) {
  if (state.squares[i]) {
    return state;
  }
  else {
    const squares = state.squares.slice();
    squares[i] = state.xIsNext ? 'X' : 'O';
    return {squares: squares,
            xIsNext: !this.state.xIsNext}
  }
}

handleClick(i) {
  this.setState(makeMove(this.state, i));
}
{% endhighlight %}


**Clojurescript**
{% highlight clojure %}

(defn make-move [state i]
  (if (-> state :squares (get i))
    state
    (-> state
      (assoc-in [:squares i] (if (:x-is-next state) "X" "O"))
      (update-in [:x-is-next] not))))

(defn handle-click [i]
  (swap! state make-move i))
{% endhighlight %}

The Javascript code listed above is written with immutability in mind, because the React developers see the value of promoting that style. But functional programming in Javascript requires knowledge and discipline. Like applying the little `slice()` copy trick in `makeMove`.

In Clojure, immutability is the default. In my opinion that's the steepest learning curve of the language, not the syntax. **If you spent your life mutating variables to get stuff done, programming without mutation is like eating soup with a fork.** But once you're in, you desperately don't want to go back.

### 3. The big render

The main render function connects the smaller parts into a whole. As you can see below, most of the differences have already been covered. Please review for yourself the pros and cons of each approach:

**JSX render function**
{% highlight js %}
render() {
    let status = "Next player: ${this.state.xIsNext ? 'X' : 'O'}";
    return (
      <div>
        <div className="status">{status}</div>
        <div className="board-row"> {[0,1,2].map ((i) => this.renderSquare(i))}  </div>
        <div className="board-row"> {[3,4,5].map ((i) => this.renderSquare(i))}  </div>
        <div className="board-row"> {[6,7,8].map ((i) => this.renderSquare(i))}  </div>
        <button onClick= {() => this.reset()}> Reset </button>
      </div>
    );
  }
{% endhighlight %}

**Reagent render function**
{% highlight clojure %}
(defn render [state]
  (let [square (partial square state)]
    [:div
     [:div.status "Next player " (if (:x-is-next @state) "X" "O")]
     [:div.board-row (doall (map square [0 1 2]))]
     [:div.board-row (doall (map square [3 4 5]))]
     [:div.board-row (doall (map square [6 7 8]))]
     [:button {:on-click #(reset! state (vanilla-state))} "Reset game!"]]))
{% endhighlight %}

### Conclusion: Code vs Data

Hopefully I managed to show you a few things that Reagent brings to the table:
* Concise and compact
* Creating functional style components is super easy
* Immutable data is the default

But the thing that fundamentally separates it from React/JSX is the **data focus**.
* JSX creates instructions: `React.createElement('div')`.
* Reagent creates data structures: `[:div]`.

The former is opaque, hard to inspect at runtime. The latter is highly transparent, and easily inspectable in more than one way.

Your app is declared using nothing but pure data, using nothing but plain functions to manipulate the data. The very rich Clojure standard library with functions like `map`, `filter` and `reduce` at your disposal, without any funky new syntax to learn.

[pure function]: https://en.wikipedia.org/wiki/Pure_function
[tutorial]: https://facebook.github.io/react/tutorial/tutorial.html
[reagent]: https://reagent-project.github.io/
[reddit]: https://www.reddit.com/r/Clojure/comments/5lkkwb/is_anyone_in_the_clojure_community_using_plain/
