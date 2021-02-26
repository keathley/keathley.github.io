---
layout: post
title:  "Using Regulator"
date:   2021-02-26 13:52:00
categories: regulator queues
---

> I originally wrote this for the backend engineers at Bleacher Report. I thought
> that it might be useful to others to repost it here. I've obfuscated the names
> of the specific services but otherwise left it as is.

[Regulator](https://github.com/keathley/regulator) is our service's first line of defense.

It protects each service by dynamically limiting the number of concurrent _things_ that can take place at any given time. I'll try to provide some context on the problem that we're trying to solve, a little bit of queueing theory, what Regulator is doing under the hood, and how to configure Regulator itself to provide the most benefit.

## Dammit Keathley, just tell me how to configure your stupid library

Ok. TL;DR - If you're configuring a "client" (someone making calls to a service) use the AIMD regulator like so:

```elixir
Regulator.install(:downstream_service, {Regulator.Limit.AIMD, [timeout: 15]})
```

The `:timeout` value should be set to the maximum, average latency you expect from the service. If the average latency drifts above this value, Regulator will treat it as an error and begin to limit the number of requests your service can make to the downstream service.

If you're adding a "service" regulator (a service protecting itself from overload) you should use the Gradient regulator with the default values:

```elixir
Regulator.install(:myself, {Regulator.Limit.Gradient, []})
```

If you want to learn about _why_ this works, please read on.

## Why do you keep saying 'queueing theory' in slack all the time?

You know how people say "it's turtles all the way down" and you never understood why or what that even means?

Same.

Anyway, that example is only relevant because it's also true about queues and queueing theory. Computer systems, turns out, almost always boil down to queues, and understanding them is one of the best uses of time if you want to make "performance" part of your career.

The good news: "queueing theory" is really just applied probability and statistics.
The bad news: no one remembers probability and statistics.

Which makes sense. Who would have thought that a junior year, blow-off math class was going to be the fucking cornerstone of performance optimizations for working-class software wranglers?

The really good news: The math isn't that bad. In fact, most of the time you don't need math as much as you need an *intuition* the math. When
you need actual math, it's typically not worse than algebra. Let's look at an example to see how this works out.

Let's imagine that Zeus receives 1 request per second. In that one request, Zeus needs to call Athena and Athena takes 100ms on average, to process the request. In technical terms, we'd say that Zeus's *Arrival Rate* is 1 RPS and its *work time* is 0.1 S. Because Athena's work time is so much lower than the rate of arrival, Zeus will only ever be processing 1 request at a time. In queueing terms, we'd say that the number of items in the queue is 0.

But, let's see what happens if the arrival rate increases to 10 RPS. We don't know _how_ these requests arrive yet - all at once, one at a time, spread out over a Pareto distribution (spoilers: it's this one) - but we're not worried about that.

Question: On *average*, given an arrival rate of 10 RPS and a work time of 0.1 seconds, how many requests are waiting in LB waiting to be fulfilled over a 1-second window?

Think about it for a bit...

Ok, if you answered "1 request" you have an intuition for what's going on here. If you got something else, that's fine. I'm about to show you the math that allows you to calculate this yourself.

In math terms, we can say that the average items in a queue are equal to the average arrival rate multiplied by the average work time:

`avg. items in queue = avg. arrival rate * avg. work time`.

You'll typically see this equation written out with fancy-pants math terms:

`L = λ x W`

It turns out that this little equation (giggle) is actually quite famous. It's
known as Little's Law and is one of the most ubiquitous pieces of mathematics. Little's Law is important to internalize because Little's law
works for *all* queueing systems, including the queues inside your queues.

In our example above, there are really 3 queues that we care about (and there are more internally that we're ignoring). There's Zeus internal queue, Athena's internal queue, and the queue that encapsulates both of them that the client experiences.

Remember when I said that its queue's all the way down? This is what I meant.

Now that we can understand a bit more about queues, we can talk about overload and what causes systems to fail.

## Overload

When one of our services fails, it's almost always due to a condition called "overload". We tend to use the term overload for a wide class of errors, but it actually has a technical definition. A queue is considered "overloaded" when the queue's arrival rate is higher than its departure rate. If your queue is accepting things faster than it can process them, you have overload, and you're going to have a bad time.

Now that we understand Little's Law we can talk more formally about what happens when a queue becomes overloaded, and we can see why when Athena gets slow, Zeus's CPU starts to spike.

Back to our initial example, if Zeus's arrival rate is 10 RPS and Athena's work time is 100ms, that means that we have 1 request waiting in Zeus to be processed. What happens if Athena's work time increases to 200ms?

To understand this we need to first answer the question, "What is Athena's
arrival rate?". What's your intuition?

If you said, "it's equal to the work time" then you are correct. In our example, Zeus functions as a serial queue. It can only send requests to Athena as fast as Athena can process them. So, if Athena's work time increases to 200ms where does the queueing occur? Zeus of course. But, Zeus's arrival rate is based on client demand. So it's still receiving 10 RPS. But Zeus's effective work time has increased to 200ms. So, we bust out Little's Law real quick and figure out how many
requests are queueing in Zeus and get 2. That's not that bad.

But, what happens if Zeus's arrival rate jumps to 100 RPS? Well, Athena's work time is going to stay fixed at 200ms. This means we have 100 * .2 which gives us 20 requests. 

So, Zeus is now going to have 20 requests piling up and waiting to be fulfilled and that pain isn't going to be felt by Athena directly.

This is another important intuition to develop. A downstream system may never detect that queueing is happening upstream. Athena's database might get a little backed up and Athena won't consider that to be an error. But that small hiccup results in increased CPU pressure in Zeus because now Zeus has requests piling up. Coincidentally, this
is why rate-limiting doesn't prevent overload.

What happens to Zeus in this scenario? Well, on a long enough time frame, it'll collapse. Once a queue has been built up in this way, it's very hard to eliminate that queue, even if Zeus's arrival rate goes back to 10 RPS. In fact, to eliminate the queue in Zeus, Athena's work time would need to drop *below* its original 100ms. While queued, those requests are holding open expensive resources like HTTP connections, using memory, and adding additional pressure to the BEAM schedulers. When situations like this occur, you have exactly 2 options:

* Do nothing and let the entire system crash.
* Drop your work time as low as possible, preferably to 0.

Ideally, we should be choosing the second option. Also, not explicitly choosing the second option is implicitly choosing the first option. Not making a choice is the same as choosing the first option.

Dropping work like this is referred to as "load shedding". Which really means dropping the work time so low that it avoids overload. These days, our tool of choice for load shedding is Regulator.

## Regulator implementation details

Regulator is based on the same techniques that power the internet's networks. These techniques are often referred to as "adaptive concurrency" or "adaptive capacity" in the literature. It's "adaptive" because the system regularly makes changes based on observed values. 

Regulator only allows a certain number of _things_ to occur in a system concurrently. For instance, Regulator may have determined that
Athena can handle 5 requests concurrently. If a 6th request is made before any of the original 5 have been completed, the 6th request will be rejected.

Periodically, Regulator examines the requests its seen and decides whether or not it should allow more or fewer things to happen concurrently. Typically, Regulator is examining signals such as changes in latency, whether an error occurred, how many requests were in-flight at the same time, etc. It does all of this to avoid any requests queueing in the system.

So, in our initial example, if Zeus used a regulator around its calls to Athena, the regulator would see the increase in work time and arrival rate and would begin to reject excess requests to avoid queueing. Because our work time has now dropped by so many orders of magnitude, we can return a 500, or we can return a cached value from memory. Either of these techniques is appropriate and has practically no difference in response time. We've eliminated the central bottleneck and that tends to be Good Enough for eliminating overload.

## Where should I use regulators?

The short answer here is:

1) On any outbound API calls
2) In front of every service

BRPC provides examples of how to do both of these. Each service should protect itself with a regulator to avoid unbounded queue growth. Likewise, upstream services should avoid calling downstream services if they become overloaded and thus should fail as quickly as possible and avoid doing useless work.

Empirically, we've found that AIMD works well for outbound calls and Gradient works well for services. But, I think it's worth questioning these ideas and fiddling around with them. It might be that we should be using AIMD everywhere, for instance. Both of these algorithms can be tweaked and tuned in various ways (refer to the regulator docs for details) but they both work on similar principles. They look at samples to determine the overall health of the system. Once they've made that determination they adjust how much work the system is allowed to do to avoid overload.

## So what did we freaking learn here?

I hope that this gives you some intuition about queues and how to protect your system from overload. The Regulator docs provide more specifics on how each limiter works and the various ways that it can be tuned.
