---
layout: default
title: Plug Inside Out
---

Ever wondered how does [`Plug`](https://hex.pm/packages/plug 'Plug') work? There
is a fair amount of code generation that happens behind the scenes - by way of
Elixir macros - to provide HTTP request handling that is simple to code, yet can
be made as complex as it needs to be. How exactly does the final code look like?
What can we infer from that code about our request handling stack? I tackle these
and related questions in this post.

There are two main modules in [`Plug`](https://hex.pm/packages/plug 'Plug') that
help us construct request handling stack
[`Plug.Builder`](https://github.com/elixir-lang/plug/blob/master/lib/plug/builder.ex)
and
[`Plug.Router`](https://github.com/elixir-lang/plug/blob/master/lib/plug/router.ex).
The builder is a module that construct the chain where incoming request is passed
through a series of plugs. Each plug does its job and returns conn (modified or
original). The final product of the builder is the specification of what goes into
handling a specific request. The router is the module that helps with routing
requests to the proper function at the end of chain.

### Builder

Let's create a simple plug using just a `Plug.Builder`

{% highlight elixir %}
defmodule MyPlug do
  use Plug.Builder

  plug Plug.Logger
  plug :hello

  def hello(conn, _opts), do: conn

end
{% endhighlight %}

This is how we can run this example:
{% highlight elixir %}
$ iex -S mix
iex> {:ok, _} = Plug.Adapters.Cowboy.http MyPlug, []
{:ok, #PID<...>}
{% endhighlight %}

Now `cowboy` is listening on port 4000 and is ready to accept requests.

### The Code

`plug` is a macro defined in `Plug.Builder` module. I would expect it to generate
some code. It does not generate code directly but it causes `Plug.Builder`
to generate code as the result of its use. The actual macro sets `@plugs` attribute
on the context module. In `@before_compile` phase `Plug.Builder` collects all
plugs and generates the actual code. Plug.Builder also sets up the

One question I had when I saw the use of `plug`: see the code that `Plug.Builder`
generates on my behalf? The answer is not out of the box but with a few simple
modifications to the builder's code we can outputs the generated code to stdout.
There is a forked plug that contains the needed modifications. <link> here. If we
recompile our plug we can see what's generated:

{% highlight elixir %}


{% endhighlight %}


### Chaining


### Router
