---
layout: post
title: Plug Inside Out
comments: true
---

Ever wondered how does [`Plug`] [plug] work? There is a fair amount of code
generation that happens behind the scenes - by way of Elixir macros - to provide
HTTP request handling that is simple to code, yet can be made as complex as it
needs to be. How exactly does the final code look like? What can we infer from
that code about our request handling stack? I tackle these and related questions
in this post.

There are two main modules in [`Plug`] [plug] that help us construct request
handling stack [`Plug.Builder`] [plug_builder] and [`Plug.Router`] [plug_router].
The builder is a module that construct the chain where incoming request is chained
through a series of plugs. Each plug does its job and returns conn (modified or
original) the value returned is passed to the next plug in the chain. The final
product of the builder is the specification of what goes into handling a specific
request. The router is the module that helps with routing requests to the proper
function at the end of chain.

### Builder

Let's create a simple plug using just [`Plug.Builder`] [plug_builder]
{% highlight elixir %}
defmodule MyPlug do
  use Plug.Builder, log_on_halt: true

  plug Plug.Logger
  plug :hello

  def hello(conn, _opts), do: conn
end
{% endhighlight %}

`plug` is a macro defined in [`Plug.Builder`] [plug_builder] module. I would
expect it to generate some code. It does not generate code directly but it causes
[`Plug.Builder`] [plug_builder] to generate code as the result of its use. The
actual macro sets `@plugs` attribute on the context module. In `@before_compile`
phase [`Plug.Builder`] [plug_builder] collects all plugs and generates code for
into the private method `plug_builder_call`. It its `__using__` [`Plug.Builder`]
[plug_builder] also sets generates `init/2` and `call/2` methods so that your
module conforms to [`Plug`] [plug_behaviour] behaviour. The `call/2` generated
method redirects the call to `plug_builder_call`.

One question I have when I see the use of `plug`: can I see the code that
[`Plug.Builder`] [plug_builder] generates on my behalf? Not out of the box but
with a few simple modifications to the builder's code we can output the generated
code to stdout. There is a forked plug that contains the needed modifications
[`here`](https://github.com/elixir-lang/plug/compare/master...ashneyderman:io_puts_plug "Forked Plug").
If we recompile our plug we can see what's generated:

### Code

To see what code gets generated we start `iex`
{% highlight bash %}
16:58:47 alex@alexmac > iex -S mix
{% endhighlight %}

Then we compile the plug like so:
{% highlight elixir %}
iex(1)> c "my_plug.ex"
body: defp(plug_builder_call(conn, _)) do
  case(Plug.Logger.call(conn, :info)) do
    %Plug.Conn{halted: true} = conn ->
      (
        require(Logger)
        _ = Logger.true("MyPlug halted in Plug.Logger.call/2")
      )
      conn
    %Plug.Conn{} = conn ->
      case(hello(conn, [])) do
        %Plug.Conn{halted: true} = conn ->
          (
            require(Logger)
            _ = Logger.true("MyPlug halted in :hello/2")
          )
          conn
        %Plug.Conn{} = conn ->
          conn
        _ ->
          raise("expected hello/2 to return a Plug.Conn, all plugs must receive a connection (conn) and return a connection")
      end
    _ ->
      raise("expected Plug.Logger.call/2 to return a Plug.Conn, all plugs must receive a connection (conn) and return a connection")
  end
end
{% endhighlight %}

So the idea here is to chain your plugs in the following manner:
{% highlight elixir %}
case Plug1.call(conn, opts1) do
  %Plug.Conn{} = conn1 ->
    case Plug2.call(conn1, opts2) do
      %Plug.Conn{} = conn2 ->
        ...
    end
end
{% endhighlight %}

### Chaining

If you ever considered/used AOP (Aspect Oriented Programming) -  specifically
with AspectJ - the style of interception used in Plug is called `before` interception.
I.e. all your custom logic (the logic inside the plug you code) is done before the
chain progresses. The other two types of interception are `after` and `around`.
`after` type is very similar to `before` except the logic is running after the
interceptor chain (therefore result is accessible inside the after interceptor).
`around` interception can run before or after, making it the more general case
of interception since it can morph into `before` or `after` interception.

The common problem with `before` interception is that it is impossible to wrap
the chain and handle the logic that has to do something before and after the
chain below it. For example, writing a plug that times all the requests is not
possible. So is a plug that retries a request. For example, updates to the database
that cause locking issues can often be retried at a later time. Both cases can
be dealt with rather easily with `around` type of interception which is not
available in [`Plug`] [plug].

Are there any facilities in plug that can help to solve the above problems? If we
look at Plug.Conn fields

{% highlight elixir %}
defmodule Plug.Conn do
  defstruct adapter:         {Plug.Conn, nil},
            assigns:         %{},
            before_send:     [],
            body_params:     %Unfetched{aspect: :body_params},
            ...
end
{% endhighlight %}

`before_send` looks suspicious. If we trace its use we notice this little snippet

{% highlight elixir %}
defp run_before_send(%Conn{before_send: before_send} = conn, new) do
  conn = Enum.reduce before_send, %{conn | state: new}, &(&1.(&2))
  if conn.state != new do
    raise ArgumentError, "cannot send/change response from run_before_send callback"
  end
  %{conn | resp_headers: merge_headers(conn.resp_headers, conn.resp_cookies)}
end
{% endhighlight %}

before any response is sent out to the client [`Plug`] [plug] will reduce [`Plug.Conn`]
[plug_conn] over a collection of `before_send` functions. We can hook up a `before_send`
function that accepts and returns modified [`Plug.Conn`] [plug_conn]. Any modifications
to the result sent to the client can be done in a `before_send`
function. This certainly helps with the case of timing the request execution but
it does not help with retries since fundamental plug interface has been defined
to accept [`Plug.Conn`] [plug_conn] and not the next interceptor in the chain
which is what's needed for retries. The solution to this with the plug itself
might be to design a custom [`Plug.Adapters.Cowboy.Handler`] [cowboy_handler].
It will let us retry the logic of the chain but it does not let us control how
deep the retry plug can be positioned in the chain. The details of this solution
cross the bounds of [`Plug`] [plug_behaviour] behavior therefore I am skipping
any further discussion.

In my next post I will describe a design with more flexible chaining.

[plug]: https://hex.pm/packages/plug "Plug"
[plug_conn]: https://github.com/elixir-lang/plug/blob/master/lib/plug/conn.ex "Plug.Conn"
[plug_builder]: https://github.com/elixir-lang/plug/blob/master/lib/plug/builder.ex "Plug.Builder"
[plug_router]: https://github.com/elixir-lang/plug/blob/master/lib/plug/router.ex "Plug.Router"
[plug_behaviour]: https://github.com/elixir-lang/plug/blob/master/lib/plug.ex "Plug"
[cowboy_handler]: https://github.com/elixir-lang/plug/blob/master/lib/plug/adapters/cowboy/handler.ex "Plug.Adapters.Cowboy.Handler"

[cowboy]: https://github.com/ninenines/cowboy "Cowboy"
