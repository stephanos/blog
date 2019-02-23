+++
author = "Stephan Behnke"
categories = ["scala", "macros"]
date = 2013-01-28T11:00:00Z
description = ""
draft = false
slug = "scala-macros-use-case-teaching-scala"
tags = ["scala", "macros"]
title = "Scala Macros Use Case: Teaching Scala"

+++

The brand new experimental macro feature in Scala 2.10 can be used in many interesting ways: 

* [writing concise test cases](https://github.com/pniederw/expecty),
* [generating serialization code](http://mandubian.com/2012/11/11/JSON-inception/)
* [inlining methods](https://github.com/paulp/declosurify)

This article shows exciting new possibilities it opens up for teaching Scala. To quote one student: "This is so awesome ... and fun!".

Exercises are a crucial part of the "Introduction to Scala" training course I developed for German-speaking software engineers. Nobody wants to listen to someone else's dialog for 8 hours - they want to get a feeling for the new language, try out the new syntax, play around with new features.

### Designing training exercises

But there is always the problem of making sure that the students are on the right track when solving the exercises. Depending on the size of the course you cannot look over everyone's shoulder all the time.

This is why I decided to leverage unit tests: Students can use pre-written tests to work through the exercises step by step, knowing immediately whether they are doing something wrong. When results are shared and discussed in the end, unpleasant surprises can be avoided.

### Adding macros to it

So, I hear you ask, what does that have to do with macros? Scala macros allow you to make certain tests that were not possible before. Let's have a look.

<script src="https://gist.github.com/4643260.js"></script>

The snippet above illustrates an exercise to teach pattern matching. In the presentation the students listened to earlier, various patterns like the alternative and wildcard pattern were introduced.

The exercise description states: **Rewrite this code in 3 lines or less**. This way, students have to think about how to apply the new features.

Furthermore, they were taught that a <strong>var</strong> should be used sparsely. The unit test is able to check for that as well.
 
So macros allow to test various code styles and metrics. Cool!

### How it works

Behind the scenes, the unit tests use [def macros](http://docs.scala-lang.org/overviews/macros/overview.html) since type macros are only available in an experimental Scala 2.11 branch. First, I thought it may not be possible with this limitation, but thanks to some help from Eugene Burmako - member of the official Scala team - it worked out great.

The method `task` - part of the trait `Testable` that each exercise inherits from - calls a macro. Now it's getting interesting. Since def macros cannot add fields or methods to existing classes I had to create a base class `Task` with all fields I would need later, e.g. to count the number of `vars` the code uses.

<table>
<colgroup>
    <col width="40%">
    <col width="60%">
</colgroup>
<tr>
    <td>
        <script src="https://gist.github.com/4643398.js"></script>
    </td>
    <td>
        <script src="https://gist.github.com/4647618.js"></script>
    </td>
</tr>
</table>

Then the macro would generate a call to the method `registerTask` with an instance of type `Task`. Since the fields are already declared, they can simply be overridden by the macro on instantiation. Et voilà!

The complete macro code is rather ugly and contains a few workarounds but the following snippet shows the small part where the number of `vars` is determined. Additionally, the method to actually overwrite the field can be seen.

<script src="https://gist.github.com/4647666.js"></script>

All in all, the macro worked pretty well in my recent Scala training. The students were quite astonished when the length and style of their solutions were judged. I think it's a great way of making sure the students adhere to the rules and concepts they just learned about; preventing them to just stick to their old Java habits.

I'm looking forward to the possibilities the type macros will bring in 2.11. **Thanks for reading.**