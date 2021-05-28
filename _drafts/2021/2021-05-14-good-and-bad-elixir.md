---
layout: post
title:  Good and Bad Elixir
date:   2021-05-14 10:00:00
categories: Elixir Patterns
---

I've seen a lot of elixir at this point, both good and bad. I see a lot of the same
patterns that lead to overall worse elixir code and I thought I would document
some of them and the various ways that you can improve your elixir code.

## Map.get/2 and Keyword.get/2 vs. Access

`Map.get/2` and `Keyword.get/2` unnessarily lock you in to using a specific data structure. This means that if you want to change the type of structure you now need to update all of the call sites. Instead of these functions you should prefer using Access:

```elixir
# Don't do
opts = %{foo: :bar}
Map.get(opts, :foo)

# Do
opts[:foo]
```

## Using access with or over `fetch!`

```elixir
opts = %{}

# Don't do...
Map.fetch!(opts, :foo)

# Do...
opts[:foo] || raise ArgumentError, "Expected a `:foo`"
```

## Don't pipe results into the next function

Elixir-ists love that `|>` operator. Sometimes, slightly too much.

```elixir
# Don't do...
def main do
  data
  |> call_service
  |> parse_response
  |> handle_result
end

defp call_service(data) do
# ...
end

defp parse_response({:ok, result}, do: Jason.decode(result)
defp parse_response(error, do: error)

defp handle_result({:ok, decoded}), do: decoded
defp handle_result({:error, error}), do: raise error
```

There are a bunch of issues with this. The first is that each of these functions is now coupled to both the callsite, and the order in which these calls are made. Instead, we should prefer to have re-usable functions. We can achieve this re-usability by handling the various results in the calling function like so:

Instead you should prefer to handle the results in your calling function as this gives the calling function control over how it wants to handle errors and decouples your other functions from the site in which they're called.

```elixir
def main do
  with {:ok, response} <- call_service(data),
       {:ok, decoded} <- parse_response(response) do
    decoded
  end
end
```

Don't be afraid to use nested case statements if you need to If you need to handle each error independently, don't be afraid to use nested case statements for this.

# Don't pipe into case statements

I used to be on the fence about this but I've seen this abused to many times
and in too many ways. Seriously y'all, put down the pipe operator and show a little
restraint.

```elixir
# Don't do...
build_post(attrs)
|> store_post()
|> case do
  {:ok, post} ->
    # ...

  {:error, _} ->
    # ...
end

# Do...

changeset =
  attrs
  |> build_post()

case store_post(changeset) do
  {:ok, post} ->
    # ...

  {:error, _} ->
    # ...
end
```

## Prefer to use higher-order functions

You can build more re-usable interfaces by

```elixir
# Don't do...
def main do
  collection
  |> parse_items
  |> add_items
end

def parse_items(list) do
  Enum.map(list, &Jason.decode!/1)
end

def add_items(list) do
  Enum.reduce(list, 0, & &1 + &2)
end

# Do...
def main do
  collection
  |> Enum.map(&parse_item/1)
  |> Enum.reduce(0, &add_item/2)
end

defp parse_item(item), do: Jason.decode!(item)

defp add_item(num, acc), do: num + acc
```

Once you done this you may realize that you didn't need the extra functions at all

```elixir
def main do
  collection
  |> Enum.map(&Jason.decode!/1)
  |> Enum.reduce(0, & &1 + &2)
end
```

## Use `for` when checking collections in tests

```elixir
# Don't do...
assert Enum.all?(posts, fn post -> match?(%Post{}, post) end)

# Do...
for post <- posts, do: assert match?(%Post{}, post)
```

## Avoid `else` in `with` blocks

`else` can be useful if you need to log errors or perform some operation that
is generic across all error values being returned. You should *not* use `else` to handle all potential errors (or even a large number of errors)

```elixir
# Don't do...
with {:ok, response} <- call_service(data),
     {:ok, decoded}  <- Jason.decode(response),
     {:ok, result}   <- store_in_db(decoded) do
  :ok
else
  {:error, %Jason.Error{}=error} ->
    # Do something with json error
  {:error, %ServiceError{}=error} ->
    # Do something with service error
  {:error, %DBError{}} ->
    # Do something with db error
end
```

Some people _like_ this pattern so much that they take each operation so that they can take different actions based on which call fails:

```elixir
with {:service, {:ok, resp}} <- {:service, call_service(data)},
     {:decode, {:ok, decoded}} <- {:decode, Jason.decode(resp)},
     {:db, {:ok, result}} <- {:db, store_in_db(decoded)} do
  :ok
else
  {:service, {:error, error}} ->
    # Do something with service error
  {:decode, {:error, error}} ->
    # Do something with json error
  {:db, {:error, error}} ->
    # Do something with db error
end
```

Both of these patterns should be avoided. YOu should build your `with` blocks in such a way that its safe to fall through and return any errors that surface. If you want to unify the various errors that can surface in your application you may find it useful to define a common error type like so:

```elixir
defmodule MyApp.Error do
  defexception [:msg, :meta]

  def new(msg, meta \\ %{}) when is_binary(msg) do
    %__MODULE__{msg: msg, meta: Map.new(meta)}
  end
end

def main do
  with {:ok, response} <- call_service(data),
       {:ok, decoded}  <- decode(response),
       {:ok, result}   <- store_in_db(decoded) do
    :ok
  end
end

# We wrap the result of Jason.decode in our own custom error type
defp decode(resp) do
  with {:error, e} <- Jason.decode(resp) do
    {:error, Error.new("could not decode: #{inspect resp}")}
  end
end
```

The `with` construct is a nice solution when used well. But its been my experience
that `with` can also be highly abused and result in code that is even harder to
read than if you were to just write out all of the case statements explicitly.

## State what you want, not what you don't

You should be intentional about what your requiring of your callers and about
what your callers are returning you. Don't bother checking that a value is not `nil` if
what you expect it to be is a string:

```elixir
# Don't do...
def call_service(%{req: req}) when not is_nil(req) do
  # ...
end

# Do...
def call_service(%{req: req}) when is_binary(req) do
  # ...
end
```

The same is true for `case` statements and `if` statements. Be more explicit about
what it is you expect. You'd prefer to raise or crash if you receive arguments
that would violate your expectations.

## Only return error tuples when the caller can do something about it.

If its possible for your API to error, and there's nothing that your API's user can do
to resolve this error, then you should prefer to raise an exception and avoid
making your user deal with any additional error handling logic.

```elixir
```

## Raise exceptions if you receive invalid data.

You should not be afraid of just raising exceptions if a return value or piece of
data has violated your expectations.
If your calling a downstream service, and you've agreed that the downstream service
should always return

## Don't return errors when the error is unrecoverable

```elixir
```

