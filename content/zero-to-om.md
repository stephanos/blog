+++
author = "Stephan Behnke"
categories = ["go", "om", "react.js", "leiningen", "javascript", "client-side", "google closure", "build tool", "todomvc"]
date = 2014-08-25T15:41:00Z
description = ""
draft = false
slug = "zero-to-om"
tags = ["go", "om", "react.js", "leiningen", "javascript", "client-side", "google closure", "build tool", "todomvc"]
title = "Zero to Om"

+++

This is the first part of a series of blog posts about Om. If you don't know anything about Om, don't worry! You'll learn everything step by step. Here are the basics: *Om is a ClojureScript interface to Facebook's React*. If that doesn't ring any bells, again, don't worry!

**NOTE**: I *just* started learning React.js, ClojureScript and Om. So this is my way of trying to learn this myself: By taking you on the ride with me. Basic knowledge about modern web development is required, though.

## The Stage

Learning by doing is fun. We are going to learn Om with the help of a little todo app.

If you are a web developer and haven't heard of [TodoMVC](http://todomvc.com/) yet you were either unemployed or in a coma for the last few years (you didn't miss much except for React.js). It's a "project which offers the same Todo application implemented [...] in most of the popular JavaScript frameworks of today".

This is how it looks:
![](/images/2014/Aug/todos.png)

As the foundation of this series I will use an [implementation of the Todo app](https://github.com/swannodette/todomvc/tree/gh-pages/labs/architecture-examples/om) written by the creator of Om himself, [David Nolen](https://github.com/swannodette). You can [play with the app](http://swannodette.github.io/todomvc/labs/architecture-examples/om/) to see what it can do.

## The Actors

Let's see what technologies we need for our endeavour.

### React.js

![](/images/2014/Aug/logo_og.png)

[React.js](http://facebook.github.io/react/) is a JavaScript library from Facebook for building user interfaces. Think of it as the V in MVC. It keeps data and DOM in-sync and is *very* efficiently at it by using an in-memory (virtual) DOM to batch all updates at once. That's exactly how 3D engines work as well.

### ClojureScript

![](/images/2014/Aug/apple-touch-icon-precomposed-1.png)

[ClojureScript](https://github.com/clojure/ClojureScript) is a Clojure dialect that compiles to JavaScript. It is a dynamically typed, functional programming language with a strong focus on immutable data structures.

### Google Closure Tools

![](/images/2014/Aug/closure.png)

[Google Closure Tools](https://developers.google.com/closure/) consist of a JavaScript library with utilities for rich, cross-browser web applications and a compiler for optimizing JavaScript.

### Light Table

![](/images/2014/Aug/logo.png)

[Light Table](http://www.lighttable.com/) is an open source IDE mainly geared towards Clojure/ClojureScript development. One of its most interesting features is the inline evaluation of code.

### Leiningen

![](/images/2014/Aug/leiningen-full.jpg)

[Leiningen](http://leiningen.org/) is a JVM-based build tool for Clojure. Its [cljsbuild](https://github.com/emezeske/lein-cljsbuild) plugin automates compiling ClojureScript into JavaScript.

### Om

Finally, [Om](https://github.com/swannodette/om) is the magic goo that combines the functional nature of React with the functional language ClojureScript and its immutable data structures, simplifying application architecture and increasing performance significantly in the process.


## Act 1

At first, let's try to run the application. If we checkout my - slightly modified version of the original - [from GitHub](https://github.com/stephanos/om-tutorial) - we'll see a bunch of files:

```
bower_components/
   todomvc-common/
src/
   todomvc/
      app.cljs
      item.cljs
      utils.cljs
bower.json
index.html
project.clj
readme.md
```

`bower.json` is the manifest file for [Bower](http://bower.io/) which manages external resources. In this case it downloaded a stylesheet and an image into `todomvc-common` for us.

`index.html` is the HTML file we open in our browser to showcase our app. So let's go for it! 

And ... **darn**!

```
GET file:///.../out/goog/base.js net::ERR_FILE_NOT_FOUND index.html:15
GET file:///.../app.js net::ERR_FILE_NOT_FOUND index.html:16
Uncaught ReferenceError: goog is not defined 
```

Well, let's look at the file to see what might have gone wrong:

```language-markup
<head>
  [...]
  <link rel="stylesheet" href="bower_components/todomvc-common/base.css">
</head>
<body>
  <section id="todoapp"></section>
  [...]
  <script src="http://fb.me/react-0.11.1.js"></script>
  <script src="out/goog/base.js"></script>
  <script src="app.js"></script>
  <script>goog.require("todomvc.app");</script>
</body>
```

The file loads a bunch of scripts and if you look at our current directory you'll quickly notice: most of them don't exist, yet. We need to build them first!

That's where we need `project.clj`. It contains the project's build configuration for Leiningen. Next, we run:

```language-bash
$ leiningen cljsbuild once dev
Compiling ClojureScript.
Compiling "app.js" from ["src"]...
```

Now we have a new file called `app.js` and a folder `out` with lots of files in it. If we refresh the browser the issues are gone and we gaze upon a functioning Todo app. Splendid!

If we look into `app.js` we'll see something like this:

```language-javascript
goog.addDependency("base.js", ['goog'], []);
goog.addDependency("../cljs/reader.js", ['cljs.reader'], ['goog.string', 'cljs.core']);
goog.addDependency("../om/dom.js", ['om.dom'], ['cljs.core']);
[...]
```

This is Google Closure at work. It features a *module loader*. Every ClojureScript file translates into a Closure module. Each module can depend on numerous other modules. The loader is found in `out/goog/base.js`. All the app's dependencies are defined in `app.js`. To kickstart our app we need to load the main module: `goog.require("todomvc.app")`. And that is exactly what the `index.html` does.


<hr/>


This concludes the first act. 
In [**Act 2**](/zero-to-om-act-2/) we will look at the ClojureScript source code in detail.
