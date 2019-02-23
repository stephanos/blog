+++
author = "Stephan Behnke"
categories = ["go", "om", "leiningen", "javascript", "google closure", "cljsbuild", "todomvc"]
date = 2014-09-25T12:51:00Z
description = ""
draft = false
slug = "zero-to-om-act-5"
tags = ["go", "om", "leiningen", "javascript", "google closure", "cljsbuild", "todomvc"]
title = "Zero to Om - Act 5"

+++

In this part we will have a closer look at the app's build configuration and discover what it can do for us. The source code can be found on [GitHub](https://github.com/stephanos/om-tutorial/tree/act-5).

**Note:** I strongly recommend reading the [previous post](/zero-to-om-act-4) first if you haven't done so already.

<hr/>

## The Build Config

The project's build configuration is defined in `project.clj`. It is written in Clojure and looks like this:

```language-clojure
;; project.clj, part 1
(defproject todomvc "0.1.0-SNAPSHOT"

  :jvm-opts ^:replace ["-Xms4g" "-Xmx4g" "-server"]

  :dependencies [[org.clojure/clojure "1.6.0"]
                 [org.clojure/clojurescript "0.0-2280"]
                 [org.clojure/core.async "0.1.267.0-0d7780-alpha"]
				 [secretary "1.2.0"]
                 [om "0.7.1"]]

  :plugins [[lein-cljsbuild "1.0.4-SNAPSHOT"]]
```

Basically, it's just one big map of vectors. The `dependencies` are automatically loaded from Leiningen. `cljsbuild` is a plugin that allows to compile ClojureScript to JavaScript - its configuration is:

```language-clojure
;; project.clj, part 2
:source-paths ["src"]

:cljsbuild { 
  :builds [{:id "dev"
            :source-paths ["src"]
            :compiler {
              :output-to "app.js"
              :output-dir "out"
              :optimizations :none
              :source-map true}}]})
```

Earlier we've learned that to build the application we can use `lein cljsbuild once dev`. The command starts Leiningen and executes the `dev` build profile of the `cljsbuild` exactly once. If we use `auto` instead, it watches the source paths for any changes and triggers a build. Very handy during development.

If you look into the `out` folder in the project's root directory after a build, you'll see that there is a folder for every dependency we've specified. Inside you'll find the library's source files, in ClojureScript and JavaScript. Additionally, the Google Closure library files can be found in the `goog` directory.

**Note:** The `source-map` option tells the plugin to generate a `<file-name>.js.map` file for each compiled file. This allows the browser to show you the corresponding ClojureScript source code instead of an incomprehensible jumble of JavaScript code when there is a runtime error.

## Publishing

There will come a time when you want to publish your application and share it with the world. The `out` folder contains roughly **2 MB of JavaScript**. Not to mention that they are scattered throughout 88 separate files. Sounds like the requirements for a browser benchmark, not like a usable web application!

So we need to reduce the number of files and the total file size. That's where the Google Closure Compiler can help us. Below you'll find a new build profile called `release` which will solve all of our problems:

```language-clojure
:cljsbuild { 
	:builds [{:id "dev"
						...}

					{:id "release"
					 :source-paths ["src"]
					 :compiler {
					 :output-to "out/app.min.js"
					 :optimizations :advanced
					 :elide-asserts true
					 :pretty-print false
					 	:source-map "app.js.map"
				:externs ["src/react-externs.js"]}}]}
```

In contrast to the `dev` profile it outputs a single file, uses advanced optimizations, removes assertions and does not pretty print the JavaScript code. To run this build profile we use `lein cljsbuild once release`. It takes noticeable longer than the development profile but we'll only run this when we actually want to publish the app anyway.

**Note:** The Closure Compiler is the bulldog among minifiers: very aggressive, stopping for no one. That's why you need to tell it what code it should NOT remove by using 'externs files', such as [react-extern.js](https://github.com/swannodette/react-cljs/blob/master/src/react/externs/react.js) in our case.

The following table gives you a feeling for the influence that the different options have on the final result. It shows the file sizes for the different 'modes' the Closure Compiler offers:

<table style="width: 90%; margin: 0 auto;">
  <col width="50%" />
  <col width="25%" />
  <col width="25%" />
  <tr>
    <td>
      <strong>mode</strong>
    </td>
    <td>
      <strong>size</strong>
    </td>
    <td>
      <strong>gzip size</strong>
    </td>
  </tr>
  <tr>
    <td>
		whitespace
    </td>
    <td>
      1343 KB
    </td>
    <td>
      180 KB
    </td>
  </tr>
  <tr>
    <td>
      simple
    </td>
    <td>
      858 KB
    </td>
    <td>
      119 KB
    </td>
  </tr>
  <tr>
    <td style="width: 33%">
      advanced
    </td>
    <td style="width: 33%">
      192 KB
    </td>
    <td style="width: 33%">
      47 KB
    </td>
  </tr>
</table>

<br/>

The advanced mode reduces the file size by roughly 78% compared to the simple mode. The Closure Compiler achieves this minification by making a dead code analysis: removing JavaScript code that won't be used by the app.

So the best result we can expect is 47 KB gzipped. Sounds reasonable. For comparison, without the option `:elide-asserts true` the result for advanced mode grows to 195 KB (48 KB gzipped). Without setting `:pretty-print false` it even grows to 275 KB (53 KB gzipped).

Of course there is one thing we've left out: the footprint of the React library. The minified version is 123 KB (34 KB gzipped). That makes 81 KB gzipped in total. I'll let you decide for yourself whether this would be an acceptable value for *your* app :)


<br/>


**Tip of the Day:** Leiningen allows you to specify an alias for single or multiple commands. This way, we can just create a `publish` command that runs a build using the release profile - and does a cleanup beforehand:

```language-clojure
:aliases {
  "publish" ["do" ["cljsbuild" "clean"] ["cljsbuild" "once" "release"]]
}
```

## Outlook

There is a project called [shadow-build](https://github.com/thheller/shadow-build) that allows you to split the final JavaScript file into multiple files, eg putting the common code (like the standard library) into one file and the rest into separate files. Then you could have separate 'areas' in your application. For example, a user who doesn't go to the admin section won't need to download the source code for it. Currently, getting this functionality into ClojureScript directly is [being discussed](http://dev.clojure.org/display/design/Google+Closure+Modules).


<hr/>


This concludes our journey through the build facilities. Finally, I recommend glancing over the [cljsbuild sample](https://github.com/emezeske/lein-cljsbuild/blob/master/sample.project.clj) once. It documents all the possible options of the ClojureScript plugin. 

In [**Act 6**](/zero-to-om-6) we'll have a look at libraries that help us create better Om applications!