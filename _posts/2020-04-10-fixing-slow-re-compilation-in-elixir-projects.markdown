---
layout: "post"
title: "Fixing recompilation in Elixir projects"
date: "2020-04-10 08:00:00"
categories: Elixir
excerpt: "Slow recompilation means slow feedback loop and distrupted workflow. Let's find out how to fix this."
cover: "/assets/images/slow.jpg"
cover_credits: >
  Photo by <a href="https://unsplash.com/@makariostang?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText" rel="nofollow">Steve Johnson</a> on Unsplash
disqus: false
---

Our goal as programmers is to deliver value by writing code. It should be efficient, maintainable, but most importantly correct.

Correctness can be checked at multiple stages of the development process, but the most immediate feedback we can get is from the compiler, test suite or the REPL.

We rarely write large chunks of code in one go, then test it as a whole, because that means unnecessary risk of wasted effort if it turns out not to work as expected. We’d rather make small, deliberate changes, then quickly test and adjust if needed before going forward.

For that however, our feedback loops must be as short as possible. That’s why we strive for fast test suites and invest time in optimizing the compilation process, because they directly affect our workflow. Slow feedback contributes to lost focus at the very least.

> Consider the rate of feedback as your speed limit. [^1]

When working on Elixir projects, slow recompilation can be especially annoying. Even the smallest of changes can result in recompilation of large part of the codebase.

```bash
$ touch some/project/module.ex
$ mix test

Running tests...
Compiling 791 files (.ex)
```

That's a long wait before test even run! Let’s find why and how to fix this.

## How compiler decides what to recompile

