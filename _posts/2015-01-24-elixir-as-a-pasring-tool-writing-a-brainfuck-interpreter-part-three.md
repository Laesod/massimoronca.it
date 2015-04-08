---
layout: post
title: Brainfuck in Elixir, part three, compiling
excerpt: This is the third in a series of articles on building a brainfuck interpreter in Elixir. In <a href="/2014/10/15/elixir-as-a-pasring-tool-writing-a-brainfuck-interpreter-part-one.html">the first part</a> we built a minimal brainfuck interpreter that can already run some basic program. In <a href="/2014/11/10/elixir-as-a-pasring-tool-writing-a-brainfuck-interpreter-part-two.html">the second part</a> we completed it by implementing loops. In this third episode we'll write a simple compiler to translate Brainfuck instructions to a machine readable intermediate format (AST) and a VM that executes it.
source-name: MIKAMAYHEM
source-url: http://dev.mikamai.com/post/109477559604/brainfuck-in-elixir-part-three-compiling
tags:
- Elixir
- brainfuck
- brainfuck compiler
- brainfuck compiler in Elixir
- parsing in Elixir
---

> This is the third in a series of articles on building a brainfuck interpreter in Elixir.  
In the [first one](/2014/10/15/elixir-as-a-pasring-tool-writing-a-brainfuck-interpreter-part-one.html) we built a minimal brainfuck interpreter that could understand the basic instructions.  
In [the second](/2014/11/10/elixir-as-a-pasring-tool-writing-a-brainfuck-interpreter-part-two.html), we completed it by implementing loops.  
In this third episode we'll write a simple compiler to translate Brainfuck instructions to a machine readable intermediate format (AST) and a VM that executes it.   

This post was supposed to be about testing and the command line tools, I changed my mind and I will talk about improving our interpreter and turning it into a compiler. 
   
Writing a performant compiler is probably one of the most challenging tasks for a programmer, but the theory behind it is actually quite simple.   
Compilers just *transform* a source code written in a programming language to some other code, usually a different programming language (including intermediate languages and machine language).   
Most of the time, they are built following a common design, this one

