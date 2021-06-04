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

When we were using pipes, each function was required to handle the result of the previous function. By forcing each function to deal with results we made them less re-usable. On top of that, we were coupling them to the order that they were called in the pipeline.

Another problem with the pipeline approach is that it tends to assume that errors can be handled in a generic way. But, this assumption is often incorrect.
When dealing with side-effects, the only function that has enough information to decide what to do with an error is the calling function. In many systems, the error cases are just as important - if not more important - then the "happy path" case. The error cases are where your going to have to perform fallbacks or graceful degredation.

If your in a situation where errors are a key part of your functions control flow, then its best to keep all of the error handling in the calling function using `case` statements.

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

This increases the size of the calling function, but the benefit is that you can read this entire function and understand how its handling both good and bad situations.

## Don't pipe into case statements

I used to be on the fence about piping into case statements, but I've seen this pattern abused too many times and in too many ways. Seriously y'all, put down the pipe operator and show a little restraint. If you find yourself piping into case, its almost always better to assign intermediate steps to a variable instead

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

changeset = build_post(attrs)

case store_post(changeset) do
  {:ok, post} ->
    # ...

  {:error, _} ->
    # ...
end
```

## Don't hide higher-order functions

Higher-order functions are great, so try not to hide them away. If you're working with collections, you should prefer to write functions that operate on a single entity rather than the collection itself. Then you can use higher order functions directly in your pipeline.

```elixir
# Don't do...
def main do
  collection
  |> parse_items
  |> add_items
end

def parse_items(list) do
  Enum.map(list, &String.to_ingeter/1)
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

defp parse_item(item), do: String.to_integer(item)

defp add_item(num, acc), do: num + acc
```

With this change our `parse_item` and `add_item` functions become re-usable in a wider set of contexts. These functions can now be used on a single item or can be lifted into the context of `Stream`, `Enum`, `Task`, or any number of other uses. Hiding this logic away from the caller is a worse design because it couples the function to its callsite and makes it less re-usable. Ideally our APIs and the functions that make up those APIs are re-usable in a wide range of contexts.

Another benefit of this change is that better solutions may reveal themselves. In this case, we may decide that we don't need the named functions at all and can use anonymous functions instead. We realize that we don't need the `reduce` and can use `sum` instead.

```elixir
def main do
  collection
  |> Enum.map(&String.to_integer/1)
  |> Enum.sum()
end
```

This final step may not always be the right choice. It depends on how much work your functions are doing. But, as a general rule, you should strive to eliminate functions that only have a single call site. Even though there's no dedicated names for these functions, the final version is no less "readable" then when we started. An Elixir programmer can still look at this series of steps and understand that the goal here is to convert a collection of strings into integers and then sum those integers. And, they can understand this without needing to read any other functions along the way.

## Avoid `else` in `with` blocks

`else` can be useful if you need to perform an operation that is generic across *all* error values being returned. You should not use `else` to handle all potential errors (or even a large number of errors).

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

For the same reason, under no circumstances should you annotate your function calls with a name just so you can differentiate between them.

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

If you find yourself doing this it means that the error conditions matter. Which means that you don't want `with` at all. You want `case`.

`with` is best used when you can fall through at any point without worrying about the specific error or contrary pattern. A good way to create a more unified way to deal with errors is to build a common error type like so:

```elixir
defmodule MyApp.Error do
  defexception [:code, :msg, :meta]

  def new(code, msg, meta) when is_binary(msg) do
    %__MODULE__{code: code, msg: msg, meta: Map.new(meta)}
  end

  def not_found(msg, meta \\ %{}) do
    new(:not_found, msg, meta)
  end

  def internal(msg, meta \\ %{}) do
    new(:internal, msg, meta)
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
    {:error, Error.internal("could not decode: #{inspect resp}")}
  end
end
```

This method provides you a struct that you can use to render various error messages from a phoenix controller or RPC handler or anywhere else that you care about. Because the struct your using is an exception, the caller can also choose to raise the error and you'll get nice error messages.

```elixir
case main() do
  {:ok, _} -> :ok
  {:error, e} -> raise e
end
```

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