Elixir compiler uses [lexical tracker](https://github.com/elixir-lang/elixir/blob/master/lib/elixir/lib/kernel/lexical_tracker.ex) to track references to modules, function dispatches, usage of aliases, imports and requires in the code, etc. It uses this information to build project modules' dependency graph and ultimately optimize its own work.

When module changes, the compiler analyzes the dependency graph, finds its dependants and marks them as stale. Next, dependants of these are marked as stale as too. The process is repeated until the whole dependency graph is traversed and all the modules stale are identified.

Whether a stale module will be recompiled depends on the type of dependency it has to a module that "made" it stale.

## Understanding module dependencies

Basically, when module `A` uses module `B` in any way, we say it depends on `B`.
Dependencies themselves are transitive. If `A` depends on `B` and `B` depends on `C`, then by implication `A` depends on `C` too.

There are three types of dependencies between Elixir modules.

##### Compile-time dependencies

Such dependecies are created when module `A` uses module `B` at compile time, for example by:

  * requiring module `B`
  * importing functions from `B`
  * using macros from `B`
  * delegating calls to `B` via `defdelegate`
  * implementing behaviour `B`
  * implementing protocol `B`

When `B` changes, `A` must be recompiled too.

##### Runtime dependencies

Runtime dependencies happen when module `A` interacts with `B` only at runtime, e.g. by calling its functions (either by fully qualified name or via an alias).

When `B` changes, `A` does not need to be recompiled.

##### Struct dependencies

This type of dependency is created when `A` uses `%B{}` struct.

`A` needs to be recompiled only when the definition of `%B{}` struct changes, because struct keys are checked at compile time. [^2]

### Strategy for faster recompilation

Actually it’s not about the speed of the compiler, but the amount of work it has to do. The less cross-dependencies in the codebase, the less modules will need to be recompiled after something changes.

The obvious strategy would be to try to reduce compile-time dependencies, but reducing runtime and struct deps is equally important.

Consider following example:

```elixir
defmodule A do
  @answer B.search()
  def get_answer, do: @answer
end

defmodule B do
  def search, do: C.search()
end

defmodule C do
  def search do
    # ...
  end
end
```

`A` has a compile-time dependency on `B` because it calls a function from that module at compile-time (when module attributes are defined).

`B` has only a runtime dependency on `C` because it calls a function from that module at runtime.

`A` doesn’t have a direct compile-time dependency on `C`, but if `C` changes, then `A` **must be recompiled**, even though `B` doesn’t have to!

```bash
$ touch lib/c.ex
$ mix compile --verbose
Compiling 2 files (.ex)
Compiled lib/c.ex
Compiled lib/a.ex
```

The process is as follows:

1. Compiler detects that `C` changed (on disk), marks it as stale and for recompilation.
2. Compiler sees that `B` depends on something stale (`C`), so marks it as stale as well, but doesn’t mark it for recompilation because it’s only a runtime dependency.
3. Compiler sees that `A` depends on something stale (`B`), marks it as stale, but this time it’s a compile-time dependency, so the file is marked for recompilation too.

Clearly `A` has an **implicit** compile dependency on `C`.

It was counter-intuitive to me at first [^3]. It happens because the compiler assumes that `A` *may* be using `C`’s code at compile time indirectly (through `B`).

This fact is going to shape our strategy when trying to avoid recompilation hell in our projects.

### Fixing your project

In large projects, it’s not uncommon to see cycles in the dependency graph. If there happen to be compile-time dependency between member modules of such cycles, any change will trigger a cascade-style recompilation of other modules in that cycle as well as ones depending on them and so on.

That’s why even changes that appear simple on the surface, sometimes get you few hundred files to recompile... and ruined workflow.

I’ve been in such projects myself and unfortunately it’s often not easy to break these cycles. Not without serious refactoring, at least.

So, as always, it pays off to consider architecture upfront, instead of counting on discovering good one by accident.

I personally look forward to using such tools as [boundary](https://github.com/sasa1977/boundary) which makes cross-module dependencies explicit. Umbrella apps can also be helpful in that aspect as they don’t allow cyclic dependencies between individual apps.

Nonetheless, there are some helpful techniques listed below that may apply to any project.

As a prerequisite, get familiar with *xref* tool.

It’s built into `mix` and can help you indentify super-connected modules in your project and modules that have deep subtrees of dependencies [^4]. Simply use `mix help xref` to start.

There's also a short and practical overview of the tool written by Wojtek Mach on [Dashbit's blog](https://dashbit.co/blog/speeding-up-re-compilation-of-elixir-projects).

I'll start with the one I consider most important.

#### Keep you library code clean

There will probably be some well-connected modules in your business layer, like those in `User` or `Account` contexts. In general, business layer code is very likely to contain a lot of cross-module dependencies, even cycles.

Library code on the other hand is supposed to be generic, it should not depend on any module from your business layer. Otherwise it would transfer all such dependencies to any place it’s referenced from.

Here’s an example from actual project:

```elixir
# web/i18n.ex
defmodule MyApp.I18n do
  @moduledoc """
  Internationalization with a gettext-based API.
  """

  use Gettext, otp_app: :my_app

  def set_locale(%User{locale: locale}),
    do: Gettext.put_locale(__MODULE__, locale)

  # …
end
```

Pattern-matching on `%User{}` struct creates a compile-time dependency on `User` module. It didn't seem to be a big deal until we realized that it creates a lot of indirect dependencies throughout the whole project because `I18n` module is very widely-used.

A simple change yielded a huge positive change in recompilation.

```diff
- def set_locale(%User{locale: locale}),
+ def set_locale(%{locale: locale}),
    do: Gettext.put_locale(__MODULE__, locale)
```

#### Don’t import everything

In Phoenix apps, `web/router.ex` builds a super-connected  `MyApp.Router.Helpers` module. When you import it to use route helpers, you indirectly import a lot of its dependencies. To avoid that, alias it instead:

```diff
- import MyApp.Router.Helpers
+ alias MyApp.Router.Helpers, as: Routes
```

The same applies to any other well-connected module.

#### Change defdelegate to proxy functions

`defdelegate` defines functions via metaprogramming at compile time. Simple "proxy" functions would be runtime dependencies instead.

```diff
- defdelegate authorize(conn), to: Auth
+ def authorize(conn), do: Auth.authorize(conn)
```

#### Don’t define module attributes with remote functions

Module attributes are defined at compile-time, if they are set by using remote functions, compile-time dependencies are created. If you don’t need to use module attributes in guards, consider functions instead.

```diff
- @extension_whitelist FileExt.images()
+ defp extension_whitelist, do: FileExt.images()
```

#### Use remote types in typespec

Consider the example:

```elixir
defmodule Hello do
  def say(%User{username: username}), do: "Hello, #{username}"
  def say(%Admin{name: name}), do: "Hello, #{name}"
end
```

`Hello` uses `%User{}` and `%Admin{}` structs,  so we have just struct dependencies, as shown by `xref`.

```bash
$ mix xref graph
lib/hello.ex
├── lib/admin.ex (struct)
└── lib/user.ex (struct)
```

Now let’s add a pretty standard function `@spec` that list these structs as accepted argument types:

```elixir
@spec say(%User{} | %Admin{}) :: binary()
```

Suddenly, we get more strict compile-time deps [^5].

```bash
$ mix xref graph
lib/hello.ex
├── lib/admin.ex (compile)
└── lib/user.ex (compile)
```

In order to fix this, we should rather define remote types and use them instead of structs.

```elixir
defmodule User do
  defstruct [:username]
  @type t() :: %__MODULE__{}
end

defmodule Admin do
  defstruct [:name]
  @type t() :: %__MODULE__{}
end

defmodule Hello do
  @spec say(User.t() | Admin.t()) :: any()
  def say(%User{username: username}), do: "Hello, #{username}"
  def say(%Admin{name: name}), do: "Hello, #{name}"
end
```

## Summary

Sometimes a quick, small change may result in removal of a crucial dependency and break a cycle in your dependency graph, yielding tangible improvements. Other times, some improvements may come at the expense of code readability and understandability, and simply will not be worth it.

Pay attention. The compiler, through slow recompilation, may be signalling problems in your code. It may prompt you to rethink your recent architectural decisions.

Issues are generally easier and cheaper to fix when detected early and recompilation that's slowing down is a plainly visible warning sign.

[^1]: D. Thomas, A. Hunt. (2020). *The Pragmatic Programmer, 20th Anniversary Edition*. Pearson Education, Inc.

[^2]: As of Elixir 1.6. See [Separate tracking structs from compile-time dependencies #6575](https://github.com/elixir-lang/elixir/pull/6575)

[^3]: Until I received explanation from Jason Axelson. See [Implicit compile-time dependencies](https://elixirforum.com/t/implicit-compile-time-dependencies/28988) in Elixir Forum.

[^4]: Technically it’s a graph, not a tree, but *xref* displays it as such.

[^5]: I actually suspect this is a bug and plan to investigate it.
