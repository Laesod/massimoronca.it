---
title: "Writing Ruby extensions in Go - an in depth review"
date: "2015-10-12"
description: "For a very long time the only way to extend Ruby with native estensions has been using <i>C</i>.<br>
My goal is to find languages that we can use to create extensions, that a Ruby programmer would understand effortlessly or with a minimum investment.<br>
This first episode will focus on Go"
source_name: MIKAMAYHEM
source_url: http://dev.mikamai.com/post/130986121064/writing-ruby-extensions-in-go-an-in-depth-review
tags:
- Ruby
- Ruby extensions
- Go shared objects
- Ruby extensions in Go
---

For a very long time the only way to extend Ruby with native estensions has been using `C`.  
I don't have the numbers with me, but I guess `C` is not the first choice for Ruby programmers when they have to pick up a secondary/complentary language.   
So I started investigating the possibility of writing them in other languages, taking advantage of the favorable moment: in the past 3-4 years we had an explosion of new languages that compile down to native code and can be easily used to produce a `C` equivalent shared library that Ruby can pick up and load.   
  
My main goal was to find languages that a Ruby programmer would understand effortlessly or with a minimum investment.   
This first episode will focus on Go.   

## You will need Go 1.5 or above

Up until version 1.4, there was really no point in building a native extension in Go, you'd have to create a `C` proxy function for every Go function being called, at the point that there was literally no benefit compared to writing everything in pure `C`.   
With version 1.5, Go made a step forward, introducing the support for building shared objects; this opens up a lot of new possibilities for writing shared code that gets executed outside the Go environment, including Ruby.    


## Done in 60 seconds

This is all your "hello world" extension will just be:

<script src="https://gist.github.com/wstucco/7b0c424cf8dddbc8fd6a.js?file=hello_world.go"></script>

Compile it with

<script src="https://gist.github.com/wstucco/7b0c424cf8dddbc8fd6a.js?file=build.sh"></script>
    
