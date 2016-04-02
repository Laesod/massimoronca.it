---
title: Building web apps with Traffic, the Go micro framework.  
date: "2013-11-29"
description: In times when software development is moving towards creating services for other applications, micro frameworks serve very well this paradigm by providing an easy way to create the building blocks on top of which larger applications can rely. Traffic is one of them.
source_name: MIKAMAYHEM
source_url: http://dev.mikamai.com/post/68453619468/building-web-apps-with-traffic-the-go-micro-framework
tags:
- traffic golang microframework
- golang
- frameworks inspired by sinatra
---

Written by the long time Ruby developer [Andrea Franz][1],  [**Traffic**][5] is a [*micro framework*][4] for Web development inspired by [Sinatra][3].

In times when software development is moving towards creating services for other applications, micro frameworks serve very well this paradigm by providing an easy way to create the building blocks, such as a REST API, on top of which larger applications can rely.  

Micro frameworks do not cover every single aspect of application building, most of them are left to the developer, for example you decide which database to use and how to access it or not to use a DB at all, but the codebase is so compact that a single person can master it in a matter of few days.

Like other popular frameworks, *Traffic* features include

- regexp routing
- chainable request filters
- middlewares
- templates
- serving of static files
- custom error handlers
- facilities to render in various format (HTML, JSON and XML)
- errors and stacktraces in the browser
- command line utility to generate new projects (although very basic)

Documentation is a little bit scarce at the moment, but there are many [examples][9] that make it easy to figure out how to use the various components.

We are going to analyze each one of them, by writing a small demo application, a simple clone of the placehold.it service powered by the Grumpy cat, instead of gray boxes.  
As usual, code is available on [github][7].

###Installing Traffic  
Assuming you already have a working installation of Go, download Traffic

``` bash
$ go get github.com/pilu/traffic
```

and the command line tool  

```bash
$ go get github.com/pilu/traffic/traffic
```

create a new project

```bash
$ traffic new demo_project
```

run it

```bash
$ cd demo_project  
$ go build && ./demo_project
```

and point your browser to [http://127.0.0.1:7000/](http://127.0.0.1:7000/)

### Routing

The first thing we need for our application is a router.  
Traffic routes are a pair of an HTTP method and a URL pattern that, if matched,
call a function that act as route-handler.  

Our first route

```go
package main

import (
  "github.com/pilu/traffic"
)

var router *traffic.Router

func main() {
  router = traffic.New()
  router.Get("/", RootHandler)
  router.Run()
}
```

Routes are matched in the same order they are declared, the first that match is executed.  

Routes can contain named parameters that are accesible using the `Param` function.  

```go
router.Get(`/:width/:height/?`, ImageHandler)

func ImageHandler(w traffic.ResponseWriter, r *traffic.Request) {
  width := r.Param("width")
  height := r.Param("height")
}
```

and parameters can be optional

```go
router.Get(`/:width)/(:height)?`, ImageHandler)
```

Route patterns can also include wildcards and regular expressions

