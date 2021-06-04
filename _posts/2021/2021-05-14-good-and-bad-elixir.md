---
layout: post
title:  Good and Bad Elixir
date:   2021-06-02 10:00:00
categories: Elixir Patterns
---

I've seen a lot of elixir at this point, both good and bad. Through all of that code, I've seen similar patterns that tend to lead to worse code. So I thought I would document some of them as well as better alternatives to these patterns.

## Map.get/2 and Keyword.get/2 vs. Access

`Map.get/2` and `Keyword.get/2` lock you into using a specific data structure. This means that if you want to change the type of structure you now need to update all of the call sites. Instead of these functions, you should prefer using Access:

```elixir
# Don't do
opts = %{foo: :bar}
Map.get(opts, :foo)

# Do
opts[:foo]
```

## Don't pipe results into the next function

Side-effecting functions tend to return "results" like `{:ok, term()} | {:error, term()}`. If your dealing with side-effecting functions, don't pipe the results into the next function. It's always better to deal with the results directly using either `case` or `with`.

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

# Do...
def main do
  with {:ok, response} <- call_service(data),
       {:ok, decoded} <- parse_response(response) do
    decoded
  end
end
```

If we try to use pipes then each of our functions needs to handle the result of the previous function. Handling each result, couples our functions to this specific pipeline. These functions are no longer re-usable in a wider context.

It's also the case that you typically care about handling each error in slightly different ways. The only function that knows how to handle errors is the call site, so it makes sense to keep this logic in the call site.

If the error handling is quite involved, don't be afraid of using nested case
statements. It's better to have all of the error handling logic in the calling
function then spread out across multiple locations.

```elixir
# Do...
def main(id) do
  case :fuse.check(:service) do
    :ok ->
      case call_service(id) do
        {:ok, result} ->
          :ok = Cache.put(id, result)
          {:ok, result}

        {:error, error} ->
          :fuse.melt(:service)
          {:error, error}
      end

    :blown ->
      cached = Cache.get(id)

      if cached do
        {:ok, result}
      else
        {:error, error}
      end
  end
end
```

This increases the size of the calling function, but the benefit is that you can read this entire function and understand what it's doing.

## Don't pipe into case statements

I used to be on the fence about this but I've seen this abused too many times and in too many ways. Seriously y'all, put down the pipe operator and show a little restraint. It's better to assign intermediate steps to a variable.

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

## Don't hide higher-order functions

Higher-order functions are great. But try not to hide them away. If you have a call site that needs to work with collections in a pipeline, you should prefer to write functions that operate on a single entity rather than the collection itself.

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

Once you've made this change, you may realize that you didn't need the extra functions at all.

```elixir
def main do
  collection
  |> Enum.map(&Jason.decode!/1)
  |> Enum.reduce(0, & &1 + &2)
end
```

Removing functions that are called in only a single location is a good way to reduce complexity and decrease the surface area of your API. It also allows the reader to see precisely what is happening in your pipeline.

## Avoid `else` in `with` blocks

`else` can be useful if you need to log errors or perform some operation that is generic across *all* error values being returned. You should not use `else` to handle all potential errors (or even a large number of errors)

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

For the same reason, under no circumstances should you annotate your function calls with a name.

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

You should build your `with` blocks in such a way that it's safe to fall through and return any errors that surface. If you want to unify the various errors that can surface in your application, then it's better to define a common error type like so:

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

The `with` construct is a nice solution when you have unified errors. If you don't have unified errors, then you should use case statements.

## State what you want, not what you don't

You should be intentional about what your requiring of your callers and about what your callers are returning you. Don't bother checking that a value is not `nil` if what you expect it to be is a string:

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

The same is true for `case` statements and `if` statements. Be more explicit about what it is you expect. You'd prefer to raise or crash if you receive arguments that would violate your expectations.

## Only return error tuples when the caller can do something about it.

If it's possible for your API to error, and there's nothing that the user of your API can do to resolve this error, then you should prefer to raise an exception and avoid making your user deal with any additional error handling logic.

```elixir
# Don't do...
def get(table \\ __MODULE__, id) do
  # If the table doesn't exist ets will throw an error. Catch that and return
  # an error tuple
  try do
    :ets.lookup(table, id)
  catch
    _, _ ->
      {:error, "Table is not available"}
  end
end

# Do...
def get(table \\ __MODULE__, id) do
  # If the table doesn't exist there's nothing the caller can do
  # about it, so just throw.
  :ets.lookup(table, id)
end
```

## Raise exceptions if you receive invalid data.

You should not be afraid of just raising exceptions if a return value or piece of data has violated your expectations.
If you're calling a downstream service, and you've agreed that the downstream service should always return JSON, and they return you unparseable JSON, you should prefer to raise. Erlang provides us better mechanisms for handling errors.

```elixir
# Don't do...
def main do
  {:ok, resp} = call_service(id)
  case Jason.decode(resp) do
    {:ok, decoded} ->
      decoded

    {:error, e} ->
      # Now what?...
  end
end

# Do...
def main do
  {:ok, resp} = call_service(id)
  decoded = Jason.decode!(resp)
end
```

This allows us to crash the process (which is good) and removes the useless error handling logic from the function

## Use `for` when checking collections in tests

This is a quick one, but it makes your test failures much more useful.

```elixir
# Don't do...
assert Enum.all?(posts, fn post -> %Post{} == post end)

# Do...
for post <- posts, do: assert %Post{} == post
```