Have I already told you that [FFI](https://github.com/ffi/ffi/wiki) is awesome?

<script src="https://gist.github.com/wstucco/7b0c424cf8dddbc8fd6a.js?file=hello_world.rb"></script>

There are a few things you need to pay attention to:  

- It is better to put `import "C"` on its own line, separated from other imports, we'll see why in a moment  
- `//export` is a special [Cgo](https://golang.org/cmd/cgo/) comment that instructs the compiler to emit a `C` function with the same name and parameters. The name of the exported `C` function must match the name of the Go function or it will fail. The comment must start exactly with `//export`, no spaces anywhere.
- a `func main` is required, but then ignored



## Autarchy, or writing extensions the hard way

This could be all, but of course it is not.   
Writing a Ruby extensions in a different language is one thing, writing it because the other language really offers some noticeable advantage, is entirely different.   
As a base for this series of articles, I will port the [`fast_blank`](https://github.com/SamSaffron/fast_blank) `C` extension.   
I've chosen it for very simple reasons:  

- it is deadly easy to port, even to languages that I'm not particularly familiar with
- it bundles a benchmark suite, so we can measure the performance gains/losses
- it is real life code that's been downloaded a quarter million times so far

> For the impatients: you can clone the [repo of the `go_fast_blank` gem on Github](https://github.com/mikamai/go_fast_blank).   
> 

Even if `FFI` is great, it is still a dependency, that needs to be installed and maintained.   
`C` extensions are usually self contained and take advantage of the `MRI` `C` programming interface to build the necessary exported APIs.   
Go has a very good support for interfacing to `C` as you can see in this file

<script src="https://gist.github.com/wstucco/7b0c424cf8dddbc8fd6a.js?file=go_fast_blank.go"></script>

There are some new tricks here, that need an explanation.   
Just before `import "C"` we find what's called "preamble" by Cgo, and it's just `C` code that the compiler put at the beginning of the generated file, as it is, before starting the compilation.   

> **IMPORTANT**: there must be no empty line between the end of the preamble and the `import "C"` directive or the compilation will fail.  
> that's the reason why I told you to put `import "C"` on its own line.

In other words, the generated code will start with  

<script src="https://gist.github.com/wstucco/7b0c424cf8dddbc8fd6a.js?file=preamble.c"></script>


You probably have guessed that the `C` prefix gives access to the `C` world directly from Go, with some added benefit: for example the `C.CString` converts Go native strings to `C` (`char*`) strings. This function allocates memory, so you *must* free the memory using `C.free`.   

We do that in `defer C.free(unsafe.Pointer(cs))`, that tells Go to free the memory as soon as the surroinding function returns and is a very common pattern.   
The pointer to the string is declared as `unsafe.Pointer` because it does not belong to the Go world.   

Another thing you might have noticed is the reverse twin function `C.GoString` that takes a `C` string and returns a Go string.  In this case no memory is allocated, the GC takes care of everything, so no freeing is required.   

Some of the code just refers to the `MRI` programming interface, defined in `ruby.h` and related headers.   
For example `C.VALUE` is a macro for various types of pointers to data structures (from strings to function pointers) and `C.rb_define_method` defines a new method.   
It takes four parameters: the class to which the method belongs to (in this case `C.rb_cString` which is the Ruby equivalent of the builtin `String` class), the name of the method (in this case `blank?`) a callback and the number of arguments (zero in our case).  
 
Basically we are writing something like

```ruby
class String
    def blank?
    ...
    end
end
```

The third argument of `C.rb_define_method` is the `C` function that gets executed when the method is invoked on the Ruby side.   
The Go runtime and the `C` code are executed in different threads, with different stacks, it it prohibited to pass a pointer to Go code to `C`, so we can't take a pointer to a Go function and simly pass it to `C`, because it won't work.   

There is a workaround, we can `//export` our Go function and pass the pointer to it instead, after casting it to `*[0]byte` (the Go equivalent of `void*`): `(*[0]byte)(C.go_fast_blank)`.   
There is only a small problem: `C.go_fast_blank` does not exists until the `C` files are compiled, so we cannot implicitly refer to it.   
We need to add a forward declaration to tell the compiler we know this function exists and it's imlpemented somewhere else outside here.   
That's what the line `extern inline VALUE go_fast_blank(VALUE);` is for, and it's a standar declaration for `rb_define_method` callbacks (`function_name(VALUE) -> VALUE`).   
The rest is quite straightforward:   

```
gs := C.GoString(C.rb_string_value_cstr(&s))
```

Take a `VALUE` convert it to a `C` string and then convert the result to a Go string.

```
if gs == "" || strings.TrimLeftFunc(gs, unicode.IsSpace) == "" {
    return C.Qtrue
}

return C.Qfalse
```

If the string is empty or after removing all the unicode spaces on the left side, it is still empty, we found a blank string. Otherwise we return false.  
`Qtrue` and `Qfalse` are just two `C` #defines that map to a Ruby boolean.      

Each extension has an `Init` function, and it's automatically called when the extension is `require`d.  
The name of the function must be `Init_#{extension_name}`, in our case `Init_go_fast_blank`.  

Last but not least, to compile our self contained extension, we need to pass some flags to the compiler. We'll do it using a specific Cgo comment: `#cgo`.   
just before the `#include "ruby.h"` add these lines

```C

#cgo: CFLAGS: -I#{RbConfig::CONFIG['rubyhdrdir']} -I#{RbConfig::CONFIG['rubyarchhdrdir']}
#cgo: LDFLAGS: -L#{RbConfig::CONFIG['libdir']} #{RbConfig::CONFIG['LIBRUBYARG']}

```

The interpolation codes must be replaced with the actual value.   
I've put it there for reference.   
Hint: the output must have `.bundle` extension and not `.so` as we did before, othewise Ruby will refuse to laod it.   
In the [repo of the `go_fast_blank` gem](https://github.com/mikamai/go_fast_blank) you can find an ad hoc `extconf.rb` that will take care of everything.   

## [Race for the prize](https://www.youtube.com/watch?v=bs56ygZplQA)

Now we have a compiled, native, Ruby extension, launch `irb` and type

```ruby
2.2.2 :001 > require 'go_fast_blank'
go_fast_blank init
 => true 
```

You should see our extension announcing itself by printing `go_fast_blank init`.   
It's time to measure the performances and comment the results.   
After launching `benchmark`, the numbers are:

![Ruby VS Go](http://i.imgur.com/4YddNT2.png "Ruby VS Go")   
Go is between 2 and 4 times slower than the original Ruby implementation!  

![GO, Y U SO SLOW](http://i.imgur.com/48BOGlI.png "Y U SO SLOW?")

Well, first of all Go is not only slower than Ruby, but it's plateuing, looks like the speed
of the Go extension is not influenced byt the length of the string, but it's just going as fast as it can, 
and that is the fastest speed possible.   
A loss in performance was to be expected, Go generate code that interacts with its memory manager and scheduler, it is somewhat in between Java and compiled languages.   
But honestly actually running slower than Ruby code was a real surprise.   
According to this [Russ Cox answer](https://groups.google.com/forum/#!msg/golang-nuts/RTtMsgZi88Q/61hgyGSkWiQJ), calling `C` from Go has an aoverhead 
similar to calling ten Go functions, looks like Go is one of those languages that can run faster ported code, than calling
the `C` implementation.   
If every function call counts for ten, it's no wonder that calling it thousands of time in a tight loop, would cause such
a tremendous loss in performances.    
To test this assumption, I moved the tight loop from Ruby to the Go extension: I ran the same comparison one thousand times
inside a Go loop and the same I did on the Ruby side.   
These are the new results:  

![Ruby VS Go updated](http://i.imgur.com/hCDdCfA.png "")

This time Go ran a bit faster, but with long strings the same slowness arises.   
I suspect the conversions routines from Ruby VALUE to Go strings are responsible for most of the overhead.   
Removing it from the equation gave me much better results (between 300 and 16 times faster than Ruby), but it's a low level optimization that makes sense only for three lines functions that are called over and over again, like this one.     
These numbers are not to be taken as a real benchmark, they are just the results of a micro benchmark and are not representetive of real performances in a real world application.
But they clearly show that running Go in a tight loop has a serious performance overhead, while if you delegate to
Go some heavy lifting work, it can give some performance boost.   


## Conclusions

Writing Ruby extensions in Go, especially in conjunction with the great `FFI` library, can be real fun.   
You got the feeling of *"scripting Ruby"* without any of the drawbacks of writing low level `C` code.   
Writing completely auto contained extensions, it's a lot more work, but it's more tedious than hard. 
The situation could improve vastly when someone will wrap the Ruby programming interface in a nice Go package to hide the `C` inheritance and maybe
write a [`go:generate`](https://blog.golang.org/generate) plugin to automate all the boilperplate code (for example
exporting the functions to `C`). But in the end it is still a lot easier than writing pure `C`.    
Perfomance wise though, I'm doubtful that you could have some gain just by rewriting parts of you app in Go.   
It is in fact quite possibly the opposite.   
Go has a performance problem when intercating with `C` and it's by design.   
   

However, there could be patterns where Go could be really helpful.  
I'm sure Go channles and concurrency are worth exploring.  
Maybe in a next episode.   

