---
title: Writing a Brainfuck interpreter in Elixir, part two
date: "2014-11-10"
description: This is the second in a series of articles on building a brainfuck interpreter in Elixir. In <a href="/2014/10/15/elixir-as-a-pasring-tool-writing-a-brainfuck-interpreter-part-one.html">the first part</a> we built a minimal brainfuck interpreter that can already run some basic program. In this second part we'll finish it implementing loop handling.
source_name: MIKAMAYHEM
source_url: http://dev.mikamai.com/post/102283561929/elixir-as-a-parsing-tool-writing-a-brainfuck
tags:
- Elixir
- brainfuck
- brainfuck interpreter
- brainfuck interpreter in Elixir
- parsing in Elixir
---


> This is the second in a series of articles on building a brainfuck interpreter in Elixir

In the [first part](http://dev.mikamai.com/post/100075543414/elixir-as-a-parsing-tool-writing-a-brainfuck) we built a minimal brainfuck interpreter that can already run some basic program.
For example

```brainfuck
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++.
# prints A

,-.
# prints the ASCII character preceding the one taken as input
# in "B" -> out "A" 
```

But honestly we can't do anything more with it.  
The first missing feature is *memory management*. We have implemented the functions that move the pointer to memory cells left and right, but we're still stuck with a non expanding memory tape of one cell only.  
Let's implement memory auto expansion, turns out it is gonna be very easy.

### Memory management

The functions that handle the `>` and `<` operators are

```elixir
defp run(@op_pinc <> rest, addr, mem, output) do
  run(rest, addr+1, mem, output)
end

defp run(@op_pdec <> rest, addr, mem, output) do
  run(rest, addr-1, mem, output)
end
```

One easy way to try to implement an auto expanding memory would be to check if we are accessing a cell past the end of the tape or before the initial one and handling those edge cases.
Something like this

```elixir
defp run(@op_pinc <> rest, addr, mem, output) do
  if length(mem) == addr+1 do
    mem = mem ++ [0]
  end
  run(rest, addr+1, mem, output)
end

defp run(@op_pdec <> rest, addr, mem, output) do
  if addr == 0 do
    mem = [0] ++ mem
  end
  run(rest, addr-1, mem, output)
end
```

This implementation has several problems:

- the second function does not work, it keeps decrementing the memory pointer and can go negative, which is an unwanted behaviour (it should be `zero`). We should add another `if` to make it work, not so good.
- We are shadowing the original `mem` variable by reassigning its value.    
- we are modifying code that already works (we'll talk about testing Elixir code in the next article) introducing branching and making it harder to mantain
- we're not *taking advantage of Elixir features*

One of the main selling point of Elixir is pattern matching in function declaration.   
But pattern matching alone could not be enough, as we've seen in this example.   
In Elixir you can add conditions to function declarations that act in tandem with pattern matching and limit the range of values a function accept. They are called `guard clauses`, I'll rewrite those two functions to use them:

> note the `when` after function declaration

```elixir
# we are moving past the end of the memory tape
defp run(@op_pinc <> rest, addr, mem, output) when length(mem) == addr + 1 do
  # append a new cell, initialize its value to zero, return the next address
  run(rest, addr+1, mem ++ [0], output)
end

# we are moving to the left of the first cell of the memory tape
defp run(@op_pdec <> rest, addr, mem, output) when addr == 0 do
  # prepend a new empty cell, initialize its value to zero, return zero as address
  run(rest, 0, [0] ++ mem, output)
end
```

This is much better,  requires no branching, can be added without modifying the existing code and by looking at the definition we can easily guess when the function is going to be called.   
The only limitation is that Erlang VM only allows [a limited set of expressions in guards](http://elixir-lang.org/getting_started/5.html#5.2-expressions-in-guard-clauses.).

With this simple addition, we have created a complete memory management system, that can automatically expand on both sides, in a virtually unlimited way.

### Loops and jumps

Implementing brainfuck operators has been quite linear until now.
We just had to follow the language specs and we obtained a working implementation.

Loops are a bit harder task though.   
The following are the representations of the two ways to define loops

the `while` loop 
![flow chart of while loop](http://i.imgur.com/IiIEPo8.jpg)  

the `do until` loop
![do until loop](http://i.imgur.com/Joke2ar.jpg)

In Elixir specs they are defined as

- `[` if the current cell value is zero, jump to the next matching `]`
- `]` if the current cell value is non-zero jump back to the matching `[`

Looks like brainfuck author overengineered the loops, making it possible to have both.
But when it's time to implement them, we can choose to check the loop condition only one time (at the beginning or at the end) and treat the other end of the loop as an *unconditional* jump.
I've chosen to implement the `while` loop.

To implement the loop in brainfuck, we need a function that matches *balanced couples* of `[` and `]` first (did someone say s-expressions?).   
We cannot simply match `]` when we find a `[` and the reason is fairly obvious: we could not have nested loops (`[[-]-]` would not work).  

The algorithm we are using is the following:

1. when we match a `[` we check the value at the current memory address, if it is zero, we jump past the end of the loop
2. if it is not zero, we extract the loop's body and execute it, like it is a stand alone program, then collect the results and jump (*unconditionally*) to the beginning of the loop
3. go back to 1.

In code it is

```elixir
defp run(@op_lbeg <> rest, addr, mem, output) do
  case mem |> byte_at addr do
    0 ->
      run(rest |> jump_to_lend, addr,  mem, output)
    _ ->
      {a, m, o} = run(rest |> loop_body, addr,  mem, output)
      # prepend [ to the input, to make sure we call this function again
      run(@op_lbeg <> rest, a, m, o)
  end
end
```

remember that in Elixir `_` means *match everything*.

To match the balanced couples of square brackets, I used this algorithm

1. when a `[` is found, pass the rest to a matcher function with a `depth` parameter with value `1` and a parameter `acc`, to hold the length of the loop body, initially set to `zero`
2. for every character we find, we increment `acc`
3. if we find a `[`, increment `depth` too 
4. if we find a `]` decrement `depth`
5. if `depth` is zero, we have found the end of loop, return the length of its body
6. if we reach the end of the input and `depth` is non-zero, square brackets are unbalanced, `raise` an error then.  

Let's translate this to Elixir

```elixir

# start the matching loop
defp match_lend(source), do: match_lend(source, 1, 0)

# if depth is zero, we have reached the other end of the loop
# return the body length
defp match_lend(_, 0, acc), do: acc

# if we reached the end of the input, but depth is not zero, the
# sequence is unbalanced, raise an error
defp match_lend(@empty, _, _), do: raise "unbalanced loop"

# [ increment the depth
defp match_lend(@op_lbeg <> rest, depth, acc), do: match_lend(rest, depth+1, acc+1)
# ] decrement the depth
defp match_lend(@op_lend <> rest, depth, acc), do: match_lend(rest, depth-1, acc+1)
# every other character just increment acc (loop body length)
defp match_lend(<<_>> <> rest, depth, acc), do: match_lend(rest, depth, acc+1)

# returns the slice of the input program starting from the end of the loop after ]
defp jump_to_lend(source), do: source |> String.slice (source |> match_lend)..-1
# return the slice of the input that represent the loop's body 
# between 0 and the body length-1 (everything but the last ])
defp loop_body(source), do: source |> String.slice 0..(source |> match_lend)-1

```

This implementation automatically works for nested loops of any depth.
Every time a `[` command is found,  the program is split in a smaller one and executed until the loop condition is met (this does not save you from infinite loops).

We have now a complete implementation of a brainfuck interpreter that can run any brainfuck program.  
To test it let's run it inside `iex` the Elixir shell

![iex brainfuck session](http://i.imgur.com/1lTQqee.gif) 

In the next post I'll talk about testing the code, creating a project and compiling down to an executable and the command line tools.

> [As usual, I've created a gist with all the code presented in this post](https://gist.github.com/wstucco/3064b6d01f1f8cf1292c)

