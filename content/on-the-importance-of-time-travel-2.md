+++
author = "Stephan Behnke"
date = 2016-08-08T20:01:07Z
description = ""
draft = true
slug = "on-the-importance-of-time-travel-2"
title = "On the Importance of Time Travel"

+++

You probably had the need to travel through time. I know I did. Time travel is especially difficult in a static world, a world you can not change after it was created. This article shows you why it is important to prepare for time traveling and how to go about it.


### How did it get so late so soon?

There comes a point in every application you build where you need it to make a decision based on *the current time*. In a game you need to show the 'Game Over' screen and play sad music when the timer elapsed, in a calendar app you want to pop up an alert before a meeting and in a smart house you need to turn on the heating before its residents come home at 6pm.

Time is actually an input to your program just like any user input. The user's nickname, last ordered pair of socks or favorite Pokémon are program variables just like the current day of the week, the days left in the month and the minutes until the end of the work day.

But there is a major difference.

Since time is an abstract concept, humanity has developed two tools to get a grasp on it, to actually _measure_ time: the calendar and the clock. And that's where the trouble begins.

For example, if I ask you to tell me the time of day in exactly two weeks, the correctness of your answer depends on the place and time I ask you. That's because if the clock's are adjusted to daylight saving time within those two weeks where you are currently at, the time of the day is actually shifted by one hour. That means you can't use simple addition and subtraction to figure out the answers to questions like this.

Luckily, most programing environments already come with tools to help you with these calculations. But it stills becomes complicated when you want to make sure your program can deal with all the idiosyncrasy of the clock and the calendar properly.

Now, when faced with great complexity, what is your first instinct as a software developer? If it is to change careers and becoming a gardener, I congratulate you to a much happier and simpler life. For the rest of us, our only hope is to *test* our program for different scenarios and make sure it works. And that's exactly were it usually becomes difficult if you didn't prepare your code properly.


### Time is an illusion

Let's use an exemplary program in pseudo-code to look at how it suffers from the complexity of time and how we can mitigate the risk of bugs.

```
class WerewolfAlarm
  static def is_werewolf_night()
    # some complicated algorithm
    return next_full_moon == new Date()
```

This program is supposed to remind us to take an extra set of silver bullets when we go out at night. But how do we test it?

```
class WerewolfAlarmTest
  def "check for werewolves"
    assertFalse WerewolfAlarm.is_werewolf_night()
```

This is a bad unit test. It's basically impossible to say whether it will succeed or fail since that depends on the day you execute it. How can we improve this?

```
class WerewolfAlarm
  static def is_werewolf_night(today)
    # some complicated algorithm
    return next_full_moon == today
```

Here the current date is provided from outside. This is perfectly fine on a small scale, but can be bothersome later. For example, the additional parameter might cascade through the internal methods of a class which might complicate things. Also, now each client is responsible for providing the current date, depending on the design of the application this might not be desirable. And, in an integration test there would still not be a way to provide a specific date from the test. What now?

```
class WerewolfAlarm
  constructor(clock)
    this.clock = clock

  def is_werewolf_night()
    # some complicated algorithm
    return next_full_moon == clock.today()
```

Now the class receives a `Clock` on construction and uses it to read the current time. It is now completely decoupled from the real world and easily testable.

```
class WerewolfAlarmTest
  def "check for werewolves"
    clock = new Clock()
    alarm = new WerewolfAlarm(clock)

    clock.set(new Date('2016-11-14'))
    assertTrue alarm.is_werewolf_night()

    clock.set(new Date('2016-11-29'))
    assertFalse alarm.is_werewolf_night()
```

This test will strictly test the algorithm and - once it works - will continue to work independently of the current day.

One additional benefit is that a test written this way will be very explicit. In a project I worked on where the time was not mockable, some tests set time attributes of domain objects (e.g. `createdAt` or `openUntil`) relative to the current time with some tricky 'date math'. What then happens is that for a few days a year the tests simply break. That's because the test calculates the hours until something elapses but can't handle the change from/to daylight saving time.


### Time to say goodbye

The implementation looks different for each programming environment. Sometimes you might use a global `Clock` object or a dependency injection framework, but the general idea is still very valuable: decouple your software from the current time and allow it to travel through time.

**tl;dr** Every time you write the equivalent of `new Date` in the programming language of your choice, you make that code very difficult to test for different dates. Pass-in some sort of `Clock` object from outside instead, allowing you to easily manipulate time.