---
layout: post
title: "Teletry Conventions"
date:   2020-07-20 10:00:00
categories: elixir telemetry
---

I’m a big fan of telemetry. It’s arguably the most important elixir
project released in the past few years. Most of the mainstream libraries
have started to adopt it, and that’s a good thing. But, there’s still
a lot of inconsistency in how telemetry is used across projects. I thought
it would be good to write up some of the conventions that we've found lead
to greater usability and flexibility.

## Keep your names consistent

You should chose event names and stick to them. Do not allow users to
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

The telemetry events you produce should be usable in a number of contexts.
One of the best ways to do this is to use the “span” convention. This was
previously just a convention that several libraries used. But its recently
been built into [telemetry
itself](https://github.com/beam-telemetry/telemetry/pull/61). The notion
of a span is pretty straightforward, when you start a call you execute
a `[:app, function, :start]` event. When you finish the call you execute
`[:app, :function, :stop]` event. If raises or throws in the middle of
your span you execute `[:app, :function, :exception]`.

These 3 events will allow you to cover >= 90%  of your users common cases.
If your consumer wants to use APM tracing, they can do that by listening
to all events. If they just want to emit timeseries, they only need to
listen to the stop and exception events. There are times when spans won’t
do enough for you. When that happens feel free to execute a one-off event.
But the majority of your events should be spans since they cover the most
cases.

## Add errors to your stop events

Sometimes things go poorly inside of a function call but it doesn’t lead
to an exception. When this happens you should include the error to your
events metadata inside an optional `:error` key. Consumers can use this to
add labels to their timeseries or add errors to their traces.

## Give me all your metadata

You don't know what people are going to do with the events that you're
executing. You want to support as many use cases as you can (including all
of the use cases you haven't thought of yet). So leans towards providing
more metadata in your events than you think you need to.

## Don't use middleware as the vehicle for your telemetry

Middleware is fine. But you should not build middleware to support telemetry
events. Just execute events from your library code. Making telemetry a middleware
concern means that you may not be capturing the full duration of a trace, users
could incorrectly add middleware after expensive operations, and you have less control
over the metadata being provided. Telemetry events are already opt-in since you
have to intentionally attach to them. Using middleware to manage your telemetry
events only introduces more complexity.

## Durations should be in native units (or explicitly stated)

You should default to native units for all of your durations. If you
*really* don’t want to use native units, then return a tuple stating
exactly what units you’re using like: `{100, :microseconds}`.

## Use a single module for telemetry and include all of your context and docs in that module

All of my projects include a module called `Lib.Telemetry`. They all follow the same pattern:

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
functions the user of your library is likely to notice it. If you make
backwards incompatible changes to your telemetry events the user probably
has no clue. But now their monitors and dashboards don't work. Don't do this.

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
Hopefully we can spread these ideas across the ecosystem.
