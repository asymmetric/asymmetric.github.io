---
title: Assignments and Pattern-Matching in Elixir
---

While reading Dave Thomas' [Programming Elixir][book] book, I was confused by the
[following][code] piece of code:

```elixir
defmodule Users do
  dave = %{ name: "Dave", state: "TX", likes: "programming" }

  case dave do
    %{state: some_state} = person ->
      IO.puts "#{person.name} lives in #{some_state}"

    _ ->
      IO.puts "No matches"
  end
end
```

My problem was understanding line 5: `%{state: some_state} = person`. How could
the `person` variable get its value, if it was **on the right side of the
assignment**?

### A digression on Pattern Matching

Now, if you're not familiar with Elixir, you might not know that in it, the `=`
operator is not exactly an assignment. What it is instead, is an operator for
**Pattern Matching**.

If you recall basic math classes from school, you might remember that *x = 1*
didn't mean *"assign the value 1 to the variable x"*. It meant: both sides of
the equation are *equal*.  Saying that *x = 1* implied that *1 = x*.

Now, in imperative languages, this is not the case. To take an example from
Ruby, if you try and run `1 = x`, you'll get an error (albeit a very cryptic one):

```ruby
irb(main):001:0> 1 = x
SyntaxError: (irb):1: syntax error, unexpected '=', expecting end-of-input
1 = x
   ^
```

Which is Ruby basically telling you that it's not expecting an assignment after
an integer.

In Elixir, the `=` operator works like it does in algebra:

```ruby
iex(1)> x = 1
1
iex(2)> 1 = x
1
```

In other words, Elixir tries to pattern-match (according to rules which are
outside the scope of this article) the left hand of the equation with its right
hand. If it succeeds, it returns the value of the equation. If it fails, it
throws an error.

This allows us to effectively *bind* values to variables, like we did above.
What we **cannot** do, however, is **introduce** variables on the right hand
side of an assignment/pattern-matching/equation.

So if I try to do `1 = x` for an **uninitialized** variable `x`, Elixir will complain:

```ruby
iex(1)> 1 = x     
** (CompileError) iex:1: undefined function x/0
```

### End of digression

So, how could it be that we could run `%{state: some_state} = person`?

The point is that I was focusing on the wrong part of the expression, which
structurally is of the form:

```ruby
left -> right # %{state: some_state} = person -> IO.puts "..."
```

The `->` operator mandates that its left side hand be a pattern. So the
**whole** expression `%{state: some_state} = person` is a pattern.

And when you're assigning inside a pattern match, the side of the assignment
**does not matter**.

For example, you can do `(1 = x) = 1`, and it won't throw an error. Or, as
José himself [explained][irc-logs], you could run `[ h = 1 | t ] = [ 1,2,3 ]`.
Or `[ 1 = h | t ] = [ 1,2,3 ]`, for that matter.

This is called "Nested pattern matching" in Dave Thomas' book, but I'm not sure
how widely adopted this term is.

So, there it is. I learnt something new. I still think that I'll use the
`variable = expression` syntax in my code, but it's good to be aware of this.

[book]: https://pragprog.com/book/elixir/programming-elixir
[code]: http://media.pragprog.com/titles/elixir/code/control/case1.exs
[irc-logs]: http://irclogger.com/.elixir-lang/2015-11-25#1448458473
