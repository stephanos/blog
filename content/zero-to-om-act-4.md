+++
author = "Stephan Behnke"
categories = ["go", "om", "react.js", "javascript", "client-side", "cursors", "core.async", "todomvc"]
date = 2014-09-18T08:57:00Z
description = ""
draft = false
slug = "zero-to-om-act-4"
tags = ["go", "om", "react.js", "javascript", "client-side", "cursors", "core.async", "todomvc"]
title = "Zero to Om - Act 4"

+++

Until now the application is rather boring. It just displays data. But we want to actually use it! In this post we will take a look at how the app reacts (no pun intented) to user input. The source code can be found on [GitHub](https://github.com/stephanos/om-tutorial).

**Note:** I strongly recommend reading the [previous post](/zero-to-om-act-3/) first if you haven't done so already.


<hr/>


Managing state is tricky. Each framework has its own mechanisms to detect and handle state changes. Here is Om's.


## Inner State

In this section we will look at how state inside the component can be managed.

### Component State

In the previous post we've already learned that components can have inner, transient state. In our example the `todo-item` component used the symbol `:edit-text` to assign the current text field's value. This snippet should refresh your memory:

```language-clojure	
(defn todo-item [todo owner]
  (reify
    om/IInitState
    (init-state [_]
      {:edit-text (:title todo)})

    om/IRenderState
    (render-state [_ state]
      ...
        (dom/input
          #js {:className "edit"
               :value (om/get-state owner :edit-text)
               :onChange #(change % todo owner)
               ...}))))))
```

Now, let's have a look at the event handler for `onChange`:

```language-clojure	
(defn change [e todo owner]
  (om/set-state! owner :edit-text (-> e .-target .-value)))
```

The handler changes the `:edit-text` state to the input field's value. `->` is a macro which allows writing the statement without additional nesting (i.e. `(.-checked (.-target e))`) but on a "thread". The snippet above shows one of two options to update a component's inner state:

  - `om/set-state!`: sets a new value
  - `om/update-state!`: applies an update function to the current value

Both will trigger an Om re-rendering.

### Application State

We've learned earlier that the entire application state is kept in a single atom, the *root atom*. But we only pass a subset to a component (e.g. `todo` to `todo-item`). How do we keep the subsets in-sync with the root state?

With *Cursors*! From Om's documentation: 

> Cursors split one big mutable atom into smaller sub-atoms that remain in-sync with the state held in the root atom. 

Basically you can imagine a cursor a bit like a pointer to a portion of the root state. A component can apply changes to its cursor and the application state will change. When data in the application state changes - for example from an HTTP request - Om will re-render all components that depend on the changed value. Therefore the binding is two-way.

To read values from a cursor you'll need to dereference it (eg `@my-cursor`) - except during the render phase (ie inside `render` and `render-state`) where it can be treated just like a regular value. To change a cursor value you have two options:

  - `om/update!`: sets a new value
  - `om/transact!`: applies an update function to the current value

Let's see how this works in our app.

The `todo-item` component contains a checkbox field which toggles the item's status. Its `onChange` event is handled here:

```language-clojure	
(defn complete [todo]
  (om/transact! todo :completed #(not %)))
```

The `transact!` function receives (1) the component's cursor `todo` to the todo item, (2) the key `:completed` to specify which part of the data to change and (3) a function literal that simply negates the current value.

Another example is in the main `todo-app` component. It also contains a checkbox but this one toggles the status of all items:

```language-clojure	
(defn toggle-all [e state]
  (let [checked (-> e .-target .-checked)]
    (om/transact! state :todos
	  (fn [todos] (vec (map #(assoc % :completed checked) todos))))))
```

Here the value of the checkbox binds to `checked`. The `transact!` update function uses `map` to return a sequence of todo items with the `completed` key set to `checked`. Don't forget: we are working with immutable data so both `map` and `assoc` do not modify the data but return new values.

**Note:** With `vec` we make sure that the return value is always a vector because Om's cursors only work with associate data structures (ie ClojureScript maps and indexed sequential data structures, such as the vector).

To learn more about cursors I suggest reading [Om's Cursor docs](https://github.com/swannodette/om/wiki/Cursors).


## Outer State

Often the state we'd like to manage will not be within direct reach of the component. The two mechanisms we've learned above will not be enough then.

### Parent Components

Sometimes it is not feasible to change the application state since a sub-component often only has a cursor to a part of the state. Then one of its parents should be responsible. But how do they communicate with another?

Usually, in JavaScript this is handled by callbacks or event bubbling. But in ClojureScript we have something even better: queue-like channels. The [Clojure blog](http://clojure.com/blog/2013/06/28/clojure-core-async-channels.html) describes it perfectly:

 > A key characteristic of channels is that they are blocking. In the most primitive form, an unbuffered channel acts as a rendezvous, any reader will await a writer and vice-versa.

Basically, this allows us to easily synchronize two or more asynchronous actions - while writing sequential code. It all goes back to [Tony Hoare's Communicating Sequential Processes (CSP)](http://en.wikipedia.org/wiki/Communicating_sequential_processes) from the 1970s. It recently gained popularity through an implementation in the [Go programming language](http://golang.org/).

Channels are provided by the library `core.async`, made possible only by the power of macros. To get started, we need to load the library at the beginning of a file:

```language-clojure	
(ns todomvc.app
  (:require-macros [cljs.core.async.macros :refer [go]])
  (:require        [cljs.core.async        :refer [put! <! chan]])
```

The `todo-app` component uses the `will-mount` lifecycle function to create a new channel once the app is started:

```language-clojure	
(defn todo-app [{:keys [todos] :as state} owner]
  (reify
    om/IWillMount
    (will-mount [_]
      (let [comm (chan)]
        (om/set-state! owner :comm comm)
        (go (while true
              (let [[type value] (<! comm)]
                (handle-event type state value))))))
```

`(chan)` returns a new unbuffered channel. This is then added to the component's state with the key `:comm`. Finally, a `go` block is opened: inside we find an endless loop that reads a value from the channel with `(<! comm)` and destructures it into a vector with two items, `[type value]`. 

It is important to note that reading from the channel is a blocking operation, waiting for data being sent through the channel forever. But wouldn't the app completely freeze since JavaScript is single-threaded? Well, that's why we need to wrap all `<!` calls with `go`: it is a macro that allows to have small, contained event loops within the overall global event loop. The inner ones block while the global one keeps running.

**Note:** The `comm` channel is now passed to every function or component that needs it. To avoid confusion: all code snippets from the previous post omitted this.

Back to our example. The data that was read from the channel is passed to `handle-event`:

```language-clojure	
(defn handle-event [type state val]
  (case type
	:destroy (destroy-todo state val)
	:edit    (edit-todo state val)
	:save    (save-todos state)
	:cancel  (cancel-action state)
	nil))
```

The function uses `case` to select a form to execute based on the value of `type`. It handles all the events that the `todo-item` component might want to send. Let's have a look at one of its event handlers that fires when you click the red 'x' on a todo item:

```language-clojure	
(defn destroy [todo comm]
  (put! comm [:destroy @todo]))
```

It uses `put!` to send the vector `[:destroy @todo]` over the channel. Notice the `@` that dereferences the cursor beforehand - so the value is sent, not the cursor. It is then read by the endless loop we saw above, passed to the `handle-event` and finally ends up as a parameter to `destroy-todo`:

```language-clojure	
(defn destroy-todo [state {:keys [id]}]
  (om/transact! state :todos
	(fn [todos] (vec (remove #(= (:id %) id) todos)))))
```

Here we see that the state cursor and the todo's id are used to update the application state. The todos are replaced by a copy of all todos, excluding any item that has the passed-in id. Remember, the function literal can also be written as `(fn [todo] (= (:id todo) id))`.

The other functions in `handle-event` basically all work the same way, nothing we haven't seen by now already.

This just scratched the surface of the possibilities of channels. To learn more about channels I suggest reading [ClojureScript core.async todos](http://rigsomelight.com/2013/07/18/clojurescript-core-async-todos.html) and trying out this [source code example](https://github.com/swannodette/hs-async/blob/master/src/hs_async/core.cljs).

### DOM nodes

There comes a time in every app when you need to work with an actual DOM node, eg reading and changing its value. By using the `ref` keyword, a *reference name* can be assigned to a node. Later, for example in an event handler, the node can be accessed by this name. 

In our app we need this for the 'new' text field of the `todo-app` component. Here with the newly added `ref` key:

```language-clojure	
(dom/input
  #js {:id "new-todo" 
       :ref "newField" 
       :onKeyDown #(enter-new-todo % state owner)}
       ...}
  ...
```

With the `ref` in place, the event handler `enter-new-todo` can access the node:

```language-clojure	
(defn enter-new-todo [e state owner]
  (when (== (.-which e) ENTER_KEY)
	(let [new-field      (om/get-node owner "newField")
		  new-field-text (string/trim (.-value new-field))]
	  (when-not (string/blank? new-field-text)
	 	(let [new-todo {:id (guid)
                        :title new-field-text
		 				:completed false}]
			(om/transact! state :todos #(conj % new-todo)))
		(set! (.-value new-field) "")))
	false))
```

When the user hits the enter key it creates a new todo: by using `om/get-node` with the parameters `owner` (the underlying React component) and `newField` (the reference name) the text field (a native browser DOM node) can be retrieved. Calling `.-value` on the node reads its text content. If it is not empty a new item is created and saved to the application state via `om/transact!`. Finally, the input's value is reset.

Another example is the 'edit' text field of the `todo-item` component, now with the added `ref` key:

```language-clojure	
(dom/input
  #js {:ref "editField" :className "edit" ...}
  ...
```

On double-clicking the todo label the `edit` event handler is called:

```language-clojure	
(defn edit [e todo owner comm]
  (let [todo @todo
        node (om/get-node owner "editField")]
    (put! comm [:edit todo])
    (doto owner
      (om/set-state! :needs-focus true)
      (om/set-state! :edit-text (:title todo)))))
```

There's a lot going on! The `todo` symbol binds to the dereferenced cursor and `node` to the input's DOM node. The vector `[:edit todo]` is written to the channel with `put!` (and handled by the `handle-event` function we saw earlier), putting the app into 'editing' mode. At last, the component state is updated to reflect the editing of the todo.

**Note:** `doto` takes `owner` and calls the function of each subsequent form  with it as the first parameter.

If you want to learn more about refs I have just what you need: [More About Refs](http://facebook.github.io/react/docs/more-about-refs.html).


<hr/>


This concludes today's act! We have seen how state is managed in Om. We didn't look into *every* line of the example app but you saw the most important parts. Feel free to browse the [full source code](https://github.com/stephanos/om-tutorial) to explore the rest of it.

In **[Act 5](/zero-to-om-act-5/)** we'll learn how to publish the app properly. Meanwhile, here are a few links to dive deeper into Om:

  - [Om sweet Om](http://blog.getprismatic.com/om-sweet-om-high-functional-frontend-engineering-with-clojurescript-and-react/), describes Om's benefits in a real-life project
  - [Basic Om Tutorial](https://github.com/swannodette/om/wiki/Basic-Tutorial), an example-driven tutorial from David Nolen
  - [Functional UI programming with React.JS and ClojureScript](vimeo.com/97516219), shows how to build an UI for the minesweeper game from scratch
  - [Om's API docs](https://github.com/swannodette/om/wiki/Documentation)