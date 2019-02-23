+++
author = "Stephan Behnke"
categories = ["go"]
date = 2016-08-07T15:43:23Z
description = ""
draft = false
slug = "the-curious-case-of-the-merciless-compiler"
tags = ["go"]
title = "The Curious Case of the Merciless Compiler"

+++

In the movie [2001: A Space Odyssey](http://www.imdb.com/title/tt0062622/) the computer program HAL 9000 goes rogue, showing no mercy towards the space ship's crew. That's exactly how newcomers to the [Go programming language](https://en.wikipedia.org/wiki/Go_(programming_language)) must feel.

Since its introduction in 2009, the language has produced gigabytes worth of online debate about its very opinionated philosophy. Since it is a statically-compiled language, it has a compiler. And just like HAL the Go compiler is quite stubborn and - dare I say - *merciless*.

Take the case of [this poor developer](http://stackoverflow.com/questions/19560334) who cannot believe the Go compiler could be so cruel:

> TBH, it is the most stupid thing I have ever seen in a programming language

Odd, I thought [the invention of the null reference](https://www.quora.com/What-is-the-worst-mistake-ever-made-in-computer-science-and-programming-that-proved-to-be-painful-for-programmers-for-years) is the clear winner here. Anyway, what brought that guy to declare that the pinnacle of stupidity had finally been reached? **The Go compiler will not let you run code with unused imports.**

![](/images/2016/08/hal.jpg)

At this point, you may wonder why the compiler is so adamant about that? As far as I know, it's unique in that regard. Why, for example, doesn't it just show a warning in this case?

Let's ask the people who ought to know: the language creators. Among them, you will find two living legends of computer science: Rob Pike and Ken Thompson. Both were involved in the creation of Unix. Let's look at their official best practice guide [Effective Go](https://golang.org/doc/effective_go.html#blank_unused) for answers:

> It is an error to import a package [...] without using it. Unused imports bloat the program and slow compilation.

Fair enough. Nobody likes slow compilers. You will find exuberant praise online how incredibly fast Go's compiler is. Its workflow feels almost like in a dynamically-typed language with a lightning-fast feedback loop.

This is no accident. 

Actually, it is one of the language's core design principles. Not allowing unused imports is just one piece of the puzzle. To get a glimpse of where it all originates, examine the following excerpt from an [interview with Rob Pike](http://www.informit.com/articles/article.aspx?p=1623555):

> The **starting point was long compile times** —for some of our big software at Google, build times can be unreasonably long, even with our large distributed compilation clusters. The dependency management (or lack thereof) in C and C++ results in far too much code going through the compiler. **You might say that Go was conceived while waiting for a big compilation.**

We've all seen countless memes about [what developers do while waiting for the compiler](https://xkcd.com/303/) - but this one certainly takes the cake. With that back story, you can understand why Go was built from the ground up to compile as fast as possible. And in order to get there, you have to make sacrifices. By not allowing *waste* like unused imports through the compiler pipeline, the process is much more efficient. But to an outsider, only seeing the tip of the iceberg, this may look like "the most stupid thing in a programming language".

But the iceberg goes much deeper.

The pursuit for the fastest compiler possible produced another merciless decision that heavily influences every application's design: **not allowing circular package dependencies.** When package `A` depends on package `B` and vice versa, we have a circular package dependency, or import cycle. For that, Go will throw an error at you faster than you can say 'cycle'. 

From a design perspective, a cycle on package-level is bad news. It tightly couples everything on that cycle, making it harder to change and reason about. Interestingly, many other programming languages are pretty tolerant about import cycles. In Java, for example, you might not even notice that there are cycles in your code at all. Then why is Go so unrelenting?

Compiler speed, again. Go uses a [one-pass compiler](https://en.wikipedia.org/wiki/One-pass_compiler) which processes each source file only once. Therefore it's not as sophisticated as for example Java's [multi-pass compiler](https://en.wikipedia.org/wiki/Multi-pass_compiler) - but mighty fast. This decision also *fundamentally* influenced Go's design. It forced the language to be simple: the language specification can easily be read in less than a day, while you'd need an ancient redwood tree to produce enough paper for Java's specification.

Technically, we have not yet answered why Go cannot just use warnings during development. Luckily, the [Go FAQ](https://golang.org/doc/faq) does:

>  Some have asked for a compiler option to turn those checks off or at least reduce them to warnings. Such an option has not been added, though, because compiler options should not affect the semantics of the language and because the Go compiler does not report warnings, only errors that prevent compilation.

After all you've read, this should come as no surprise. Turning those errors into warnings would severely undermine the core philosophy of the language with its focus on speed and simplicity. Therefore, it's only right to deny any such requests.

At this point it's worth mentioning that there are easy ways around this issue during development. The idiomatic way is to use the underscore character for the import name:

```
import _ "fmt"
```

This way, it will simply be ignored by the compiler. But an even smarter way had emerged: the (now official) command tool `goimports` automatically adds missing imports and removes unused ones. You can configure your editor of choice to run it after you save a file. As you would expect by now, it is lightning fast. It takes the developer out of the loop, solving an issue from one tool with another one. I cannot help but smile at the irony yet the brilliance of that.

As we have seen, the Go compiler might be considered merciless. At least HAL 9000 had a gentle voice to soften the blow! But as you have seen, it has the very best intentions. It is a small price to pay for the simplicity and speed you get in return. And ultimately, your code and workflow are all the better for it.
