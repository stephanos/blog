+++
author = "Stephan Behnke"
categories = ["go", "om", "react.js", "javascript", "client-side", "todomvc", "schema", "secretary", "sablono", "om-tools"]
date = 2014-11-09T18:00:00Z
description = ""
draft = false
slug = "zero-to-om-6"
tags = ["go", "om", "react.js", "javascript", "client-side", "todomvc", "schema", "secretary", "sablono", "om-tools"]
title = "Zero to Om - Act 6"

+++

Welcome to our next act. Today we're going to meet a few additional libraries that'll help us write great Om applications. Let's get started!

As always, I strongly recommend reading the [previous post](/zero-to-om-act-5) first if you haven't done so already.

<hr/> 


## sablono

In a previous post I showed you how the application's UI is rendered:

```language-clojure
(dom/div nil
  (header)
  (dom/input 
    #js {:id "new-todo" :ref "newField"
         :placeholder "What needs to be done?"
         :onKeyDown #(enter-new-todo % state owner)})
  (listing state)
  (footer state)))))
```

As you can see, a `dom/*` HTML element receives a map of properties: `#js {...}` or `nil` for no properties. This is all a bit awkward. No designer will ever be happy with this. And that's where [sablono](https://github.com/r0man/sablono) can help you turn the code into this:

```language-clojure
(html
  [:div
	(header)
	[:input
		{:id "new-todo" :ref "newField"
		 :placeholder "What needs to be done?"
		 :on-key-down #(enter-new-todo % state owner)}]
	(listing state comm)
	(footer state)])))
```

Ah, much better! Each HTML element is represented by a keyword at the beginning of a vector. Then, an optional map defines its attributes. Everything else must evaluate to another HTML element, and so on. All attributes match ClojureScript's default naming conventions and are automatically converted to the camel-cased version on rendering.

To get started with sablono you just need to include its namespace and macro:

```language-clojure
(ns todomvc.app
  (:require [...]
            [sablono.core :as html :refer-macros [html]])
```

Note that by using sablono the compiled output grows from 192 KB	(47 KB gzipped) to  201 KB (48 KB gzipped).


## secretary

