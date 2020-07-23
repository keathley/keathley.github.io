---
layout: post
title: "Telemetry Conventions"
date:   2020-07-20 10:00:00
categories: elixir telemetry
---

I’m a big fan of telemetry. It’s arguably the most important elixir
project released in the past few years. Most of the mainstream libraries
have started to adopt it, and that’s a good thing. But, there’s still
a lot of inconsistency in how telemetry is used across projects. I thought
it would be good to write up some of the conventions that I've been using.

I'm treating this as a living document. I expect that things may change
and I'll try to capture those changes here.

## Keep your names consistent

You should choose event names and stick to them. Do not allow users to
customize the event names or change them based on the module that is using
them. `[:my_lib, :call, :start]` is consistent and makes building tooling
really simple. Don’t do things like `[:<users_repo_name>, :call, :start]`.
Breaking consistency in this way makes it harder to build
consistent monitoring and tracing tools around your library.

Stick to a basic naming scheme: `[:app_name, :function_call, ...]` and
you’ll be good to go.

If you need to differentiate between multiple instances of your library
you should include relevant information in the event's metadata. I had to
do this for [Regulator](https://github.com/keathley/regulator). When we
execute telemetry events, the name of each regulator is provided in the
metadata.

## Use spans

The telemetry events you produce should be usable in several contexts. One
of the best ways to do this is to use "spans". The notion of a span is
straightforward. When you start a function call you execute a `[:app,
function, :start]` event. When you finish the function call you execute
`[:app, :function, :stop]` event. If something inside the function call
raises or throws you execute an `[:app, :function, :exception]` event.

These 3 events will cover at least 90% of your user's needs. If your
consumer wants to support APM or tracing they can do that by listening to
all events. If they just want to emit time series, they only need to
listen to the stop and exception events. 

There are times when spans won’t be enough, and when that happens feel
free to execute a one-off event. Otherwise, just use spans.

## Add errors to your stop events

Sometimes things go poorly inside of a function call that doesn’t lead
to an exception. When this happens you should include the error to your
events metadata inside an optional `:error` key. Consumers can use this to
add labels to their time-series or add errors to their traces.

## Give me all your metadata

You don't know what people are going to do with the events that you're
executing. You want to support as many use cases as you can (including all
of the use cases you haven't thought of yet). So lean towards providing
more metadata in your events than you think you need to.

## Allow users to add more metadata

Speaking of metadata, its totally reasonable for you to allow users to add
additional information to an events metadata. This is often useful for users who
want to add additional context or business related metrics to each event.

## Don't rely on middleware to emit your events

A lot of libraries use middleware to emit telemetry events. I have some feedback on this pattern:

Stop it.

Unless there's no other way to provide telemetry, you should be executing
your events from your core library code. The only reasons to use
middleware would be for users to opt-in to telemetry or for customization
of your telemetry events. But, telemetry is _already_ opt-in. Users have
to attach handlers to your events and executing an event with no handler is low-cost.

Allowing users to customize the events isn't something you should do,
as we've already discussed.

A major problem with emiting telemetry in middleware is that it necessarily means
that you aren't getting the full trace. You won't be capturing the time
between your library being called and your library calling the telemetry middleware.
This problem gets worse if the user happens to place the middleware after other, potentially expensive, middleware.

It's less precise and only serves to make your users lives more complicated. If
you have no other way to provide telemetry from your library, by all means, use middleware. But otherwise, avoid it.

## Durations should be in native units (or explicitly stated)

You should default to native units for all of your duration measurements.
If you *really* don’t want to use native units, then return a tuple
stating exactly what units you’re using like: `{100, :microseconds}`.

## Use a single module for telemetry and include all of your context and docs in that module

All of my projects include a module called `Lib.Telemetry`, and they all follow the same pattern:

```elixir
defmodule LibName.Telemetry do
  @moduledoc """
  Description of all events
  """

  @doc false
  def start(name, meta, measurements \\ %{}) do
    time = System.monotonic_time()
    measures = Map.put(measurements, :system_time, time)
    :telemetry.execute([:app_name, name, :start], measures, meta)
    time
  end

  @doc false
  def stop(name, start_time, meta, measurements \\ %{}) do
    end_time = System.monotonic_time()
    measurements = Map.merge(measurements, %{duration: end_time - start_time})

    :telemetry.execute(
      [:app_name, name, :stop],
      measurements,
      meta
    )
  end

  @doc false
  def exception(event, start_time, kind, reason, stack, meta \\ %{}, extra_measurements \\ %{}) do
    end_time = System.monotonic_time()
    measurements = Map.merge(extra_measurements, %{duration: end_time - start_time})

    meta =
      meta
      |> Map.put(:kind, kind)
      |> Map.put(:error, reason)
      |> Map.put(:stacktrace, stack)

    :telemetry.execute([:app_name, event, :exception], measurements, meta)
  end

  @doc false
  def event(name, metrics, meta) do
    :telemetry.execute([:app_name, name], metrics, meta)
  end
end
```

This keeps all of my other code relatively easy to read and provides
a module where I can add docs for all of the events I’m going to emit.

Speaking of docs, you need to write some. For each event, explain what
measurements you’re going to return, what metadata you’re going to return,
and in what context the specific event is going to be executed.

## Test your events

Your telemetry is an API and breaking it is probably more costly than if
you break some sort of functional interface. At least if you break your
functions the user of your library is likely to notice it before they
deploy to production. If you make backward-incompatible changes to your
telemetry events, the user probably has no clue and won't discover it until
they've deployed to production and realize that their monitors and dashboards
are now broken.

Luckily, it’s pretty straightforward to test your telemetry events. I typically
do something like this (which iirc. is a pattern I stole from Redix’s test suite).

```elixir
test "telemetry events" do
  {test_name, _arity} = __ENV__.function
  parent = self()
  ref = make_ref()

  handler = fn event, measurements, meta, _config ->
    assert event == [:your_app, :name, :start]
    assert is_integer(measurements.system_time)
    send(parent, {ref, :start})
  end

  :telemetry.attach_many(to_string(test_name),
    [
      [:your_app, :name, :start],
    ],
    handler,
    nil
  )

  # some function call...

  assert_receive {^ref, :start}
  assert_receive {^ref, :stop}
end
```

## Conclusion

I hope that this provides a good framework for anyone who wants to add
telemetry to their libraries or applications. As library authors, we need to
view good telemetry the same way we view good docs or good tests. These things
matter and they can dramatically enhance the experience of using your library.
Hopefully, we can spread these ideas across the ecosystem.

