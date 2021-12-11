---
layout: post
title:  Testing your README.md
date:   2021-12-11 11:00:00
categories: Elixir Patterns
---

I've wanted a good way to test READMEs in elixir. I started building something myself
before my good friend [Wojtek](https://twitter.com/wojtekmach) pointed out that there was a simple solution.

I like to use the README or at least a section of the README as the module doc
for the main module in my libraries. Typically that looks something like this:

```elixir
defmodule Vapor do                                                              
  @moduledoc "README.md"
             |> File.read!()
             |> String.split("<!-- MDOC !-->")
             |> Enum.fetch!(1)                                                  
end
```

I use `<!-- MDOC !-->` to mark the section of the README that I want to use as module doc. This way I can remove the installation instructions or other material that doesn't serve any purpose in the module doc.

The trick that Wojtek clued me into, is that if you write your examples using the doctest syntax, then these can be automatically doctested.

I've slowly started adding this to all new modules that I write. Its an easy way to ensure that your README doesn't get out of date with reality. If you want to look at a complete example of this, you can check out how I've structured [Norm](https://github.com/elixir-toniq/norm).
