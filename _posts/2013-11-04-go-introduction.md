---
layout: post
title: I tried Go and I liked it
excerpt: Created inside Google by Ken Thompson and Rob Pike, fathers of Unix and UTF-8, to overcome the limitations of C++ \(compile times being the most annoying\), Go is a concurrent, garbage-collected language with fast compilation. This article will introduce you to the main language features and present you real life code snippets.
source-name: MIKAMAYHEM
source-url: http://dev.mikamai.com/post/65984358855/i-tried-go-and-i-liked-it
---

![Gopher in all of its glory][1]   

Say hello to [Gopher][2].   
Gopher is the mascotte of a new language from [Google][3], called [Go][4], with the capital **G**.  
Not a very clever name from a search engine company, if you ask me, but that's probably the only bad thing you will hear about it.

> **Hint:** use the word `golang` to search on Google

## A brief introduction

Created inside Google by [Ken Thompson][5] and [Rob Pike][6], fathers of Unix and UTF-8, to overcome the limitations of C++ (compile times being the most annoying), Go is a *concurrent*, *garbage-collected* language with *fast compilation*.  

Its real strength is just simplicity.  
As Rob Pike once said, [less is exponentially more](http://commandcenter.blogspot.it/2012/06/less-is-exponentially-more.html) and I strongly agree with him.   

### Features

 - blazing fast compilation speed
 - statically compiled binaries (the result is a single binary with no external dependencies)
 - type safe, statically typed with some type inference support. More errors get caught at compile time, less time is spent debugging
 - [garbage collected][7] with support for pointers, but no pointer arithmetics (for safety and good health of programmers minds).
 - strict compiler: [you can't declare a variable or import a package without using it][8]
 - concurrency and parallelism through [goroutines][9]. Goroutines are one of the peculiarities of Go, they are a cheap, lightweight construct built on top of threads, that run concurrently with other goroutines. If you have more than one core processor, they also run in parallel, in a completely transparent way for the programmer. Communication is managed sending messages through [channels][10] which are basically type safe queues. 
 - Object orientation but **no** classes. Any type can be an object.
 - **No type inheritance** in favour of *[composition][11]* and *[duck typing][12]*. IS-A relationships are banned!
 - [multiple return values][13]
 - [rich][14] standard library.
 - a powerful set of [command line tools][15] including one to [enforce coding conventions][16] and one for [automatic code documentation][17].  
Many IDE that support Go, launch `gofmt` just before save, to ensure that every Go file obey the rules.
 - last, but not least, cross compiling. Go compiler can create binaries for platforms/architectures different from the one it is running, provided [the platform is supported][18]. 

### Installing Go
You can download Go from the [golang.org website][19].  
There are packages for many different platforms (Linux, Windows, Mac OS, BSD) and architectures (x86, x64, ARM).  
Detailed instructions can be found [here][20].

Basically you need to create two environment variables:

 - `GOROOT` which is the system wide Go root folder (it should be
   configured automatically by the installer)  
 - `GOPATH` that will
   contain all your code and all the packages you are going to install
 - it is optional, but recommended, to add `$GOPATH/bin` to the `$PATH` variable 

### First steps and getting help

You should now have a working installation of the Go development environment, but if you come from Ruby or Python, you might get lost while reading someone else's code or trying to figure out which is the idiomatic way to solve a problem in Go.  
No worries, Go has an answer for that too.  
Included in the package there is [godoc][21].  
When launched with the `-http` param, godoc act as a web server that present the documentation in form of web pages.   
This is a typical godoc summoning ritual

``` bash
$ godoc -http=:60666
$ open http://localhost:60666
```

My advice is to read carefully [How to write Go code][22] and [Effective Go][23] before starting writing Go code.  
Go does not provide REPL, but you can try your snippets in the [Playground][24].

> **NOTE:** every package you install with the go tool or write by yourself, will update the documentation as well.  

## SHUT UP! SHOW US THE CODE!

Ok!

``` go
package main

import (
	"fmt"
	"net/http"
)

func main() {

	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello World!")
	})

	fmt.Println("Server running at http://localhost:8080/")
	fmt.Println("hit CTRL+C to shut it down")
	http.ListenAndServe(":8080", nil)
}
```
This is our first Go program.  
Every client that connects to localhost, port 8080, will receive the *"Hello World!"* message.  
The example looks a lot like the *Node.js* [hello world web serve example][25], but unlike *Node.js*, this code is already multithreaded. Every new client is served by a `goroutine`, that the `net/http` starts behind the scenes, taking advantage of concurrency and multiple cores, if present.   
Talking about simplicity, is there anything simpler than that?  

> **Note:** To run the code you have to save it in a file and then execute `go run <filename>` from the console  

Let's explore  the code above in more detail:

### Packages and imports
The first statement in a Go program must always be

``` go
package name
```  

Executable packages have to reside in a package called ***main***.  
The entry point function is called ***main*** as well.  

Next we find the import section, you can import packages by declaring your intentions with the ***import*** directive.  
This is the first Go idiom we encounter: *statement grouping*.  
You can import packages (or declare variables) one per line, like in

``` go
import "fmt"
import "net/http"
```
or you can group them together, surrounding the imports with parenthesis, like in our example.

### Variables and type inference
Variables are declared *name first, then type*.  
If you *declare and assign*, you can let the compiler infere the type.  

``` go
// when variables are declared without assignment, Go assign them a default value:
// zero for numbers, empty string for strings, nil for pointers and nullable types
var a string
var b int

// type inference
c := "Hello, Wolrd!"  // := operator is available only inside function body
var s = "a string"    // this pattern is available outside functions

``` 

Of course *statement grouping* is available too

``` go
var (
    s string
    i int
    f float64
)
```

Don't worry about lining up the elements between the brackets, `gofmt` will take care of it for you.  

### Anonymous functions and OO

Go support [anonymous functions][26] and high order functions.  
Functions are first class citizens in Go and can be assigned and carried around like regular variables.

``` go
log := func(s string) {
    fmt.Printf("[%s] %s", time.Now(), s)
}

log("Hello World!")

// sample output
// [2009-11-10 23:00:00 +0000 UTC] Hello World!

// a more complex example

type logLevel string 
type logger func(string) // create a new type for a function that takes a string as input 

getLogger := func(l logLevel) logger {
	return func(s string) {
		fmt.Printf("[%s] %s: %s\n", time.Now(), l, s)
	}
}

err := getLogger("Err")
warn := getLogger("Warn")
info := getLogger("Info")

err("File not found")
warn("Timezone is not set")
info("loggin' some stuff")

// sample output
// [2009-11-10 23:00:00 +0000 UTC] Err: File not found
// [2009-11-10 23:00:00 +0000 UTC] Warn: Timezone is not set
// [2009-11-10 23:00:00 +0000 UTC] Info: loggin' some stuff

```


Let's now improve our web server with new functionalities.  
We want to send out an `X-Powered-By` header with the name of our web server.  
What we need to do is define a new ***type*** that  act as handler for the requests and will append the new header to the default set of headers.  
In Go, an http handler is any object that implements the Handler's *interface*, and the Handler's *interface* contains only one method: `ServeHTTP`.  
So we are going to create a new object that implements the `ServeHTTP` method and pass it to `ListenAndServe`.  
Instead of classes, Go uses **structs**.  
In this case it is an empty struct, but it can contain any other type as member variables (more on this later).

``` go
// declare the new type Middleware.
// Note: the type can be literally **any** type
// type seq []int is a perfectly legal declaration
// it creates a new type (not an alias) seq that represents a slice of ints
type Middleware struct {
}

// implements the ServeHTTP method as requested by Handler's interface
// notice the syntax: 
// after the keyword func we declare the type that the method will be attached to 
// and then we pass a parameter m that represents our instance variable
// in a very similar way to Python's self
func (m *Middleware) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	fmt.Printf("Request to %s is being handled by our middleware\n", r.URL.Path)
	w.Header().Set("X-Powered-By", "mikamai-web-server")
}

// initialize and return a new object literal of type Middleware
// in Go we also declare the return type, after the parameters of the function
// in this case a pointer, denoted by the *, to Middleware type
func NewMiddleware() *Middleware {
	return &Middleware{}
}
```

We now pass the new handler to the listener

``` go
http.ListenAndServe(":8080", NewMiddleware())
```

Every time a client connects, it doesn't receive any message back, except for the new header.   
![X-Powered-By mikamai][27]

Nice, but not very interesting, plus we lost the ability to serve the content to the client, cause 
the handler function we declared before is not getting called.  
How can we fix that?  
Before explaining how to forward the call to the default handler, we're going to modify our middleware
to do something different.  
In addition to adding the header with the artist's signature, we want to limit the ability to visit 
our web site through one and only one specified host.  

``` go
// we expand our middleware to contain the definition of the single allowed host
type Middleware struct {
    allowedHost string
}

// we modify the initializer function (our constructor) as well
func NewMiddleware(host string) *Middleware {
	return &Middleware{
	    allowedHost: host, // trailing comma is required by Go compiler
	}
}
```

Rewrite `ServeHTTP` to check for the host validity

``` go
func (m *Middleware) ServeHTTP(w http.ResponseWriter, r *http.Request) {

	// strip the port from hostname
	host := strings.Split(r.Host, ":")[0] // import "strings" in order to use Split

    // the signature is sent anyway
	w.Header().Set("X-Powered-By", "mikamai-web-server")

	if host == m.allowedHost {
		fmt.Printf("Request for host %s is being handled by our middleware\n", host)

		// net/http has a default handler called DefaultServerMux
		// we feeded it with an handler for "/" in the first example
		// forward the call to the default handler and send "Hello World!" to the client
		http.DefaultServeMux.ServeHTTP(w, r)
	} else {
		// request is denied, wrong hostname
		fmt.Printf("Request for host %s is strictly forbidden by our middleware\n", host)

		// order is important.If we send data before the header, the server assumes the return code is 200 OK
		w.WriteHeader(403)
		fmt.Fprintf(w, "<h1>Forbidden</h1>you don't have permission to access host %s", host)
	}
}
```

update the handler's initialization

``` go
http.ListenAndServe(":8080", NewMiddleware("localhost"))
```

et voilà

![access denied][28]  
  
----------

![access granted][29]

----------
![console output][35]


All the code presented in this article can be downlaoded from [github][36]

  [1]: http://upload.wikimedia.org/wikipedia/commons/2/23/Golang.png "Gopher"
  [2]: http://golang.org/doc/gopher/
  [3]: http://google.com
  [4]: http://golang.org
  [5]: http://en.wikipedia.org/wiki/Ken_Thompson "Ken Thompson"
  [6]: http://en.wikipedia.org/wiki/Rob_Pike "Rob Pike"
  [7]: http://talks.golang.org/2012/splash.article#TOC_14.
  [8]: http://golang.org/doc/effective_go.html#blank_unused
  [9]: http://golang.org/doc/effective_go.html#goroutines
  [10]: http://golang.org/doc/effective_go.html#channels
  [11]: http://talks.golang.org/2012/splash.article#TOC_15.
  [12]: http://en.wikipedia.org/wiki/Duck_typing
  [13]: http://golang.org/doc/effective_go.html#multiple-returns
  [14]: http://golang.org/pkg/
  [15]: http://golang.org/doc/cmd
  [16]: http://golang.org/cmd/gofmt/
  [17]: http://blog.golang.org/godoc-documenting-go-code
  [18]: http://golang.org/doc/install#requirements
  [19]: https://golang.org
  [20]: http://golang.org/doc/install
  [21]: http://godoc.org/code.google.com/p/go.tools/cmd/godoc
  [22]: http://golang.org/doc/code.html
  [23]: http://golang.org/doc/effective_go.html
  [24]: http://play.golang.org/
  [25]: http://nodejs.org/about/
  [26]: http://golang.org/ref/spec#Function_literals
  [27]: http://31.media.tumblr.com/83ea05b44971684313f8d6d1c535b2d9/tumblr_mvhrpwaNvX1rhmakfo1_r1_500.png
  [28]: http://24.media.tumblr.com/aedc0f5dd849aae18ef022f7d10f3dad/tumblr_mvhrpwaNvX1rhmakfo2_r1_1280.png
  [29]: http://31.media.tumblr.com/d3cd5cabdc9bb2a83f12923093d6581f/tumblr_mvhrpwaNvX1rhmakfo3_r1_1280.png
  [30]: http://google.com
  [31]: http://golang.org
  [32]: http://en.wikipedia.org/wiki/Rob_Pike "Rob Pike"
  [33]: http://en.wikipedia.org/wiki/Ken_Thompson "Ken Thompson"
  [34]: http://plan9.bell-labs.com/plan9/glenda.html "Glenda the Plan9 Bunny"
  [35]: http://25.media.tumblr.com/a472850a96238d71194c7fbb23909ae7/tumblr_mvhrpwaNvX1rhmakfo4_r2_1280.png
  [36]: https://gist.github.com/wstucco/7248624
