---
layout: post
title: Writing a Brainfuck interpreter in Elixir, part one
excerpt: Brainfuck is an tiny, crazy, esoteric, turing complete, programming language made of only 8 instructions. The perfect language to write an interpreter for, the perfect small project to learn about Elixir fun!
tags:
- Elixir
- brainfuck
- brainfuck interpreter
- parsing in Elixir
---


> For instructions on how to install the Elixir environment you can take a look at the [getting started guide](http://elixir-lang.org/getting_started/1.html#1.1-installers).

To get used to the language and try some of the code in this post, you can start `iex`, the Elixir shell.
One of its best features are autocompletion of module and function names and the integrated documentation accessible with the command `h`.
This is an example of an `iex` session:

![image](http://i.imgur.com/IigY3j3.jpg)


### Brainfuck, the language

Brainfuck is an esoteric, turing complete, programming language with a very small set of instructions: there are only 8 of them.
.
A typical implementation requires a memory of at least 30 thousand cells, but ideally infinite on both sides (we'll see this is very easy to implement in Elixir), each initially set to zero and a data pointer that points to the first memory cell.
The available commands are:

- `>` increment the data pointer (point to the cell on the right)
- `<` decrement the data pointer (point to the cell on the left)
- `+` increment the value at the data pointer location
- `-` decrement the value at the data pointer location
- `.` output the value at the data pointer as byte
- `,` read a byte into the cell at pointer location
- `[` if the current cell value is zero, jump to the next matching `]`
- `]` if the current cell value is non-zero jump back to the matching `[`

All other characters are considered comments, hence ignored.

### Implementation details

We are going to write an Elixir module that only export one public function `run` that accept a brainfuck program as string and scan it, character by character, until we reach the end.
As a result it returns a triplet containing the final data pointer address, the memory state and the output generated.
I assumed that each memory cell is an unsigned byte, that overlap on overflow (255+1 becomes zero again).
Input and output operations work on bytes too.

In Elixir pattern matching is a fundamental feature for controlling the program flow, there are no loop instructions, so we are forced to use recursion.
The condition of our loops are expressed in the function definition.

The logic of our interpreter is very simple: we are going to consume the program string char by char by using the pattern below
> `<>` is the operator for string concatenation, it work on bitstring, but since strings in Elixir are binaries, it works on strings too

```elixir
run(first_char <> rest_of_the_program, ... ) do
  ...
  run(rest_of_the_program, ...)
end
```

As an example, the simple program `++.`, which increment the location zero two times, and then output the result will flow like this


```elixir
# step 1
run("+" <> "+.", 0, [0], "")
  # inc the value at zero
  run("+.", 0, [0+1], "")

# step 2
run("+" <> ".", 0, [1], "")
  # inc the value at zero
  run(".", 0, [1+1], "")

# step 3
run("." <> "", 0, [2], "")
  # append the value at zero to output
  run("", 0, [2], "" <> <<2>>)

# final step
run("", 0, [2], <<2>>)
  # return {addr, memory, output} -> {0, [2], <<2>>}


```



### First steps

Let's begin with the definition of our module, we are going to define our instruction set and the run function

```elixir
defmodule Brainfuck do

    # opcodes
    @op_vinc "+" # increment value at memory address
    @op_vdec "-" # decrement value at memory address
    @op_pinc ">" # increment memory address
    @op_pdec "<" # decrement memory address
    @op_putc "." # output byte at memory address
    @op_getc "," # input byte into memory address
    @op_lbeg "[" # loop begin
    @op_lend "]" # loop end

    def run(program), do: run(program, 0, [0], "")

```

You may have noticed that run doesn't really do anything, except call another run function with different parameters.
This is a common pattern in Elixir programs: functions are defined as name/arity (the arity is the number of parameters a function takes), so for example the first `run` function is `run/1` and the one we are calling is `run/4`, for Elixir they are two completely different functions, even if they share the same name.
Elixir have strictly immutable types, we need to carry the state around by passing it in parameters, we'll see in a minute what the parameters are for.


### With a little help from my friends

Before getting our hands dirty (they're not gonna be that dirty, I promise) I'm going to show you some helper functions I created to keep code as clean as possible.

```elixir
defp inc_at(list, addr), do: List.update_at(list, addr, &(&1+1 |> rem 255))
defp dec_at(list, addr), do: List.update_at(list, addr, &(&1-1 |> rem 255))
```

First thing to notice is that they start with a `defp` not `def`.
It means they are private to the module and not visible from the outside.
Both of them take two parameters, a list and an address that represent the position inside the list, and update the corresponding value.
It's easy to guess by their name what they do: `inc_at` increment the value of `list` at position `adrr` and `dec_at` decrement it.
We use the facilities provided by the core `List` module by calling `update_at`, that takes a list, an address and a function that update the value.
As we said in Elixir data types are immutable, so `update_at` is returning a copy of the list with the modified value, it is not modifying the value in place.
The only tricky part is understanding the third parameter, the update function.
The `&()` is called [capture syntax](http://elixir-lang.org/getting_started/8.html#8.4-function-capturing) in Elixir and is basically a shorthand for creating anonymous functions.
Rewriting the same function without it would look like this `fn(a) -> a+1 |> rem 255 end`, the capture syntax is more concise and allows us to get rid of the function parameter and use params placeholders (`&1` represent the first parameter, `&2` the second and so on).

The other thing you might have noticed, if you are new to Elixir, is the `|>` symbol. That's called the pipe operator and act much like a unix pipe, it `cat`s the argument(s) on left as **first parameter** of the function on the right.
So `rem &1+1, 255` can be rewritten as `&1+1 |> rem 255`.
It has no advantage in this case as number of characters typed, but it makes clearer what we are doing: we are taking the value of the first parameter, adding (or subtracting) 1 to it and then piping the result on the function `rem` with the parameter 255.
`rem` returns the remainder of the int division, that's how we keep memory values in the byte size range, by going back to zero when the value overflows 255.


In the same family, but with a different purpose, I created `put_at`, that completely replace a value in a list at a speicified address.
Of course this function takes a third parameter, the new value we are putting into the list.

```elixir
defp put_at(list, addr, val), do: List.replace_at(list, addr, val)
```


### The real `run` function

So what are those parameters we pass to our internal function for?
I'll explain by showing you the final step of our interpreter

```elixir
defp run("", addr, mem, output), do: {addr, mem, output}
```

The first parameter is the program string *at this point* of the execution, second is the pointer to the memory cell currently active, third is the current state of the memory tape and the last is the output string we are going to return.
We know our program has ended when the run function is called with and empty string as first parameter.
We then return the triplet containing the current data pointer, the memory cells and the output string.

Without the previous function our program would not end and it will give us an error like this

```elixir
** (FunctionClauseError) no function clause matching in Brainfuck.run/4
    iex:25: Brainfuck.run("", 0, [0], "")
```

because there is no function matching the pattern with the empty string as first param.

The second basic function is the generic one that matches a string starting with some character, no matter which,  skips it, and calls run again with the rest of the string.

```elixir
defp run(<<_>> <> rest, addr, mem, output), do: run(rest, addr, mem, output)
```

With this two functions in place we already have a scanner, a completely useless scanner, that skips everything and then returns the initial state.The complete code for this is just a few lines long

```elixir
defmodule Brainfuck do
    def run(program), do: run(program, 0, [0], "")
    # exit program
    defp run("", addr, mem, output), do: {addr, mem, output}
    # skip everything
    defp run(<<_>> <> rest, addr, mem, output), do: run(rest, addr, mem, output)
end

Brainfuck.run("hello world")
# output: {0, [0], ""}
```

> Note that Elixir matches patterns from top to bottom, so we need to put the function that skips unrecognized commands at the end, otherwise more specific patterns would be ignored.


### Basics, strings and the I/O

I'm gonna start implementing commands, by defining the functions that handle the `I\O` operations, basically they output a byte and read a byte from input (in our case `stdin`).
To do that, first we need to introduce two more helper functions

```elixir
defp byte_at(list, addr), do: list |> Enum.at addr
defp char_at(list, addr), do: [list |> byte_at addr] |> to_string
```

 `byte_at` extracts the byte at position `addr` in `list` (AKA our memory cells), while `char_at` returns the same byte as string value.
In Elixir strings are binaries, or, in other words, strings of bits.
To convert a byte value to a string, we cannot simply use `to_string` function, because it will convert the byte to its string representation, not to the character rapresented by its value, so we need to wrap it inside [] and make it a byte list (the internal representartion of ASCII strings).
As an experiment, you can start `iex` and try this code

```elixir
65 |> to_string # print "65"
[65] |> to_string # print "A"
[65, 66, 67] |> to_string # print "ABC"
```

Handle @op_putc opcode, that appends one byte to `output`

```elixir
defp run(@op_putc <> rest, addr, mem, output) do
    run(rest, addr, mem, output <> (mem |> char_at addr))
end
```

When @op_putc is at the beginning of the program, this function call `run` with the new output formed by appending the character at the current memory location to the old output.
Rest becomes the new program, while address and memory are unchanged.

Next is @op_getc, which reads a byte from `stdin` and puts it in the current memory location.

```elixir
defp run(@op_getc <> rest, addr, mem, output) do
    val = case IO.getn("Input\n", 1) do
        :eof -> 0
        c    -> c
    end
    run(rest, addr, mem |> put_at(addr, val), output)
end
```

It's a bit trickier than the previous, but gives us the opportunity to introduce the `case` statement.
In Elixir everything is an expression, and returns a value.
We use this feature to assign to `val` the result of the `case` expression.
Inside the case we use pattern matching to match the return value of `IO.getn`, which, straight from the Elixir interactive help

```elixir
Gets a number of bytes from the io device. If the io device is a unicode
device, count implies the number of unicode codepoints to be retrieved.
Otherwise, count is the number of raw bytes to be retrieved. It returns:

  • data - the input characters
  • :eof - end of file was encountered
  • {:error, reason} - other (rare) error condition; for instance, {:error,
    :estale} if reading from an NFS volume
```

We read one byte from the input, if it returns `:eof`, return 0, if it returns some `data`, we return it (it is guaranteed to be one byte long).
We ignore error conditions, since they are very rare, especially in our simple case.
The new memory will have `val` value at `addr` position.

Not hard at all until now.

![very easy](http://i.imgur.com/6oA7eED.png)


### Let's talk about memory

There are two opcodes in brainfuck that operates on memory values, `+` and `-`.
The implementation is very straightforward

```elixir
defp run(@op_vinc <> rest, addr, mem, output) do
    run(rest, addr, mem |> inc_at(addr), output)
end

defp run(@op_vdec <> rest, addr, mem, output) do
    run(rest, addr,  mem |> dec_at(addr), output)
end
```

Now that we (hopefully) grasped the basics of the Elixir syntax and how pattern matching is used, it should be pretty easy to understand how these two functions work.

Last two functions we meet today handle the data pointer.
I'll just show you the two basic cases, when the pointer moves inside the memory length, we'll keep handling the auto expansion of the tape to the left and right for the next part.

```elixir
defp run(@op_pinc <> rest, addr, mem, output) do
    run(rest, addr+1, mem, output)
end

defp run(@op_pdec <> rest, addr, mem, output) do
    run(rest, addr-1, mem, output)
end
```

Almost no need to explain what it is going on, the data pointer is simply incremented or decremented and the new value is passed to `run`.

In the next post I'll talk about how to handle expanding the memory tape when needed and, the most fun part, where Elixir capabilities really shine, handling loops and jumps in a very easy way.

> [I've created a gist with all the code presented in this post](https://gist.github.com/wstucco/bc6a5037fe8b1fbf1cf0), of course it misses loops, but you can use it as a starting point for your own experiments