For the routing, meaning the mapping between application state and browser URL, we can use [secretary](https://github.com/gf3/secretary) (great name, right?).

When we look at the namespace declaration of `app.cljs` we can see the various items required for routing:

```language-clojure
(ns todomvc.app
  (:require [...]
            [goog.events :as events]
            [secretary.core :as secretary :include-macros true :refer [defroute]])
  (:import  [goog History]
            [goog.history EventType]))
```

That's quite a lot. Let's see what we have here: 

  * from secretary, a macro called `defroute` and the namespace `secretary.core`
  * from Google Closure, the namespace `goog.events` as well as the elements `History` and `EventType` from `goog.history`

First, let's define the routing rules:

```language-clojure
(defroute "/"        []       (swap! app-state assoc :showing :all))
(defroute "/:filter" [filter] (swap! app-state assoc :showing (keyword filter)))
```

There are two routes. When the first one matches, it sets the `:showing` entry of the application state to `all`. When the second one matches it sets it to the value of `filter`.

But how are the routes wired to the browser? That's where our modules from Google Closure come into play:

```language-clojure
(def history (History.))

(events/listen history EventType.NAVIGATE
  (fn [e] (secretary/dispatch! (.-token e))))

(.setEnabled history true)
```

A new instance of Google Closure's `History` is created, and a new event listener calls secretary's `dispatch!` function with the current history state (the event's `token` field) for each of its navigation events.

**Note**: Since the `setEnabled` method fires an event for the current location immediately, it has to come after the event listener.

The only place where the user can navigate to different URLs is the `footer component`:

```language-clojure
(defn footer [{:keys [todos] :as state}]
  (let [count (count (remove :completed todos))
	    sel (-> (zipmap [:all :active :completed] (repeat ""))
                (assoc (:showing state) "selected"))]
	(html
	  [:footer {:id "footer" :style (hidden (empty? todos))}
      [:span {:id "todo-count"}
	  	[:strong count]
		(str " " (pluralize count "item") " left")]
	  [:ul {:id "filters"}
		[:li [:a {:href "#/" :class (sel :all)} "All"]]
		[:li [:a {:href "#/active" :class (sel :active)} "Active"]]
		[:li [:a {:href "#/completed" :class (sel :completed)} "Completed"]]]])))
```

We can see that there are three possible routing states: `all`, `active` and `completed`. For each state there is a hyperlink in the footer that triggers the corresponding route. Nothing more to it!

By the way, the original Om application from the last posts already included secretary. Compiled to JavaScript it was 192 KB (47 KB gzipped) but without any routing it shrinks to 172 KB (40 KB gzipped).


## om-tools

Prismatic is a very early adopter of Om and with [om-tools](https://github.com/Prismatic/om-tools) they provide a collection of utilities to help eliminate a few annoyances and add extra functionality. To get started you need to include the namespace:

```language-clojure
(ns todomvc.app
  (:require [...]
  	      [om-tools.core :refer-macros [defcomponent]])
```

Here is how the `todo-item` component looked like until now:

```language-clojure
(defn todo-item [todo owner]
  (reify
    om/IInitState
    (init-state [_]
      ...)
    
    om/IRenderState
    (render-state [_ state]
      ...)
```

That's a lot of boilerplate code! om-tools adds a new macro called `defcomponent` that brings simplicity to it:

```language-clojure
(defcomponent todo-item [todo owner]
  (init-state [_]
	...)

  (render-state [_ state]
    ...)
```

It makes component definitions much smaller and easier to read.

Another big feature is the integration of Prismatic's [schema](https://github.com/Prismatic/schema) library which allows declarative data description and validation. I could write an entire blog post about it but luckily [Prismatic already did](http://blog.getprismatic.com/schema-0-2-0-back-with-clojurescript-data-coercion/). So I'll just highlight how this works in our little TodoMVC application. First, we need to include the `schema` library:

```language-clojure
(ns todomvc.app
  (:require [...]
  	      [om-tools.core :refer-macros [defcomponent]])
```

Then we define a schema for a Todo:

```language-clojure
(def Todo
	{:id s/Str
	 :title s/Str
	 :completed s/Bool})
```

It's pretty self-explanatory so far. Now we can add the schema to Om component parameters:

```language-clojure
(defcomponent todo-item [todo :- Todo owner]
   ...
```

The ` :- Todo` connects the parameter to the schema. Now, in theory, when the component is created the parameter is validated. But before that we need to enable the validation:

```language-clojure
(s/with-fn-validation
  (om/root todo-app app-state
	{:target (.getElementById js/document "todoapp")}))
```

Unfortunately, when I tried this it didn't work. I haven't found out the reason for this yet but it may be because the component is created via `build-all`. Not sure. But the [om-tools example](https://github.com/Prismatic/om-tools/tree/master/examples/sliders) actually works, so go on over there to see it in action.

Anyway, to get what we came here for: let's validate the schema explicitly.

```language-clojure
(defcomponent todo-item [todo :- Todo owner]
  (init-state [_]
	(s/validate Todo todo)
```

Now, when we create a new Todo with the field `completd` it results in an error once the `todo-item` component is created:

```
Uncaught Error: Value does not match schema: {:completed missing-required-key, :completd disallowed-key}
```

This means we can catch errors earlier and prevent some of those subtle bugs we all know and love from dynamic languages. All the improvements come at a price, though: the resulting JavaScript code adds up to 264 KB (61 KB gzipped) now.

There are a few other things the library brings to the table - like mixins - so make sure to look into the [documentation](https://github.com/Prismatic/om-tools).

---

That's it for today. You have seen how a few libraries can help you get even more value out of Om.