```go
router.Get(`/:width/*`, ImageHandler)

// match (width)x(height) format
router.Get(`/(?P<width>\d+)(x(?P<height>\d+)?)?/?`, ImageHandler)

```

### Before filters

Traffic allows to prepend the request handler with filters, which are like regular request handlers that get executed before the real handler.  
Before fitlers can be chained and can be attached to all routes or just some of them.  
If a before filters write something in the Response Body, the request chain is interrupted.  

```go
// if route match, before executing ImageHandler, Traffic executes the two filters
// RequireValidImageParameters and GenerateImageCache in order  
// If one of them fails and write to the response body, the execution stops
// before reaching the actual handler
router.Get(`/:width/?(:height)?/?`, ImageHandler).
  AddBeforeFilter(RequireValidImageParameters).
  AddBeforeFilter(GenerateImageCache)

func RequireValidImageParameters(w traffic.ResponseWriter, r *traffic.Request) {
  width, err := strconv.Atoi(r.Param("width"))
  if err != nil { // conversion error, either var is empty or not a number
    // cannot continue
    w.WriteHeader(http.StatusNotFound)
    return
  }

  height, err := strconv.Atoi(r.Param("height"))
  if err != nil {
    // if height is omitted the image is gonna be a square
    height = width
  }

  if (width <= 2560 && width > 0) && (height <= 2560 && height > 0) {
    // set vars for the next filter
  } else {
    // bad request
    w.WriteHeader(http.StatusBadRequest)
    w.Render("400", nil)
  }
}

func GenerateImageCache(w traffic.ResponseWriter, r *traffic.Request) {
  // pseudo code
  if not cache_folder_exists and create_folder_fails
    throw error with panic

  write_image_file_according_to_parameters
}

func ImageHandler(w traffic.ResponseWriter, r *traffic.Request) {
  // output the image with the correct content-type
  w.Header().Set("Content-Type", "image/jpeg")

  // at this point we can safely assume that the image file already exists
}

// this filter is global to the router and is applied before each request
router.AddBeforeFilter(PoweredByHandler)

func PoweredByHandler(w traffic.ResponseWriter, r *traffic.Request) {
  w.Header().Set("X-Powered-By", "Grumpy cat")
}
```



### Templates and static files

Traffic supports templates in the standard Go format.  
Template library documentation can be found [here][8].  
Traffic Response Writer has a method to render templates called `Render`, that takes the template name (without the extension) and an optional param that contains the data to be rendered.
By default templates are placed in the `/view` folder.  
Templates can be nested one isnide the other like in our `404` example

```handlebars
{% raw %}
{{ template "includes/header" }}
  <div class="error error-404"></div>
{{ template "includes/footer" }}
{% endraw %}
```
If you are writing an API you might find the methods `WriteJSON` and `WriteXML` useful too.

Traffic also support serving static assets: every file placed in the `/public` folder is directly accessible.  
For example if we put a css inside `/public/css/app.css` it will be automatically accessible as
`http://address/css/app.css`.  

**Update**:  static files are served through `StaticMiddleware` that is added automatically only if environment is *‘development’*.
Environment is set using the `TRAFFIC_ENV` variable, so if you set it to`production`, you have to manually add the `StaticMiddleware`  

```go  
if traffic.Env() == "production" {
    router.Use(traffic.NewStaticMiddleware(traffic.PublicPath()))
}
```


### Custom error handlers

The Traffic router has builtin handlers for `404` and `500` erros that can be customized.

```go
// Custom not found handler
router.NotFoundHandler = NotFoundHandler

func NotFoundHandler(w traffic.ResponseWriter, r *traffic.Request) {
  w.Render("404")
}

// Custom error handler
router.ErrorHandler = ErrorHandler

func ErrorHandler(w traffic.ResponseWriter, r *traffic.Request) {
  w.Render("500")
}
```

### Conclusions

[Traffic][5] is a young framework specifically crafte for small to medium applications.  
I was able to create the demo app [Purrraceholder][7] (read it with a japanese accent) in a couple of hours, without previous knowledge of Traffic internals.  
I know there are people that can write a blog in 15 minutes, but I think hours is a more realistic time frame and, most of all, you are really in control of what's happening.  
If you wanna play with [Traffic][5], you can start by forking [Purraceholder][7] and adding some features.
These are the firsts that come to mind:  

- add more cats, there are never enough cats on the internet
- treat special cases with special cats: longcat for vertical images, monorail cat for horizontal ones
- add support for text overlay

happy coding with [Traffic][5].



[1]: http://gravityblast.com/
[2]: http://www.samanthajohn.com/post/17643298219/dont-learn-rails
[3]: http://www.sinatrarb.com
[4]: http://flask.pocoo.org/docs/foreword/#what-does-micro-mean
[5]: https://github.com/pilu/traffic/
[6]: http://placehold.it/
[7]: https://github.com/wstucco/purrraceholder
[8]: http://golang.org/pkg/html/template/
[9]: https://github.com/pilu/traffic/blob/master/examples/
