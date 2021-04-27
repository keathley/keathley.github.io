---
layout: post
title:  "Setting the Isolation Level in Ecto Transactions"
date:   2018-04-23 09:33:00
categories: elixir ecto
excerpt: Even in ACID databases you occasionally need to use stronger isolation guarantees.
redirect_from:
  - /elixir/ecto/2018/04/23/setting-isolation-levels-in-ecto-transactions
---

## TL;DR

If you want to set the isolation level in an ecto transaction you can do this:

```elixir
Repo.transaction fn ->
  Repo.query!("set transaction isolation level repeatable read;")

  # The rest of your operations...
end
```

Read on if you wanna learn a little bit more about weak isolation and how I went about testing this problem.

## The Problem

I recently saw <a href="https://twitter.com/kellabyte/status/987055274421665792" target="_blank">this tweet</a> from [@kellabyte](https://twitter.com/kellabyte). The problem being described here is known as a "Lost Update" and is described in a bit more detail [here by Martin Kleppmann](https://github.com/ept/hermitage).

Postgres is succeptible to this by default. This is because postgres (like other ACID databases) uses "weak isolation" by default. There are good reasons for using weaker isolation guarantees mostly relating to performance. But in certain situations you need to use one of the [stronger isolation levels](https://www.postgresql.org/docs/9.1/static/transaction-iso.html).

In postgres the default isolation level is `read committed`. That means if you get an interleaving of operations like so...

```sql
begin; -- T1
begin; -- T2
select used from coupons where code='foo'; -- T1
select used from coupons where code='foo'; -- T2
update coupons set used=true where code='foo'; -- T1
update coupons set used=true where code='foo'; -- T2, BLOCKS
commit; -- T1. This unblocks T2, so T1's update is overwritten
commit; -- T2
```

Both transactions will commit successfully and the update from T1 will be effectively lost. The solution in postgres is to set the transactions isolation level to `repeatable read`. This will cause the second transaction to abort when the first transaction commits. Its an easy enough solution assuming you know where to look.

But, alas, in my day job I don't have the joy of writing sql queries by hand. Typically all of my database access goes through Ecto or [Moebius](https://github.com/robconery/moebius). I wanted to figure out how to solve this problem using my daily tools.

## Test harness

Before I could work on a solution I needed to create a test harness that would reliably interleave the operations from the separate transactions. To achieve these interleavings I created a central coordinator process and had it spawn two "children" processes; one for each transaction. Each child process blocked until it received a message from the coordinator. The code for the entire thing looked like this:


```elixir
defmodule EctoIsolation.Coordinator do
  def race(mod) do
    t1 = start_transaction(mod, :t1)
    t2 = start_transaction(mod, :t2)

    :ok = sync(t1, :select)
    :ok = sync(t2, :select)

    :ok = sync(t1, :update)
    send(t2, :update) # We don't want to block here in case the transaction crashes

    :ok = sync(t1, :commit)
    send(t2, :commit)
  end

  def sync(pid, msg) do
    send(pid, msg)
    receive do
      {^pid, :done} ->
        :ok
    end
  end

  def start_transaction(mod, name) do
    pid = self()
    spawn fn ->
      mod.transaction(fn -> mod.run(pid, name) end)
    end
  end
end

defmodule EctoIsolation.UnsafeTransaction do
  alias EctoIsolation.{
    Coupon,
    Repo,
  }
  import Ecto.Query, only: [from: 2]

  def transaction(f) do
    Repo.transaction(f)
  end

  def run(parent, name) do
    receive do
      :select ->
        :ok
    end
    select()
    send(parent, {self(), :done})

    receive do
      :update ->
        :ok
    end
    update()
    send(parent, {self(), :done})

    receive do
      :commit ->
        :ok
    end
    send(parent, {self(), :done})
  end

  defp select do
    Repo.one(from c in Coupon, select: c.code, where: c.code == "foo")
  end

  defp update do
    query = from c in Coupon,
      update: [set: [used: true]],
      where: c.code == "foo"
    Repo.update_all(query, [])
  end
end
```

With this setup I could run everything right from iex like so:

```elixir
iex(2)> EctoIsolation.Coordinator.race(EctoIsolation.UnsafeTransaction)

10:59:56.328 [debug] QUERY OK db=0.4ms queue=0.2ms
begin []
t1: Selecting.

10:59:56.328 [debug] QUERY OK db=1.1ms queue=0.3ms
begin []
t2: Selecting.

10:59:56.333 [debug] QUERY OK source="coupons" db=4.1ms
SELECT c0."code" FROM "coupons" AS c0 WHERE (c0."code" = 'foo') []

10:59:56.337 [debug] QUERY OK source="coupons" db=3.4ms
SELECT c0."code" FROM "coupons" AS c0 WHERE (c0."code" = 'foo') []
t1 Updating.
t2 Updating.
t1 Committing.

10:59:56.338 [debug] QUERY OK source="coupons" db=0.7ms
UPDATE "coupons" AS c0 SET "used" = TRUE WHERE (c0."code" = 'foo') []
:commit

t2 Committing.
10:59:56.344 [debug] QUERY OK db=5.7ms
commit []

10:59:56.344 [debug] QUERY OK source="coupons" db=5.7ms
UPDATE "coupons" AS c0 SET "used" = TRUE WHERE (c0."code" = 'foo') []

10:59:56.344 [debug] QUERY OK db=0.6ms
commit []
```

As expected both transactions commit successfully, which is not what we want. But now that we have a test we can work on the solution.

## The solution

It took me several tries (and a little [help from twitter](https://twitter.com/elixirlang/status/988306485720608770)) to finally figure out how to get everything set up correctly. I'll spare you all of the embarassing details and just say that in traditional Keathley-fashion I was making things much more complicated then they needed to be. Turns out all you need to do is change...

```elixir
def transaction(f) do
  Repo.transaction(f)
end
```

to...

```elixir
def transaction(f) do
  Repo.transaction fn ->
    Repo.query!("set transaction isolation level repeatable read;")
    f.()
  end
end
```

Now if we run our transaction again we should get something like this:

```elixir
iex(1)> EctoIsolation.Coordinator.race(EctoIsolation.SafeTransaction)

10:20:41.412 [debug] QUERY OK db=0.1ms queue=0.1ms
begin []

10:20:41.412 [debug] QUERY OK db=0.2ms
begin []
t1: Selecting.

10:20:41.417 [debug] QUERY OK db=0.6ms
set transaction isolation level repeatable read; []

10:20:41.417 [debug] QUERY OK db=0.5ms
set transaction isolation level repeatable read; []

10:20:41.431 [debug] QUERY OK source="coupons" db=1.5ms
SELECT c0."code" FROM "coupons" AS c0 WHERE (c0."code" = 'foo') []
t2: Selecting.

10:20:41.438 [debug] QUERY OK source="coupons" db=2.8ms
SELECT c0."code" FROM "coupons" AS c0 WHERE (c0."code" = 'foo') []
t1 Updating.
t1 Committing.

10:20:41.439 [debug] QUERY OK source="coupons" db=0.7ms
UPDATE "coupons" AS c0 SET "used" = TRUE WHERE (c0."code" = 'foo') []
t2 Updating.

10:20:41.446 [debug] QUERY OK db=6.0ms
commit []

10:20:41.456 [debug] QUERY ERROR source="coupons" db=15.7ms
UPDATE "coupons" AS c0 SET "used" = TRUE WHERE (c0."code" = 'foo') []

10:20:41.456 [debug] QUERY OK db=0.2ms
rollback []

10:20:41.464 [error] Process #PID<0.204.0> raised an exception
** (Postgrex.Error) ERROR 40001 (serialization_failure): could not serialize access due to concurrent update
     (ecto) lib/ecto/adapters/sql.ex:440: Ecto.Adapters.SQL.execute_or_reset/7
     (ecto_isolation) lib/ecto_isolation/safe_transaction.ex:34: EctoIsolation.SafeTransaction.run/2
     (ecto) lib/ecto/adapters/sql.ex:576: anonymous fn/3 in Ecto.Adapters.SQL.do_transaction/3
     (db_connection) lib/db_connection.ex:1283: DBConnection.transaction_run/4
     (db_connection) lib/db_connection.ex:1207: DBConnection.run_begin/3
     (db_connection) lib/db_connection.ex:798: DBConnection.transaction/3
```

Success! We got the correct error.

## Conclusions

It's arguable that all of this effort was overkill for what amounted to a one-line fix.

That argument isn't wrong.

But there were a bunch of benefits to going through this exercise. I added to my knowledge about Ecto which is a tool I have to use every day. More importantly I worked out a reasonable pattern for testing race conditions that I can use again in the future. Plus its fun to throw something together! Thats really reason enough.

All of the test code is [available on github](https://github.com/keathley/ecto_isolation) if you wanna check it out.

