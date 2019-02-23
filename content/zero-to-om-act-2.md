+++
author = "Stephan Behnke"
categories = ["go", "om", "react.js", "javascript", "client-side", "todomvc"]
date = 2014-09-04T05:18:00Z
description = ""
draft = false
slug = "zero-to-om-act-2"
tags = ["go", "om", "react.js", "javascript", "client-side", "todomvc"]
title = "Zero to Om - Act 2"

+++

In this second post we will look at actual ClojureScript source code, step by step. The source code can be found on [GitHub](https://github.com/stephanos/om-tutorial).

**Note:** I strongly recommend reading the [previous post](/zero-to-om/) first if you haven't done so already.


<hr/>


## ClojureScript 101

Before we dive in we'll look at the philosophy of ClojureScript.

Since ClojureScript is a Lisp dialect, it strips away most of the syntax you may know from other programming languages. This makes it very simple - but it looks pretty unfamiliar at first.

```language-clojure
(+ 1 1)
```

This expression (also called S-expression, short for *symbolic expression*) has the value `2`. Every expression has a value. The parentheses define its scope. Basically, **everything** in ClojureScript boils down to this simple form:

```language-clojure
(function param1 param2 ... paramN)
```

The first element in the expression is the function position, the others are its parameters. This is called *prefix notation*, in contrast to the *algebraic notation* `1 + 1` from C-like languages.

Of course, expressions can be nested, too: 

```language-clojure
(println (- 10 (* 2 5)))
```

This expression will print `0` since expressions are evaluated from the inside out. But interestingly, it's value is `nil` since `println` only has a side-effect and returns no meaningful value.

Note that `+` and `println` are actually just symbols that evaluate to values, in this case functions. They are similar to constants in this regard.

There are three types of values that can be in the function position:

* a function, like `println` or `+`
* a special form, build-in "primitives" like `if`, `fn` and `def`
* a macro, functions running at *compile-time*

One more thing about macros: they are a very powerful way to extend the language and transform your code. For example, all parameters of a function must be evaluated before it can be executed - but in the case of `(and (fn ...) (fn ...))` we don't want to evaluate the second parameter if the first one is already false! Good thing `and` is a macro: it transforms the code in a way that each parameter is executed lazily and the function returns as early as possible.

But enough about macros. We will now continue by going through the todo application's source code.

## First Steps

Our application consists of three ClojureScript source files: `app.cljs`, `item.cljs` and `utils.cljs`. Let's start easy. In order to get a feeling for ClojureScript we'll look at the smallest file first: `utils.cljs`.

```language-clojure
;; utils.cljs, part #1
(ns todomvc.utils
  (:require [cljs.reader :as reader])
  (:import [goog.ui IdGenerator]))
```  

In the first line, a namespace for the file is specified via `ns`. Every ClojureScript file needs to define a unique namespace, `todomvc.utils` in this case. This prevents name collisions and integrates nicely with the module concept of Google Closure. Note: It's not a coincidence that the namespace matches the file path (`todomvc/utils`) - it's a must. 

In the second line, the external ClojureScript namespace `cljs.reader` is loaded with `:require`. It is named `reader` via the required `:as` and can be referred to by this alias throughout the file. 

In the third line, `:import` loads the module `IdGenerator` from the namespace `goog.ui` of the Google Closure library. The difference to `:require` is that we load specific elements, not namespaces, in order to refer to them by name.

**Note**: Words starting with `:` are keywords. They are like symbols - but just evaluate to themselves instead of an arbitrary value.
 
```language-clojure
;; utils.cljs, part #2
(defn pluralize [n word]
  (if (== n 1)
    word				;; 'true'-block
    (str word "s")))	;; 'false'-block
```

`defn` defines a function called `pluralize` with two parameters: `n` and `word`. In the body, `if` returns the original word when `n` equals 1 - otherwise a new string with the character 's' appended to `word`.

**Note**: `defn` is just a shortcut to define an anonymous function with `fn` and binding it to a symbol with `def`.

```language-clojure
;; utils.cljs, part #3
(defn now []
  (js/Date.))
```

`now` is a function without any parameters. It returns a new instance of JavaScript's `Date` object. Since it is a native JavaScript object the prefix `js/` must be used to access the `js` namespace. 

**Note**: The inconspicuous `.` is responsible for calling the object's constructor, like `new` in JavaScript.

```language-clojure
;; utils.cljs, part #4
(defn guid []
  (.getNextUniqueId (.getInstance IdGenerator)))
```

`guid` uses the Closure library's `IdGenerator` to return a new unique ID. It also shows how to call a method on a JavaScript object: `(.methodName jsObject)`, notice the `.` at the beginning. The expression translates to `IdGenerator.getInstance().getNextUniqueId()` in JavaScript.

```language-clojure
;; utils.cljs, part #5
(defn hidden [is-hidden]
  (if is-hidden
    #js {:display "none"}
    #js {}))
```

`hidden` returns the JavaScript object `{display: "none"}` when the parameter `is-hidden` is true. As you see, to return a native JavaScript object use `#js {...}` the elements are defined in key/value pairs without any special characters like `:`.

**Note**: In ClojureScript neither *camelCase* nor *snake_case* are used, but *dash-separated-lowercase*.

```language-clojure
;; utils.cljs, part #6
(defn store
  ([ns] (store ns nil))	;; 1 parameter
  ([ns edn]				;; 2 parameters
    (if-not (nil? edn)
      (.setItem js/localStorage ns (str edn))
      (let [s (.getItem js/localStorage ns)]
        (if-not (nil? s)
          (reader/read-string s)
          [])))))
```

`store` reads data from and writes data to the browser's local storage. It contains a lot of interesting new things. 

First, it's a multi-arity function. This means it allows multiple numbers of arguments, `[ns]` and `[ns edn]` in this case. It's similar to function overloading in other languages. The common use case is to provide default values for parameters. Here it uses `nil` for the second parameter if not specified otherwise.

Secondly, we see a few new functions. `if-not` is obviously the negated version of `if`. `nil?` (it's best practice to have the '?' suffix at the end of functions with a boolean return type) checks if the provided value equals `nil` (which is identical to `null` in JavaScript). `let` allows binding symbols to values locally, sort of like a confined `def`: It starts with a vector of bindings, pairs of symbol/value, and concludes with a body of expressions. Here it binds `s` to the result of the local storage access.

Thirdly, it serializes and deserializes ClojureScript data structures: The textual result of `(str edn)` is written to the local storage with `.setItem` and reads from local storage with `.getItem` - deserialized with `(reader/read-string s)` from the `reader` namespace loaded earlier.

**Note:** [EDN](https://github.com/edn-format/edn) (*extensible data notation*) is like an extended version of JSON with a few additional benefits like non-string map keys, sets and tags.

<hr/>

That's it for now. In [**Act 3**](/zero-to-om-act-3) we'll look at the Om-specific code in detail.

In the meantime, if you want to learn more about ClojureScript I wholeheartedly recommend the [tutorial for Light Table](https://github.com/swannodette/lt-cljs-tutorial) from David Nolen, the creator of Om.