![Compiler design](http://twimgs.com/ddj/images/article/2012/0512/latfig1.gif)
> copyright [Dr. Dobb's](http://www.drdobbs.com/architecture-and-design/the-design-of-llvm/240001128)

Our compiler will be much simpler: we will completely skip the optimizer (for now) and will directly execute the output of the fronted (AST or abstract syntax tree, from now on).  

Technically, we are writing the frontend of the compiler, which is the starting point for everything else to come.   

#### The intermediate language

Brainfuck is already an intermediate language, very similar to assembly. Each symbol is an `opcode`: for example `>` and `<` can be easily mapped to a relative `JMP`, `+` maps to `INC`, `-` to `DEC`, `[` and `]` are `JZ` (jump if zero) and `JNZ` (jump if not zero).   
`.` and `,` are more complex, there's no single instruction in assembly for reading and writing chars to the screen, but basically they are the `C` equivalent of `putchar` and `getchar`.   

Everybody loves assembly, but we will not use those `opcodes`, we will use something more similar to labels, something mnemonic, because, right now, we just need to know which block to eceute, we have very simple instructions, with no parameters, that do just one thing.   
Things will change when we'll get our hands on the optimizer, but for now we'll keep things simple, and map Brainfuck instructions to [Elixir atoms](http://elixir-lang.org/getting_started/2.html#2.3-atoms) (think about them as Ruby's symbols).   

Our instructions set will be the following:  

```elixir
+ -> :inc_d
- -> :dec_d
> -> :inc_p
< -> :dec_p
. -> :put_c
, -> :get_c
```


Loops are mapped to [Elixir keywords](http://elixir-lang.org/getting_started/7.html#7.1-keyword-lists), we already ignore the end loop instruction `]`, because we unconditionally jump back to `[` when we find one.  
That leaves `[` as the only complex instruction in the set, the only that carries a parameter (the body of the loop).   
So loops are defined as  

```elixir
[ -> {:loop, [loop body]}
```

or in the condensed form   

```
[ -> [loop: [loop body]]
```

Loop body is always a list of instructions.  

#### The compiler implementation

To write the interpreter, we already wrote a Brainfuck scanner, tokenizer and parser. We'll take advantage of it to emit our `AST`, turns out it can be written in a very compact way

```elixir
defmodule Brainfuck.Compiler do

  def compile(program) when is_binary(program) do
    compile(program |> to_char_list)
  end

  def compile(program) do 
    compile(program, [], [])  
  end
  
  defp compile([], _, stack) when length(stack) > 0 do 
    raise "unmatched '['"  
  end
  
  defp compile([], ast, stack) when length(stack) == 0 do 
    ast  
  end

  defp compile([ic | tail], ast, stack) do
    case {[ic], stack} do
      { '+', _ } -> compile tail, ast ++ [:inc_d], stack
      { '-', _ } -> compile tail, ast ++ [:dec_d], stack
      { '>', _ } -> compile tail, ast ++ [:inc_p], stack
      { '<', _ } -> compile tail, ast ++ [:dec_p], stack
      { '.', _ } -> compile tail, ast ++ [:put_c], stack
      { ',', _ } -> compile tail, ast ++ [:get_c], stack
      { '[', _ } -> compile tail, [], [ast] ++ stack
      { ']', [] } -> raise "unmatched ']'"
      { ']', [h | t] } -> compile tail, h ++ [loop: ast], t
      _ -> compile tail, ast, stack
    end
  end

end
```

Pretty short, pretty easy to follow.  
First we create a `Brainfuck.Compiler` module, that is our namespace for the compiler, we then put a few conditions: if the input is a string, we convert it to a [char list](http://elixir-lang.org/getting_started/6.html#6.3-char-lists) that is easier to traverse, and we declare that when `compile` is called with an empty list, there are no more instructions to translate, we are done and return the `AST`.  
Unless the stack is not empty, which is an error condition, we'll se why in a moment.  

Every instruction found, is appended to the `AST` list, every not recognized symbol, is ignored and discarded.  

What it does is take this input

```brainfuck
+-><[.,]
```

and translate it to

```elixir
[:inc_d, :dec_d, :inc_p, :dec_p, loop: [:put_c, :get_c]]
```


Loops are handled recursively: once we find a `[`, we save in a `stack` the current `AST`.  
We use the stack as a FIFO queue, we prepend the `AST` to the actual value of the stack, because the loop could be nested, we then execute the body of the loop like it was a standalone program.  
When we find a `]`, we pop from the head of the stack and prepend it to the loop `AST` and keep popping until we have emptied the stack.   
This way we can track unbalanced pairs of `[` and `]`.  
If we pop and the stack is empty, we popped once too often.   
If we get to the end, where no instruction is left to be picked up, and the stack is not empty, we haven't popped enough.  


### The Virtual Machine

Our virtual machine is not really a virtual machine, in the strict sense of the term, it is more a runtime that knows our bytecode and how to execute it.  
It is really not much different from the interpreter, it reads a list of inputs and decide what to do with them.   
But, it has some advantages.   
The first one is that the compiler ensures correctness of our code: we can't be sure that the code does what it is supposed to do or that there won't be an nfinite loop, but we can assume it is formally correct (no unbalanced loops, for example).  
The second one is that having a bytecode, enable us to optimize the code.  
The simpler optimization is that we don't have to scan the code back and forth to find the boundaries of the loops, they are already expressed in the `AST`.  
Infact to `run` loops the VM we just executes them  

```elixir
defp run(program = [{:loop, loop} | tail], addr, mem, input, output) do
  case mem |> byte_at addr do
    0 ->
      run(tail, addr, mem, input, output)
    _ ->
      {a, m, i, o} = run(loop, addr, mem, input, output)
      run(program, a, m, i, o)
  end
end
```

> `keywords` in Elixir are matched with `{:key, value}`

This optimization alone makes our programs run up to six times faster.   
Conclusions are based on higly non-scientifical benchmarks  


```bash
$ time ./brainfuck test/fixtures/bench_ok.bf
  OK
  
  real  1m10.399s
  user  1m9.483s
  sys 0m0.391s

$ time ./brainfuck -i test/fixtures/bench_ok.bf
  OK
  
  real  6m2.453s
  user  5m57.601s
  sys 0m1.704s 
```

The more loops there are in the Brainfuck code, the more it should benefit from the compilation.   

You can find all the code [on github](https://github.com/wstucco/elixir-brainfuck/), to create the brainfuck executable run `mix escript.build`, if you run it with the `-i` flag, it will use the interpreter written in the previous two articles, otherwise it will use the compiler.  
To run the tests use `mix test`.   
  

And if you find the reason why the interpreter gets stuck in an infinite loop running [this brainfuck program](https://github.com/wstucco/elixir-brainfuck/blob/master/test/fixtures/bench.bf), please, [let me know](mailto:massimo@mikamai.com).
