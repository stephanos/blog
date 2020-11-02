+++
author = "Stephan Behnke"
date = 2020-10-30T00:38:50Z
description = ""
draft = false
slug = "rewire-new-approach-to-dependency-injection-in-elixir"
title = "Rewire: A new Approach to Dependency Injection in Elixir"
+++

I've been working with Elixir for 3 years full-time now and while I think it's an exceptional language and development environment, the testing story always felt incomplete to me. Something was missing. In this post, I'll explain what that is and how I attempted to fix it.

## Injecting Mocks

While I strive to minimize the use of mocks, I find they are still quite useful in certain situations.

Before Elixir, I've mainly worked with Java. For better or worse, in Java you have a multitude of options to inject dependencies into your classes. Most notably, `@Autowired` that allows you to simply override annotated fields during testing with your mock. Could not be simpler.

In Elixir things are a little different. That's because Elixir does not have classes or class fields. Modules are _stateless_. So how does one inject a dependency in Elixir?

Let's look at this simplified example and explore our options:

```elixir
defmodule Conversation do
  def start(), do: English.greet()
end
```

## 🛑 Function Arguments

The easiest approach does not require any libraries: passing-in dependencies using the function arguments.

```elixir
defmodule Conversation do
  def start(lang_mod \\ English), do: lang_mod.greet()
end
```

```elixir
defmodule MyTest do
  use ExUnit.Case, async: false

  test "start/0" do
    defmodule EnglishMock do
      def greet(), do: "g'day"
    end
    assert Conversation.start(EnglishMock) == "g'day"
  end
end
```

While many (including myself) find this to look "odd" at first, it is admittedly easy to do.

However, it comes with quite a few drawbacks:

1) Your application code is now littered with testing concerns.
2) Navigation in your code editor doesn't work as well.
3) Searches for usages of the module are more difficult.
4) The compiler is not able to warn you in case `greet/0` doesn't exist on the `English` module.

## 🛑 Global Override

The Elixir library [mock](https://hex.pm/packages/mock) (wrapping the Erlang library [meck](https://hex.pm/packages/meck) under the hood) allows overriding any module globally.

```elixir
defmodule MyTest do
  use ExUnit.Case, async: false   # not concurrently!

  import Mock

  test "start/0" do
    with_mock English, [greet: fn() -> "g'day" end] do
      assert Conversation.start() == "g'day"
    end
  end
end
```

Here the `English` module is temporarily replaced with a mock that stubs out the `greet` function. So far so good - but it comes with a cost. One of ExUnit's most valuable features is the ability to run tests _concurrently_. However, to stub out modules _globally_ we have to exempt this test module from being run concurrently (notice the `async: false`). This might seem like a small price to pay but if your application grows you might soon find yourself with a slow test suite. This can easily be avoided!

## 🛑 Configuration Lookup

The more or less official mocking library for Elixir is [mox](https://hex.pm/packages/mox).

```elixir
# in test_helper.exs
Mox.defmock(EnglishMock, for: English)
Application.put_env(:myapp, :english, EnglishMock)
```

```elixir
defmodule Conversation do
  def start(), do: english().greet()
  defp english(), do: Application.get(:myapp, :english, English)
end
```

```elixir
defmodule MyTest do
  use ExUnit.Case, async: true  # concurrently!

  import Mox

  test "start/0" do
    stub(English, :greet, fn -> "g'day" end)
    assert Conversation.start() == "g'day"
  end
end
```

`mox` provides a mock that is "injected" into the module under test by doing a lookup in the app's configuration.

The advantage is that the "odd" function parameter is gone, but all of the other issues are still there. But at least it can be run concurrently since the mock is set up _per process_ (and each test module is its own process in ExUnit).

## 🎉 rewire

I wasn't satisfied with any of these options. So I experimented a little with Elixir metaprogramming and the result was [rewire](https://hex.pm/packages/rewire).

It focuses purely on dependency injection and can be used with any mocking library, like `mox`.

```elixir
# in test_helper.exs
Mox.defmock(EnglishMock, for: English)
```

```elixir
defmodule MyTest do
  use ExUnit.Case, async: true  # concurrently!

  import Rewire
  import Mox

  rewire Conversation, English: EnglishMock  # inject!

  test "start/0" do
    stub(EnglishMock, :greet, fn -> "g'day" end)
    assert Conversation.start() == "g'day"
  end
end
```

By following this approach, we keep our production code completely free of testing concerns and the test can still be run concurrently!

## Ehm, But How Does it Work?

`rewire` is a macro, imported via `import Rewire`.

Let's look at what code the macro generated here:

```elixir
defmodule Conversation.R518 do
  def start(), do: EnglishMock.greet()
end

alias Conversation.R518, as: Conversation
```

First, it generates a _copy_ of the original module with the `English` reference _replaced_ by `EnglishMock`. You might also notice that the module name has changed. Since the module might be rewired in multiple places, this is supposed to prevent naming collisions.

Then, it adds an alias to the rewired module under the _original_ name.

You might wonder how it generates a new module from the original one. The library finds the module's source file path by calling `module_info`, parses the code into an AST with `Code.string_to_quoted`, traverses the AST to replace any rewired dependencies using `Macro.traverse` and evaluates the result with `Code.eval_quoted`. Check out the [source code](https://github.com/stephanos/rewire/blob/master/lib/rewire) for details.

## Limitation

As far as I know, the only situation where you cannot use `rewire` to inject your dependencies is when you are dealing with a process that has been started _before_ your test.

Take for example a [Phoenix](https://www.phoenixframework.org/) controller test. Since you'll be writing tests against the server (using `ConnCase`), a dependency in the controller cannot be rewired after the fact.

## La Fin

I hope you enjoyed this blog post. If you have any questions or feedback, please leave a comment. And if you're curious, try out [rewire](https://hex.pm/packages/rewire) yourself.
